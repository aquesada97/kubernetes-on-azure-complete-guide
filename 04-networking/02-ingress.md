# Ingress

An **Ingress** resource provides HTTP/HTTPS routing rules that expose multiple Services through a single load balancer. An **Ingress controller** implements these rules.

---

## Why Ingress?

Without Ingress, each Service needs its own Azure Load Balancer (costly and complex):
```
Internet → Azure LB 1 → Service A (api.example.com)
Internet → Azure LB 2 → Service B (web.example.com)
Internet → Azure LB 3 → Service C (api.example.com/v2)
```

With Ingress, one load balancer handles all traffic:
```
Internet → Azure LB → Ingress Controller → Service A (api.example.com)
                                         → Service B (web.example.com)
                                         → Service C (api.example.com/v2)
```

---

## Install NGINX Ingress Controller

The NGINX Ingress Controller is the most widely used Ingress controller for Kubernetes.

```bash
# Add Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install in a dedicated namespace
kubectl create namespace ingress-nginx

# Install
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
  --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get service ingress-nginx-controller -n ingress-nginx
```

Wait for the LoadBalancer to get an external IP:

```bash
kubectl get service ingress-nginx-controller -n ingress-nginx --watch
# NAME                       TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)
# ingress-nginx-controller   LoadBalancer   10.0.0.100   20.10.100.50    80:31080/TCP,443:31443/TCP
```

---

## Install Application Gateway Ingress Controller (AGIC)

For production AKS workloads, Azure Application Gateway Ingress Controller (AGIC) provides native Azure WAF integration.

```bash
# Enable AGIC addon on existing cluster
az aks enable-addons \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --addons ingress-appgw \
  --appgw-name myAppGateway \
  --appgw-subnet-cidr "10.225.0.0/16"
```

---

## Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx          # Match your ingress controller
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## Path-Based Routing

Route different paths to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-v1-service
            port:
              number: 8080
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 8080
      - path: /metrics
        pathType: Exact
        backend:
          service:
            name: metrics-service
            port:
              number: 9090
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
```

### PathType Options

| PathType | Description | Example path `/api` matches |
|----------|-------------|----------------------------|
| `Exact` | Exact path match | `/api` only |
| `Prefix` | Path prefix match | `/api`, `/api/v1`, `/api/users` |
| `ImplementationSpecific` | Ingress controller defines behavior | Controller-dependent |

---

## Host-Based Routing

Route different hostnames to different services:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing-ingress
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

---

## TLS/HTTPS with Ingress

### Manual TLS Secret

```bash
# Create TLS secret from certificate files
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace default
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-tls-secret    # TLS secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Automatic TLS with cert-manager

cert-manager automatically provisions and renews TLS certificates (e.g., from Let's Encrypt).

```bash
# Install cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

```yaml
# ClusterIssuer for Let's Encrypt
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
---
# Ingress with automatic TLS
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-auto-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls           # cert-manager creates this
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## Common NGINX Ingress Annotations

```yaml
metadata:
  annotations:
    # Rewrite URL before forwarding to backend
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/limit-connections: "5"

    # Client body size (file upload limit)
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://example.com"

    # Basic auth
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # Force SSL redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"

    # Whitelist specific IPs
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"

    # Custom backend config
    nginx.ingress.kubernetes.io/proxy-buffer-size: "8k"
```

---

## Ingress with URL Rewriting

```yaml
# /app/* → /* (strip /app prefix)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /app(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## Useful Ingress Commands

```bash
# List ingresses
kubectl get ingress -A

# Describe ingress (see events, backend status)
kubectl describe ingress my-ingress

# View NGINX controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=100

# Test ingress routing
curl -H "Host: myapp.example.com" http://20.10.100.50/
curl -H "Host: api.example.com" http://20.10.100.50/api/v1/users

# Check cert-manager certificates
kubectl get certificates -A
kubectl describe certificate myapp-tls
```

---

## Summary

| Feature | Description |
|---------|-------------|
| Ingress Controller | Implements routing rules (NGINX, AGIC) |
| Host-based routing | Different hostnames → different services |
| Path-based routing | Different paths → different services |
| TLS termination | HTTPS → HTTP at controller, or passthrough |
| cert-manager | Automatic Let's Encrypt TLS certificates |

---

## Next Steps

Continue to [03-network-policies.md](./03-network-policies.md) to secure traffic between pods.
