# Network Policies

**Network Policies** are Kubernetes resources that control traffic flow between Pods, namespaces, and external endpoints. By default, all pod-to-pod traffic is allowed — Network Policies let you enforce a "deny by default, allow by exception" model.

---

## Prerequisites

Network Policies require a CNI plugin that supports them. In AKS:

- **Azure CNI** — supports Kubernetes Network Policy and Azure Network Policy Manager (NPM)
- **Kubenet** — limited support; Azure NPM not available
- **Calico** — available as an overlay policy engine on AKS

```bash
# Enable Calico network policy when creating cluster
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --network-plugin azure \
  --network-policy calico \
  ...

# Or with Azure Network Policy Manager
az aks create \
  --network-plugin azure \
  --network-policy azure \
  ...
```

---

## How Network Policies Work

A Network Policy selects pods via `podSelector` and defines:
- `ingress` rules — inbound traffic to selected pods
- `egress` rules — outbound traffic from selected pods

**Key behavior:**
- Without any NetworkPolicy, all traffic is allowed
- Once a NetworkPolicy selects a pod, only traffic matching a rule is allowed
- Multiple NetworkPolicies selecting the same pod are additive (OR logic)

---

## Default Deny All

Start with deny-all, then explicitly allow required traffic:

```yaml
# Deny all ingress traffic in the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # Matches ALL pods in namespace
  policyTypes:
  - Ingress                # Apply to ingress traffic only
```

```yaml
# Deny all egress traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

```yaml
# Deny all ingress AND egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## Allow Specific Ingress

### Allow from Specific Pods

Only allow traffic from the frontend to the backend:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend           # This policy applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend      # Only allow from frontend pods
    ports:
    - protocol: TCP
      port: 8080
```

### Allow from Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-monitoring
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring   # Allow from all pods in 'monitoring' namespace
    ports:
    - protocol: TCP
      port: 9090
```

### Allow from Specific Pods in Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: monitoring
      podSelector:           # AND: must match both namespace AND pod labels
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090
```

> 💡 **Important:** `namespaceSelector` + `podSelector` in the same `-from` item = AND logic (both must match). Separate `-from` items = OR logic.

### Allow from Anywhere (all ingress on a port)

```yaml
ingress:
- ports:
  - protocol: TCP
    port: 80
  # No 'from' section = allow from anywhere
```

---

## Allow Specific Egress

### Allow DNS and External HTTPS

Every pod needs DNS resolution and often needs to call external APIs:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
  # Allow HTTPS to internet
  - ports:
    - protocol: TCP
      port: 443
  # Allow communication to another service
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### Allow Egress to Specific Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-to-infra
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          purpose: infrastructure
```

### Allow Egress to External IP Block

```yaml
egress:
- to:
  - ipBlock:
      cidr: 10.0.0.0/8        # Allow to entire 10.x network
      except:
      - 10.5.0.0/16           # Except this subnet
  ports:
  - protocol: TCP
    port: 443
```

---

## Real-World Example: 3-Tier Application

```yaml
# Tier 1: Frontend — allow HTTP from internet (via ingress controller)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx    # From ingress controller namespace
    ports:
    - port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - port: 8080
  - ports:           # DNS
    - port: 53
      protocol: UDP
---
# Tier 2: API — allow only from frontend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - ports:           # DNS
    - port: 53
      protocol: UDP
---
# Tier 3: Database — allow only from API
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: api
    ports:
    - port: 5432
  egress:
  - ports:           # DNS only
    - port: 53
      protocol: UDP
```

---

## Namespace Isolation

Label namespaces to enable namespace-selector policies:

```bash
# Label namespace
kubectl label namespace production name=production
kubectl label namespace monitoring name=monitoring
kubectl label namespace ingress-nginx name=ingress-nginx
```

```yaml
# Isolate a namespace — only allow traffic within the namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}         # Same namespace (empty selector = all pods)
```

---

## Testing Network Policies

```bash
# Test connectivity before and after applying policies
# Run a test pod
kubectl run -it --rm nettest \
  --image=busybox \
  --namespace=production \
  --restart=Never \
  -- wget -O- --timeout=5 http://my-service:8080

# If blocked: wget: can't connect to remote host (10.0.0.50): Connection refused
# If allowed: <!DOCTYPE html>...

# Check which policies apply to a pod
kubectl get networkpolicies -n production
kubectl describe networkpolicy allow-frontend-to-backend -n production
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| Default behavior | All traffic allowed (no NetworkPolicy) |
| Policy scope | Pod selector + ingress/egress rules |
| Deny all | Empty podSelector with no rules = block all |
| AND vs OR | Same `-from` item = AND; separate items = OR |
| DNS | Always allow UDP/TCP port 53 when restricting egress |
| Namespace labels | Required for namespaceSelector to work |

---

## Next Steps

Proceed to [05 - Storage](../05-storage/README.md) to learn about persistent storage in AKS.
