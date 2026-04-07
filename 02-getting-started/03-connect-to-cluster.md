# Connect to an AKS Cluster

This document covers how to configure kubectl to connect to AKS clusters, understand kubeconfig, and manage multiple cluster contexts.

---

## Getting Credentials

```bash
az aks get-credentials \
  --name aks-training-cluster \
  --resource-group rg-aks-training

kubectl get nodes
```

### Flags

| Flag | Description |
|------|-------------|
| `--admin` | Get admin credentials (bypasses Azure AD auth) |
| `--overwrite-existing` | Overwrite existing context if it exists |
| `--file <path>` | Write to a specific kubeconfig file instead of default |
| `--context <name>` | Set a custom context name |

```bash
az aks get-credentials \
  --name aks-training-cluster \
  --resource-group rg-aks-training \
  --admin

az aks get-credentials \
  --name aks-training-cluster \
  --resource-group rg-aks-training \
  --file ~/.kube/aks-training
```

---

## Understanding kubeconfig

The kubeconfig file (`~/.kube/config`) stores cluster connection information.

```bash
kubectl config view
kubectl config view --raw
```

### kubeconfig Structure

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
    certificate-authority-data: <base64-encoded-ca-cert>
    server: https://aks-training-cluster-abc123.hcp.eastus.azmk8s.io:443
  name: aks-training-cluster

users:
- name: clusterUser_rg-aks-training_aks-training-cluster
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: kubelogin
      args:
      - get-token
      - --login
      - azurecli
      - --server-id
      - <app-id>

contexts:
- context:
    cluster: aks-training-cluster
    user: clusterUser_rg-aks-training_aks-training-cluster
    namespace: default
  name: aks-training-cluster

current-context: aks-training-cluster
```

---

## Managing Contexts

A **context** is a combination of: cluster + user + namespace.

```bash
kubectl config get-contexts
kubectl config current-context
kubectl config use-context aks-training-cluster
kubectl config rename-context old-name new-name
kubectl config delete-context my-old-cluster
kubectl config get-contexts -o name
```

### Example Multi-Cluster Context Switching

```bash
az aks get-credentials --name dev-cluster --resource-group rg-dev
az aks get-credentials --name prod-cluster --resource-group rg-prod

kubectl config get-contexts
# CURRENT   NAME           CLUSTER        AUTHINFO                    NAMESPACE
# *         dev-cluster    dev-cluster    clusterUser_rg-dev_dev...   default
#           prod-cluster   prod-cluster   clusterUser_rg-prod_...     default

kubectl config use-context prod-cluster
kubectl config use-context dev-cluster
```

---

## kubelogin for Azure AD Authentication

When AKS is configured with Azure AD (Entra ID) integration, kubectl uses **kubelogin** to handle authentication.

### Login Methods

```bash
# Interactive browser login (default)
kubelogin convert-kubeconfig -l azurecli

# Device code (for headless environments)
kubelogin convert-kubeconfig -l devicecode

# Service principal (for automation)
kubelogin convert-kubeconfig -l spn \
  --client-id <app-id> \
  --client-secret <secret>

# Managed identity (in Azure-hosted environments)
kubelogin convert-kubeconfig -l msi
```

### Convert Kubeconfig for Non-Interactive Auth

```bash
export KUBECONFIG=~/.kube/config
kubelogin convert-kubeconfig -l azurecli

kubectl get nodes
```

---

## Multiple Kubeconfig Files

```bash
export KUBECONFIG=~/.kube/aks-training
kubectl get nodes

export KUBECONFIG=~/.kube/config:~/.kube/aks-training:~/.kube/local-cluster
kubectl config view --merge --flatten
kubectl config view --merge --flatten > ~/.kube/merged-config
```

---

## kubectx and kubens (Optional Tools)

```bash
# macOS
brew install kubectx

# Linux
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# Usage
kubectx               # List contexts
kubectx prod-cluster  # Switch context
kubens                # List namespaces
kubens kube-system    # Switch namespace
```

---

## Verify Connection and Access

```bash
kubectl cluster-info
kubectl config view -o jsonpath='{.clusters[0].cluster.server}'
kubectl auth whoami
kubectl auth can-i create deployments
kubectl auth can-i create deployments --namespace production
kubectl auth can-i --list --namespace default
```

---

## Troubleshooting Connection Issues

```bash
# "Unable to connect to the server"
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --overwrite-existing

# "You must be logged in to the server (Unauthorized)"
az login
kubelogin convert-kubeconfig -l azurecli

# Check AKS cluster state
az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query provisioningState
```

---

## Next Steps

Proceed to [03 - Workloads](../03-workloads/README.md) to start deploying applications to your cluster.
