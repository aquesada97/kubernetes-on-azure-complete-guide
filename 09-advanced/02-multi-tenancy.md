# Multi-Tenancy in AKS

Multi-tenancy allows multiple teams or applications to share an AKS cluster while maintaining isolation, resource fairness, and security boundaries.

---

## Multi-Tenancy Models

### Soft Multi-Tenancy (Namespace-Based)
- Teams share the same cluster
- Isolated by namespaces + RBAC + NetworkPolicies
- Resources are logically separated
- Lower cost, some blast radius risk

### Hard Multi-Tenancy (Cluster-per-Team)
- Each team gets their own cluster
- Complete isolation
- Higher cost, more operational overhead
- Suitable for strong compliance requirements

### Hybrid (Namespace + Dedicated Node Pools)
- Teams share the cluster
- Critical teams get dedicated node pools (via taints/tolerations)
- Balanced approach for most enterprises

---

## Namespace Setup for Teams

```bash
# Create team namespaces
kubectl create namespace team-alpha
kubectl create namespace team-beta
kubectl create namespace team-gamma

# Label namespaces (required for NetworkPolicies)
kubectl label namespace team-alpha team=alpha name=team-alpha
kubectl label namespace team-beta team=beta name=team-beta
kubectl label namespace team-gamma team=gamma name=team-gamma
```

---

## ResourceQuotas

Prevent any team from consuming all cluster resources:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-alpha-quota
  namespace: team-alpha
spec:
  hard:
    # Compute
    requests.cpu: "10"             # Total CPU requests for all pods
    requests.memory: "20Gi"        # Total memory requests
    limits.cpu: "20"               # Total CPU limits
    limits.memory: "40Gi"          # Total memory limits
    # Pods
    pods: "50"                     # Maximum number of pods
    # Services
    services: "20"
    services.loadbalancers: "2"    # Limit expensive LB services
    services.nodeports: "0"        # Disallow NodePort services
    # Storage
    persistentvolumeclaims: "20"
    requests.storage: "500Gi"
    # Objects
    configmaps: "50"
    secrets: "50"
    deployments.apps: "20"
```

```bash
# Apply quota
kubectl apply -f quota.yaml

# Check quota usage
kubectl describe resourcequota team-alpha-quota -n team-alpha
```

---

## LimitRange

Set default resource requests/limits to prevent unspecified pods from consuming unlimited resources:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-alpha-limits
  namespace: team-alpha
spec:
  limits:
  # Default container limits and requests
  - type: Container
    default:               # Applied if container has no limits
      cpu: "500m"
      memory: "512Mi"
    defaultRequest:        # Applied if container has no requests
      cpu: "100m"
      memory: "128Mi"
    max:                   # Containers cannot request more than this
      cpu: "4"
      memory: "8Gi"
    min:                   # Containers must request at least this
      cpu: "50m"
      memory: "64Mi"
  # Pod-level limits
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
  # PVC limits
  - type: PersistentVolumeClaim
    max:
      storage: "100Gi"
    min:
      storage: "1Gi"
```

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange team-alpha-limits -n team-alpha
```

---

## RBAC for Teams

Assign each team admin access to their namespace, with view access to shared namespaces:

```yaml
# Team Alpha admin binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-admin
  namespace: team-alpha
subjects:
- kind: Group
  name: "aad-group-team-alpha"        # Azure AD group object ID
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
---
# Team Alpha view access to shared namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-alpha-view-shared
  namespace: shared
subjects:
- kind: Group
  name: "aad-group-team-alpha"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Namespace Isolation with Network Policies

Deny cross-namespace traffic by default, allow only what's needed:

```yaml
# Default deny all ingress from other namespaces
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}               # Only from same namespace
---
# Allow from shared ingress controller
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: team-alpha
spec:
  podSelector: {}
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
---
# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: team-alpha
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

---

## Dedicated Node Pools per Team

Use taints and tolerations to dedicate nodes to specific teams:

```bash
# Create a dedicated node pool for Team Alpha
az aks nodepool add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name alphapool \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --node-taints team=alpha:NoSchedule    # Only team-alpha pods can run here
```

```yaml
# Team Alpha pods must tolerate the taint
spec:
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "alpha"
    effect: "NoSchedule"
  nodeSelector:
    agentpool: alphapool              # Target the specific pool
```

---

## Azure Policy for Governance

Enforce standards across all namespaces using Azure Policy:

```bash
# Apply built-in AKS policies
az policy assignment create \
  --name "aks-no-privileged-containers" \
  --policy "e345eecc-fa47-480f-9e88-67dcc122b164" \
  --scope $(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query id -o tsv) \
  --enforcement-mode Default

# Common AKS Policy Definitions
# - Do not allow privileged containers
# - Ensure resource limits are set
# - Require read-only root filesystem
# - No host network/port/process namespace
# - Disallow latest image tags
```

---

## Namespace Automation with Helm/Flux

For multiple teams, automate namespace configuration with a Helm chart or Flux Kustomization:

```yaml
# kustomization.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: team-alpha-namespace
  namespace: flux-system
spec:
  interval: 10m
  path: "./teams/team-alpha"
  prune: true
  sourceRef:
    kind: GitRepository
    name: cluster-config
```

---

## Summary

| Feature | Resource |
|---------|---------|
| Team isolation | Namespaces |
| Resource fairness | ResourceQuota |
| Default sizing | LimitRange |
| Access control | RoleBindings |
| Network isolation | NetworkPolicies |
| Node dedication | Taints + Tolerations |
| Policy enforcement | Azure Policy + Gatekeeper |

---

## Next Steps

Continue to [03-disaster-recovery.md](./03-disaster-recovery.md) to learn about backup and recovery strategies.
