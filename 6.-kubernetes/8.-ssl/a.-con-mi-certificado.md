# a. Con mi certificado

Esta forma es la más sencilla. Una vez tenemos nuestro certificado \(el bundle y la key\) solo tenemos que generar el secret y subirlo al namespace correcto con el nombre adecuado.

[Como vimos,](../7.-nginx-ingress.md#1-nginx-ingress) el Ingress espera un secret con el nombre `ssl-cert-secret` así que lo vamos a crear desde el CLI de kubectl, que es más sencillo.

```bash
kubectl create secret tls ssl-cert-secret --key my_cert.key --cert my_cert-bundle.crt --namespace=staging
```

Ahora podemos ver que se ha creado el secret y si visitamos nuestro dominio, ya debe servirse con el certificado.

```text
kubectl get secrets --namespace=staging
```

