## Cert-manager with cloudflare for automatic TLS certificates

Configuration files for configuration of cert-manager to fully automatic get certificates for application in Kubernetes.

Installation of cert-manager:

**Static Install**
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

More information: [https://cert-manager.io/docs/installation/]

Installation of ClusterIssuer with secret for api keys:

**issuer.yaml**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cloudflare-issuer
spec:
  acme:
    email: <email>
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-key
    solvers:
    - dns01:
        cloudflare:
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token           
```

**issuer-secret.yaml**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-token-secret
  namespace: cert-manager 
type: Opaque
stringData:
  api-token: <token>
```

Get the certificate

**certificate.yaml**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: tls-ingress-certificate
  namespace: <namespace>
spec:
  dnsNames:
    - "host.domainname.tld"
  secretName: tls-ingress-certificate
  issuerRef:
    name: letsencrypt-cloudflare-issuer
    kind: ClusterIssuer
```

Get more information

```shell
kubectl -n cert-manager describe clusterissuers.cert-manager.io

kubectl -n <namespace> get certificaterequests.cert-manager.io

kubectl -n <namespace> get orders.acme.cert-manager.io

kubectl -n <namespace> describe orders.acme.cert-manager.io <order>

kubectl -n <namespace> get events
```