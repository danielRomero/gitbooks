# b. Con LetsEncrypt \(cert-manager\)

Esta manera de conseguir el certificado es un poco más compleja.

{% hint style="info" %}
Necesitamos tener [Helm](https://github.com/kubernetes/helm/blob/master/docs/install.md) instalado.
{% endhint %}

En primer lugar necesitamos instalar [cert-manager](https://github.com/jetstack/cert-manager/), un add-on para kubernetes que nos ayuda a obtener certificados de _lets-encrypt_ para nuestra aplicación. Para ello, lo instalamos vía [`Helm`](https://helm.sh/).

```bash
helm install \
  --name cert-manager \
  --namespace kube-system \
  --set rbac.create=false \
  stable/cert-manager
```

con esto, instalamos el add-on bajo el namespace kube-system, disponible  desde el resto de namespaces.

El tipo de validación que vamos a seguir es [validación por DNS](https://cert-manager.readthedocs.io/en/latest/tutorials/acme/dns-validation.html)

#### Issuer

Issuer es un componente nuevo que añade el add-on. Este componente se encargará de gestionar la obtención del certificado. Solo sabe de la configuración, es decir, no sabe que dominio tiene que aplicar, solo sabe el como. Del dominio se encargará el `certificate` como veremos más adelante.

{% code-tabs %}
{% code-tabs-item title="infrastructure/kubernetes/staging/ssl/issuer.yaml" %}
```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Issuer
metadata:
  name: letsencrypt-issuer-pre
spec:
  acme:
    # production server validation => https://acme-v01.api.letsencrypt.org/directory
    server: https://acme-staging.api.letsencrypt.org/directory
    email: my_email@provider.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-issuer-account-key
    # ACME DNS-01 provider configurations
    dns01:
      # Here we define a list of DNS-01 providers that can solve DNS challenges
      providers:
        - name: route53
          route53:
            region: eu-west-1
            # optional if ambient credentials are available; see ambient credentials documentation
            accessKeyID: AKIAXXXXXXXXXXXXX
            secretAccessKeySecretRef:
              name: route53-credentials-secret
              key: secret-access-key
# RATE LIMITS:
# If you’ve hit a rate limit, we don’t have a way to temporarily reset it.
# You’ll need to wait until the rate limit expires after a week.
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl create -f infrastructure/kubernetes/staging/ssl/issuer.yaml --namespace=staging
```

Como se puede ver, se especifica el servidor de la API de LetsEncrypt que tiene que utilizar, el nombre del secret donde crear la clave privada de la validación y el secret con los credenciales de route53.

{% hint style="warning" %}
Es importante utilizar el _staging_ de la api de _letsencrypt_ durante las pruebas y cuando veamos que ha tenido éxito, utilizar la URL de producción \(comentada en la línea 7\) debido a los límites de uso de la API de _letsencrypt_, que puede llegar a hacerte esperar una semana.
{% endhint %}

Como es obvio necesitamos el secret con la clave de AWS 

{% code-tabs %}
{% code-tabs-item title="infrastructure/kubernetes/staging/secrets/route53.yaml" %}
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: route53-credentials-secret
type: Opaque
data:
  # echo -n "XXXXXXXXXXXXXXXXXXXXXX" | base64
  secret-access-key: WFhYWFhYWFhYWFhYWFhYWFhYWFhYWA==
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl create -f infrastructure/kubernetes/staging/secrets/route53.yaml --namespace=staging
```

#### Certificate

Una vez tenemos creado el Issuer, ahora toca el certificate donde indicaremos que Issuer vamos a utilizar, para qué dominios necesitamos generar el certificado y el nombre del secret donde se guardará el mismo.

{% code-tabs %}
{% code-tabs-item title="infrastructure/kubernetes/staging/ssl/certificate.yaml" %}
```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: Certificate
metadata:
  name: letsencrypt-cert-pre
spec:
  # Name of the secret where will be stored the cert on success domain validation
  secretName: ssl-cert-secret
  issuerRef:
    # Issuer name to use
    name: letsencrypt-issuer-pre
  commonName: my_domain.com
  dnsNames:
  - my_domain.com
  acme:
    # ACME validation settings
    config:
    - dns01:
        provider: route53
      domains:
      - my_domain.com
```
{% endcode-tabs-item %}
{% endcode-tabs %}

```bash
kubectl create -f infrastructure/kubernetes/staging/ssl/certificate.yaml --namespace=staging
```

Como se puede ver, en la línea 7 se indica el nombre del secret que se generará tras una validación exitosa. Ese mismo nombre es el que debemos poner en nuestro Ingress, definido previamente [aquí](../7.-nginx-ingress.md#1-nginx-ingress).

