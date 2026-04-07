# Lab 05: RBAC and Security

In this lab, you will configure Kubernetes RBAC by creating ServiceAccounts, Roles, and RoleBindings to implement least-privilege access control for applications and users.

**Estimated Time:** 40 minutes  
**Difficulty:** Intermediate

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

- Create ServiceAccounts for application pods
- Define Roles with specific permissions
- Bind Roles to ServiceAccounts and Users
- Test permissions using `kubectl auth can-i`
- Configure a pod to use a custom ServiceAccount
- Understand the difference between Role and ClusterRole

---

## Prerequisites

- Completed [Lab 04](../lab-04-scaling/README.md)
- AKS cluster with kubectl access

---

## Lab Setup

```bash
kubectl create namespace lab05
kubectl create namespace lab05-restricted
kubectl config set-context --current --namespace=lab05
```

---

## Part 1: Create ServiceAccount and Role

### 1.1 Apply the Manifests

```bash
cd labs/lab-05-rbac

kubectl apply -f manifests/serviceaccount.yaml
kubectl apply -f manifests/role.yaml
kubectl apply -f manifests/rolebinding.yaml

# Verify
kubectl get serviceaccounts -n lab05
kubectl get roles -n lab05
kubectl get rolebindings -n lab05
```

### 1.2 Inspect the Resources

```bash
# View the ServiceAccount
kubectl describe serviceaccount lab05-sa -n lab05

# View the Role
kubectl describe role pod-reader -n lab05

# View the RoleBinding
kubectl describe rolebinding pod-reader-binding -n lab05
```

---

## Part 2: Test Permissions with kubectl auth can-i

```bash
# Check if the SA can list pods in lab05
kubectl auth can-i list pods \
  --as system:serviceaccount:lab05:lab05-sa \
  --namespace lab05
# → yes

# Check if the SA can create pods (not allowed)
kubectl auth can-i create pods \
  --as system:serviceaccount:lab05:lab05-sa \
  --namespace lab05
# → no

# Check if the SA can list pods in another namespace (not allowed)
kubectl auth can-i list pods \
  --as system:serviceaccount:lab05:lab05-sa \
  --namespace lab05-restricted
# → no

# Check if the SA can delete pods (not allowed)
kubectl auth can-i delete pods \
  --as system:serviceaccount:lab05:lab05-sa \
  --namespace lab05
# → no

# List ALL permissions for the SA
kubectl auth can-i --list \
  --as system:serviceaccount:lab05:lab05-sa \
  --namespace lab05
```

---

## Part 3: Deploy a Pod with the ServiceAccount

### 3.1 Create a Pod Using the ServiceAccount

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rbac-test-pod
  namespace: lab05
spec:
  serviceAccountName: lab05-sa         # Use our custom SA
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ["sleep", "3600"]
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "100m"
        memory: "128Mi"
EOF

kubectl get pod rbac-test-pod -n lab05
```

### 3.2 Test Permissions From Inside the Pod

```bash
# Exec into the pod
kubectl exec -it rbac-test-pod -n lab05 -- /bin/bash

# Inside the pod — these should work (allowed by Role)
kubectl get pods -n lab05
kubectl get services -n lab05
kubectl get deployments -n lab05

# These should fail (not in Role)
kubectl create deployment test --image=nginx -n lab05
# Error from server (Forbidden)

kubectl delete pod rbac-test-pod -n lab05
# Error from server (Forbidden)

# Cross-namespace — should fail
kubectl get pods -n lab05-restricted
# Error from server (Forbidden)

exit
```

---

## Part 4: Create a ClusterRole and ClusterRoleBinding

### 4.1 Read-Only Cluster View

```bash
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab05-cluster-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces", "persistentvolumes"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
EOF

# Bind cluster role to the SA
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab05-cluster-reader-binding
subjects:
- kind: ServiceAccount
  name: lab05-sa
  namespace: lab05
roleRef:
  kind: ClusterRole
  name: lab05-cluster-reader
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 4.2 Verify Cluster-Wide Permissions

```bash
# Node list should now work (cluster-wide)
kubectl auth can-i list nodes \
  --as system:serviceaccount:lab05:lab05-sa
# → yes

# StorageClasses
kubectl auth can-i list storageclasses \
  --as system:serviceaccount:lab05:lab05-sa
# → yes

# But cannot create nodes
kubectl auth can-i create nodes \
  --as system:serviceaccount:lab05:lab05-sa
# → no
```

---

## Part 5: ResourceQuota with RBAC

Demonstrate how RBAC works alongside ResourceQuota:

```bash
# Create a ResourceQuota in the restricted namespace
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: restricted-quota
  namespace: lab05-restricted
spec:
  hard:
    pods: "5"
    requests.cpu: "1"
    requests.memory: "1Gi"
EOF

# Create a role that can deploy in restricted namespace
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: restricted-deployer
  namespace: lab05-restricted
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create", "update", "patch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF
```

---

## Part 6: Audit RBAC

```bash
# View all role bindings across namespaces
kubectl get rolebindings -A
kubectl get clusterrolebindings

# Find which subjects have admin in your namespaces
kubectl get rolebindings -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.roleRef.name}{"\n"}{end}'

# Check which ClusterRoles are bound to a user
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]?.name == "lab05-sa") | .metadata.name'
```

---

## Cleanup

```bash
kubectl delete namespace lab05
kubectl delete namespace lab05-restricted
kubectl delete clusterrole lab05-cluster-reader
kubectl delete clusterrolebinding lab05-cluster-reader-binding
```

---

## ✅ Lab Completion Checklist

- [ ] Created ServiceAccount, Role, and RoleBinding
- [ ] Verified permissions with `kubectl auth can-i`
- [ ] Deployed a pod using the custom ServiceAccount
- [ ] Tested allowed and denied operations from inside the pod
- [ ] Created a ClusterRole and ClusterRoleBinding
- [ ] Verified cluster-wide permissions

---

## 🤔 Discussion Questions

1. What is the difference between Role and ClusterRole?
2. Why should you never grant cluster-admin to an application ServiceAccount?
3. What does `--as` in `kubectl auth can-i` simulate?
4. How would you audit who has `cluster-admin` in your cluster?

---

## 🎉 Congratulations!

You have completed all 5 AKS Training Labs! You now have hands-on experience with:
- Deploying and managing applications
- Ingress routing
- Persistent storage
- Autoscaling
- RBAC and security

Continue exploring the advanced topics in [Module 09](../../09-advanced/README.md) for production-ready patterns.
