# Services

A **Service** provides a stable network endpoint to access a dynamic set of Pods. Since Pods are ephemeral and their IPs change, Services provide a consistent DNS name and IP (ClusterIP) that doesn't change.

---

## How Services Work

Services use **label selectors** to find their backing Pods. When you create a Service, Kubernetes creates an **Endpoints** (or **EndpointSlice**) object that tracks the IPs of all matching Pods.

```
Client → Service (ClusterIP: 10.0.0.50:80)
             ↓ (iptables/IPVS rules)
         ┌───┴────────────────┐
         ↓           ↓        ↓
      Pod 1       Pod 2    Pod 3
   10.244.1.5  10.244.2.3  10.244.3.7
```

```bash
# See the Endpoints backing a Service
kubectl get endpoints my-service
kubectl get endpointslices -l kubernetes.io/service-name=my-service
```

---

## Service Types

### 1. ClusterIP (Default)

Exposes the Service on an internal IP in the cluster. Only accessible from within the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-backend
  namespace: default
spec:
  type: ClusterIP          # Default — can be omitted
  selector:
    app: my-backend
  ports:
  - name: http
    protocol: TCP
    port: 80               # Port the Service listens on
    targetPort: 8080       # Port on the Pod to forward to
```

**Use cases:** Internal microservice communication, database access within the cluster.

```bash
# DNS name format: <service-name>.<namespace>.svc.cluster.local
# From within any pod: curl http://my-backend.default.svc.cluster.local:80
# Or short form within same namespace: curl http://my-backend:80
```

### 2. NodePort

Exposes the Service on each Node's IP at a static port (30000–32767). Accessible from outside the cluster using `NodeIP:NodePort`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80             # ClusterIP port (internal)
    targetPort: 8080     # Pod port
    nodePort: 30080      # Node port (optional — auto-assigned if omitted)
```

```bash
# Access via any node's IP
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
curl http://$NODE_IP:30080
```

**Use cases:** Development, testing. Not recommended for production (limited security, manual port management).

### 3. LoadBalancer

Exposes the Service externally using a cloud provider load balancer. In AKS, this creates an **Azure Load Balancer** with a public IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-public-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-sku: standard        # Standard SKU
    service.beta.kubernetes.io/azure-dns-label-name: my-app-training    # DNS label
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
```

```bash
# Wait for external IP to be assigned
kubectl get service my-public-service --watch

# Get the external IP
kubectl get service my-public-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

**Use cases:** Exposing a single service to the internet when you don't need path-based routing.

### 4. Internal Load Balancer (AKS-specific)

Creates an Azure Internal Load Balancer — accessible only within the VNet.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "my-private-subnet"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

**Use cases:** Backend services that should only be accessible from within your VNet or peered networks (not the internet).

### 5. ExternalName

Maps a Service to an external DNS name. No proxying — just DNS CNAME.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: mydb.database.windows.net
```

Pods can now connect to `external-db:1433` which resolves to `mydb.database.windows.net`.

**Use cases:** Abstracting external services so they can be replaced without changing application code.

---

## Service Type Comparison

| Type | Accessibility | Azure Integration | Use Case |
|------|--------------|-------------------|----------|
| ClusterIP | Cluster only | None | Internal microservices |
| NodePort | Node IP + port | None | Dev/test |
| LoadBalancer | Public internet | Azure Public LB | Production single service |
| Internal LB | VNet only | Azure Internal LB | Private backend services |
| ExternalName | External DNS | None | External service abstraction |

---

## Headless Services

A **headless Service** has `clusterIP: None`. Instead of a ClusterIP, DNS returns the individual Pod IPs directly. Used with StatefulSets.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None        # Headless — no VIP
  selector:
    app: my-stateful-app
  ports:
  - port: 5432
    targetPort: 5432
```

```bash
# DNS returns all pod IPs
nslookup my-headless-service.default.svc.cluster.local
# → 10.244.1.5
# → 10.244.2.3

# Individual pods addressable as:
# pod-0.my-headless-service.default.svc.cluster.local
# pod-1.my-headless-service.default.svc.cluster.local
```

---

## Multi-Port Services

Services can expose multiple ports. Each port must have a `name`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-multi-port-service
spec:
  selector:
    app: my-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8080
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  - name: metrics
    protocol: TCP
    port: 9090
    targetPort: 9090
```

---

## Session Affinity

Route all requests from the same client to the same Pod:

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600    # 1 hour session stickiness
```

---

## Service Discovery

Kubernetes provides two mechanisms for service discovery:

### DNS-Based (Recommended)

```bash
# Same namespace
curl http://my-service:80

# Cross-namespace
curl http://my-service.other-namespace.svc.cluster.local:80

# Full FQDN
curl http://my-service.default.svc.cluster.local:80
```

### Environment Variables (Legacy)

When a Pod starts, Kubernetes injects environment variables for each Service in the same namespace:

```bash
# Format: <SERVICE_NAME>_SERVICE_HOST and <SERVICE_NAME>_SERVICE_PORT
echo $MY_SERVICE_SERVICE_HOST
echo $MY_SERVICE_SERVICE_PORT
```

> ⚠️ **Note:** Environment variable discovery only works for Services created *before* the Pod starts. DNS-based discovery is recommended.

---

## Service without Selector (Manual Endpoints)

Create a Service without a selector to manually define endpoints. Useful for:
- External databases
- Services in another namespace
- Legacy systems

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-service
spec:
  ports:
  - port: 1433
    targetPort: 1433
---
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service   # Must match Service name
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 1433
```

---

## Useful Service Commands

```bash
# List all services
kubectl get services -A

# Get service details
kubectl describe service my-service

# Get service endpoints
kubectl get endpoints my-service

# Test service connectivity from inside cluster
kubectl run -it --rm test --image=busybox --restart=Never -- \
  wget -O- http://my-service:80 --timeout=5

# Watch service for external IP assignment
kubectl get service my-service --watch

# Delete a service
kubectl delete service my-service
```

---

## Summary

| Service Type | ClusterIP | External Access | Azure Resource |
|-------------|-----------|-----------------|----------------|
| ClusterIP | Yes | No | None |
| NodePort | Yes | Node IP + port | None |
| LoadBalancer (public) | Yes | Public Internet | Azure Public LB |
| LoadBalancer (internal) | Yes | VNet only | Azure Internal LB |
| ExternalName | No | DNS CNAME | None |
| Headless | None | Pod IPs via DNS | None |

---

## Next Steps

Continue to [03-configmaps-secrets.md](./03-configmaps-secrets.md) to learn how to manage application configuration.
