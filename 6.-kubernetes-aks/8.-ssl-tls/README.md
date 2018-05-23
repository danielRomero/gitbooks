# 8. SSL/TLS

Como es normal queremos que nuestra aplicación se sirva con un certificado SSL válido de forma que nuestra web se sirva de forma segura.

[Previamente](../7.-nginx-ingress.md#1-nginx-ingress) hemos visto que el Ingress es el recurso que gestiona este apartado, si nos fijamos en las siguientes líneas vemos que se especifica un secret donde se almacena el certificado para el SSL.

```yaml
spec:
  tls:
  - hosts:
    - my-domain.com
    secretName: ssl-cert-secret
```

Hay distintas maneras de gestionar ese secret.

1. [Con nuestro propio certificado](a.-con-mi-certificado.md)
2. [Generando uno automáticamente con LetsEncrypt \(validación vía Route53 API\)](b.-con-letsencrypt-cert-manager.md)

