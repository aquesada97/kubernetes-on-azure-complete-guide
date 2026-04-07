# RBAC (Role-Based Access Control)

Kubernetes RBAC controls who can do what in the cluster. It is the primary authorization mechanism and works by granting permissions through Roles and Bindings.

---

## RBAC Components

| Resource | Scope | Purpose |
|----------|-------|---------|
| `Role` | Namespace | Permissions within a single namespace |
| `ClusterRole` | Cluster-wide | Permissions across all namespaces or non-namespaced resources |
| `RoleBinding` | Namespace | Binds a Role or ClusterRole to subjects within a namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to subjects across all namespaces |

**Subjects** (who gets permissions):
- `User` — Human user (from Azure AD or certificate)
- `Group` — Group of users (from Azure AD)
- `ServiceAccount` — Identity for a Pod (non-human)

---

## Roles

### Creating a Role (Namespace-Scoped)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]                # "" = core API group
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

### Common Verbs

| Verb | HTTP Method | Description |
|------|-------------|-------------|
| `get` | GET | Read a single resource |
| `list` | GET | List resources |
| `watch` | GET+watch | Watch for changes |
| `create` | POST | Create a resource |
| `update` | PUT | Replace a resource |
| `patch` | PATCH | Partial update |
| `delete` | DELETE | Delete a resource |
| `deletecollection` | DELETE | Delete multiple resources |
| `exec` | POST | Execute in a pod |

### Common API Groups

| API Group | Resources |
|-----------|-----------|
| `""` (core) | pods, services, configmaps, secrets, namespaces, persistentvolumeclaims |
| `apps` | deployments, replicasets, statefulsets, daemonsets |
| `batch` | jobs, cronjobs |
| `networking.k8s.io` | ingresses, networkpolicies |
| `rbac.authorization.k8s.io` | roles, rolebindings, clusterroles, clusterrolebindings |
| `autoscaling` | horizontalpodautoscalers |
| `storage.k8s.io` | storageclasses, persistentvolumes |

---

## ClusterRoles

ClusterRoles can be used for:
1. Cluster-wide permissions (across all namespaces)
2. Non-namespaced resources (nodes, PVs, StorageClasses)
3. Used with RoleBinding in a specific namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "namespaces", "nodes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses", "persistentvolumes"]
  verbs: ["get", "list", "watch"]
```

### Built-in ClusterRoles

AKS includes several built-in ClusterRoles:

| ClusterRole | Access Level |
|-------------|-------------|
| `cluster-admin` | Full cluster access |
| `admin` | Full access within a namespace |
| `edit` | Read/write most resources |
| `view` | Read-only access |

---

## RoleBindings

### Bind a Role to a User

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
- kind: User
  name: "jane@example.com"       # Azure AD user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader               # Must exist in same namespace
  apiGroup: rbac.authorization.k8s.io
```

### Bind a ClusterRole to a Group within a Namespace

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-editor
  namespace: development
subjects:
- kind: Group
  name: "dev-team-azure-ad-group-id"    # Azure AD group object ID
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit                             # Built-in ClusterRole
  apiGroup: rbac.authorization.k8s.io
```

### ClusterRoleBinding (Cluster-Wide)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-team-admin
subjects:
- kind: Group
  name: "platform-team-azure-ad-group-id"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

---

## ServiceAccounts

A **ServiceAccount** provides an identity for pods. Pods use ServiceAccounts to authenticate to the Kubernetes API.

### Creating a ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    # Required for Workload Identity (Module 06)
    azure.workload.identity/client-id: "<managed-identity-client-id>"
```

### Binding a Role to a ServiceAccount

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-app-sa-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-app-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### Using a ServiceAccount in a Pod

```yaml
spec:
  serviceAccountName: my-app-sa    # Assign SA to pod
  automountServiceAccountToken: true
  containers:
  - name: app
    image: my-app:1.0
```

### Default ServiceAccount

Every namespace has a `default` ServiceAccount. Pods use it if no SA is specified. By default, it has minimal permissions and should not be granted additional access. Always create dedicated ServiceAccounts for each application.

---

## Azure RBAC for AKS

When AKS is configured with Azure RBAC (`--enable-azure-rbac`), you can manage cluster access using Azure role assignments.

### Built-in Azure RBAC Roles for AKS

| Azure Role | Kubernetes Equivalent |
|-----------|----------------------|
| Azure Kubernetes Service RBAC Cluster Admin | cluster-admin |
| Azure Kubernetes Service RBAC Admin | admin |
| Azure Kubernetes Service RBAC Writer | edit |
| Azure Kubernetes Service RBAC Reader | view |

```bash
# Grant a user the AKS RBAC Admin role for a specific cluster
az role assignment create \
  --role "Azure Kubernetes Service RBAC Admin" \
  --assignee "jane@example.com" \
  --scope $(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)

# Grant cluster admin via Azure RBAC
az role assignment create \
  --role "Azure Kubernetes Service RBAC Cluster Admin" \
  --assignee-object-id "<group-object-id>" \
  --assignee-principal-type Group \
  --scope $(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query id -o tsv)
```

---

## RBAC for Teams: Namespace-Based Multi-Tenancy

```yaml
# Namespace for team A
apiVersion: v1
kind: Namespace
metadata:
  name: team-a
  labels:
    team: team-a
---
# Team A admin role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a
subjects:
- kind: Group
  name: "team-a-aad-group-id"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin            # Full access within namespace
  apiGroup: rbac.authorization.k8s.io
---
# Team A read-only (view) for namespace team-b
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-view-team-b
  namespace: team-b
subjects:
- kind: Group
  name: "team-a-aad-group-id"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Checking Permissions

```bash
# Check if current user can perform an action
kubectl auth can-i get pods
kubectl auth can-i create deployments -n production
kubectl auth can-i delete secrets

# Check permissions for a specific user
kubectl auth can-i get pods --as jane@example.com
kubectl auth can-i get pods --as system:serviceaccount:default:my-app-sa

# List all permissions for current user in a namespace
kubectl auth can-i --list -n production

# Get current user identity
kubectl auth whoami
```

---

## Best Practices

1. **Principle of least privilege**: Grant only the minimum permissions required
2. **Dedicated ServiceAccounts**: One SA per application, never use `default`
3. **Namespace isolation**: Use namespaces to separate teams/environments
4. **Audit regularly**: Review and remove unused roles and bindings
5. **Use Azure RBAC**: Leverage Azure AD groups for easier management
6. **No cluster-admin for humans**: Use namespace-scoped admin roles instead
7. **Monitor RBAC**: Use Azure Audit Logs to track permission usage

---

## Summary

| Resource | Scope | Use When |
|----------|-------|----------|
| Role | Namespace | Namespace-specific permissions |
| ClusterRole | Cluster | Cross-namespace or cluster resources |
| RoleBinding | Namespace | Assign role to user/group/SA in namespace |
| ClusterRoleBinding | Cluster | Assign ClusterRole globally |
| ServiceAccount | Namespace | Pod identity for K8s API access |

---

## Next Steps

Continue to [02-managed-identity.md](./02-managed-identity.md) to learn about Azure Managed Identity for pods.
