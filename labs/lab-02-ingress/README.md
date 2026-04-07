# Lab 02: Configure Ingress

In this lab, you will install the NGINX Ingress Controller and configure host-based and path-based routing rules to expose multiple services through a single load balancer.

**Estimated Time:** 45 minutes  
**Difficulty:** Intermediate

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

- Install NGINX Ingress Controller using Helm
- Configure path-based routing to multiple backend services
- Configure host-based routing
- Enable TLS with a self-signed certificate
- Use Ingress annotations to customize behavior

---

## Prerequisites

- Completed [Lab 01](../lab-01-deploy-first-app/README.md)
- Helm installed
- `kubectl` configured

---

## Lab Setup

```bash
kubectl create namespace lab02
kubectl config set-context --current --namespace=lab02
```

---

## Part 1: Install NGINX Ingress Controller

### 1.1 Install via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create namespace ingress-nginx

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.replicaCount=2 \
  --set controller.nodeSelector."kubernetes\.io/os"=linux \
  --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux

# Wait for LoadBalancer IP
kubectl get service ingress-nginx-controller -n ingress-nginx --watch
```

### 1.2 Save the Ingress Controller IP

```bash
INGRESS_IP=$(kubectl get service ingress-nginx-controller \
  -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Ingress IP: $INGRESS_IP"
```

---

## Part 2: Deploy Sample Applications

### 2.1 Apply Manifests

```bash
cd labs/lab-02-ingress
kubectl apply -f manifests/deployment.yaml
kubectl apply -f manifests/service.yaml

kubectl get pods -n lab02
kubectl get services -n lab02
```

---

## Part 3: Configure Path-Based Ingress

### 3.1 Apply the Ingress

```bash
kubectl apply -f manifests/ingress.yaml

# Verify
kubectl get ingress -n lab02
kubectl describe ingress lab02-ingress -n lab02
```

### 3.2 Test Path Routing

```bash
# Test /app1 path (routes to app1)
curl -H "Host: lab02.example.com" http://$INGRESS_IP/app1

# Test /app2 path (routes to app2)
curl -H "Host: lab02.example.com" http://$INGRESS_IP/app2

# Test root path (routes to main app)
curl -H "Host: lab02.example.com" http://$INGRESS_IP/
```

---

## Part 4: TLS Configuration

### 4.1 Create a Self-Signed Certificate

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=lab02.example.com/O=lab02"

# Create TLS secret
kubectl create secret tls lab02-tls-secret \
  --cert=tls.crt \
  --key=tls.key \
  --namespace=lab02
```

### 4.2 Update Ingress with TLS

```bash
# Edit ingress.yaml to add TLS section, then apply
kubectl apply -f manifests/ingress.yaml

# Test HTTPS (self-signed, so use -k for insecure)
curl -k -H "Host: lab02.example.com" https://$INGRESS_IP/
```

---

## Part 5: Explore Ingress Annotations

### 5.1 Add Rate Limiting

```bash
kubectl annotate ingress lab02-ingress \
  nginx.ingress.kubernetes.io/limit-rps="5" \
  --overwrite

# Test rate limiting
for i in {1..10}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -H "Host: lab02.example.com" http://$INGRESS_IP/
done
# Should see 503 after 5 requests
```

### 5.2 Add Custom Response Headers

```bash
kubectl annotate ingress lab02-ingress \
  nginx.ingress.kubernetes.io/configuration-snippet='
    add_header X-Custom-Header "AKS-Training-Lab02" always;' \
  --overwrite

curl -I -H "Host: lab02.example.com" http://$INGRESS_IP/
# Should see X-Custom-Header in response
```

---

## Part 6: Inspect NGINX Configuration

```bash
# View NGINX controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller --tail=50

# Get the generated NGINX config
NGINX_POD=$(kubectl get pods -n ingress-nginx -l app.kubernetes.io/component=controller -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $NGINX_POD -n ingress-nginx -- cat /etc/nginx/nginx.conf | grep -A 20 "server_name lab02"
```

---

## Cleanup

```bash
kubectl delete namespace lab02
# Keep ingress-nginx for future labs, or:
# helm uninstall ingress-nginx -n ingress-nginx
# kubectl delete namespace ingress-nginx
```

---

## ✅ Lab Completion Checklist

- [ ] Installed NGINX Ingress Controller
- [ ] Deployed two backend applications
- [ ] Configured path-based routing
- [ ] Tested routing with curl using Host header
- [ ] Created TLS secret and enabled HTTPS
- [ ] Applied rate limiting annotation

---

## 🤔 Discussion Questions

1. Why does the Ingress need a LoadBalancer IP at the controller level but not individual services?
2. What is the difference between path-based and host-based routing?
3. When would you use AGIC (Application Gateway Ingress Controller) over NGINX?

---

## Next Lab

Proceed to [Lab 03: Persistent Storage](../lab-03-storage/README.md).
