# 4. Imágenes de Docker

Para desplegar este proyecto vamos a hacer 4 imágenes sencillas de docker:

* Builder
* Web
* App
* Sidekiq

La idea es que el builder sea la base de nuestro web y sidekiq pero además nos sirva para la app como veremos a continuación.

La infraestructura se compondrá de:

*  Un Nginx \(web\) que además de servir como reverse proxy servirá los ficheros estáticos de nuestra aplicación desde la carpeta public
* Un App \(puma\) que se encargará de servir la aplicación.
* Sidekiq que hará los trabajos de background.

#### 1. Builder

Se trata de una imagen de base para Ruby que se encargará de:

* Instalar las dependencias necesarias para que nuestro Rails arranque
* Copiar el proyecto dentro de la imagen
* Instalar las gemas
* precompilar assets

{% code-tabs %}
{% code-tabs-item title="infrastructure/dockerfiles/builder.Dockerfile" %}
```bash
# The base image, with ruby pre-installed
# It runs the bundle install and assets precompilation
#see: https://hub.docker.com/_/ruby/
FROM ruby:2.3

# Install dependencies:
# - build-essential: To ensure certain gems can be compiled
# - imagemagick: To process images and resize
RUN apt-get update && apt-get install -qq -y \
  build-essential \
  imagemagick \
  --fix-missing \
  --no-install-recommends

# Set an environment variable to store where the app is installed to inside
# of the Docker image.
ENV INSTALL_PATH /app
RUN mkdir -p $INSTALL_PATH

# This sets the context of where commands will be ran in and is documented
# on Docker's website extensively.
WORKDIR $INSTALL_PATH

# (optional/recommended) Environment variables for Dockerized production Rails apps
ENV RAILS_LOG_TO_STDOUT true
ENV BUNDLE_PATH /bundle
ENV GEM_HOME /bundle

# Ensure gems are cached and only get updated when they change. This will
# drastically increase build times when your gems do not change.
COPY Gemfile Gemfile
COPY Gemfile.lock Gemfile.lock
RUN bundle install --deployment -j $(nproc)

# Copy code from working directory outside Docker to working directory inside Docker
COPY . .

#Sometime an extra bundle call is needed to install binaries / native extensions
RUN bundle install --deployment -j $(nproc)

# Copy database configuration and rename for kubernetes
COPY config/database.yml.kubernetes config/database.yml

# Precompile assets
RUN bundle exec rake DATABASE_URL=mysql2:does_not_exist assets:precompile
# assets are stored in /app/public/assets/
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### 2. Web

Esta imagen tendrá nuestro nginx sirviendo los assets previamente compilados en el builder.

También tendrá la configuración del nginx que necesitamos.

{% code-tabs %}
{% code-tabs-item title="infrastructure/dockerfiles/web.Dockerfile" %}
```bash
FROM myacrname.azurecr.io/myrailsproyect/builder:latest as builder

###

FROM nginx:alpine

ENV APP_DIR /var/www/myrailsproyect
RUN mkdir -p $APP_DIR/log $APP_DIR/public

WORKDIR $APP_DIR

# Copy assets from public folder to inside the image
COPY --from=builder /app/public ./public

# replace the default nginx configuration with our custom
COPY ./infrastructure/dockerfiles/nginx.conf /etc/nginx/conf.d/default.conf

# start the nginx service
CMD ["nginx", "-g", "daemon off;"]
```
{% endcode-tabs-item %}
{% endcode-tabs %}

y nuestra configuración de nginx simplificada:

{% code-tabs %}
{% code-tabs-item title="infrastructure/dockerfiles/nginx.conf" %}
```bash
upstream myrailsproyect-backend {
  server myrailsproyect-rails-svc:3000; # Kubernetes reacheable service
}

server {
  listen 80;

  root /var/www/myrailsproyect/public;

   # GZIP configuration
  gzip on;
  gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;

  # Properly server assets
  location ~ ^/(assets)/ {
    root /var/www/myrailsproyect/public;
    gzip_static on;
    gzip_proxied expired no-cache no-store private auth;
    expires max;
    add_header Cache-Control public;
    add_header ETag "";
    break;
  }

  # Proxy request to rails app
  location / {
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_redirect off;

    proxy_pass_header Set-Cookie;
    proxy_pass http://myrailsproyect-backend;
    break;
  }
}
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Como podemos ver aquí no hay ninguna configuración sobre SSL, de eso nos encargaremos más adelante con kubernetes.

#### 3. App

Esta imágen se encarga de servir la aplicación rails. Parte de la imagen generada con el builder y la extiende.

{% code-tabs %}
{% code-tabs-item title="infrastructure/dockerfiles/app.Dockerfile" %}
```bash
FROM myacrname.azurecr.io/myrailsproyect/builder:latest

# This sets the context of where commands will be ran in and is documented
# on Docker's website extensively.
WORKDIR $INSTALL_PATH

# Copy database configuration and rename for kubernetes
COPY config/database.yml.kubernetes config/database.yml

# (optional/recommended) Environment variables for Dockerized production Rails apps
ENV RAILS_SERVE_STATIC_FILES true

# The default command to start the rails web server.
CMD bundle exec rails server -b 0.0.0.0
```
{% endcode-tabs-item %}
{% endcode-tabs %}

#### 4. Sidekiq

Exactamente igual que la anterior solo que no levanta un servidor web sino el sidekiq

{% code-tabs %}
{% code-tabs-item title="infrastructure/dockerfiles/sidekiq.Dockerfile" %}
```bash
FROM myacrname.azurecr.io/myrailsproyect/builder:latest

# This sets the context of where commands will be ran in and is documented
# on Docker's website extensively.
WORKDIR $INSTALL_PATH

# Copy database configuration and rename for kubernetes
COPY config/database.yml.kubernetes config/database.yml

# The default command to start the sidekiq service.
CMD bundle exec sidekiq -C config/sidekiq.yml
```
{% endcode-tabs-item %}
{% endcode-tabs %}

**Nota:** quizá esta no es la mejor manera de hacerlo, no soy ningún experto en docker o kubernetes.

Ok, con esto ya tenemos los dockerfiles listos, ahora veamos como funciona esto en el stage`build`de nuestro CI-CD

