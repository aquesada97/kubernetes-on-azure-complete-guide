# Hands-On Labs

Welcome to the AKS Training Labs. Each lab builds on the concepts from the modules and provides step-by-step, practical exercises.

---

## 📋 Labs Overview

| Lab | Title | Covers | Est. Time | Prerequisites |
|-----|-------|--------|-----------|---------------|
| [Lab 01](./lab-01-deploy-first-app/README.md) | Deploy Your First App | Deployments, Services, kubectl | 45 min | Modules 01-03 |
| [Lab 02](./lab-02-ingress/README.md) | Configure Ingress | NGINX Ingress, host routing, TLS | 45 min | Lab 01 |
| [Lab 03](./lab-03-storage/README.md) | Persistent Storage | PVC, Azure Disk, Azure Files | 40 min | Lab 02 |
| [Lab 04](./lab-04-scaling/README.md) | Autoscaling | HPA, load testing, CA | 45 min | Lab 03 |
| [Lab 05](./lab-05-rbac/README.md) | RBAC and Security | ServiceAccounts, Roles, Bindings | 40 min | Lab 04 |

---

## Prerequisites for All Labs

- Completed relevant training modules
- Running AKS cluster (`aks-training-cluster`)
- kubectl configured and connected
- `RESOURCE_GROUP` and `CLUSTER_NAME` environment variables set

```bash
export RESOURCE_GROUP="rg-aks-training"
export CLUSTER_NAME="aks-training-cluster"
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
kubectl get nodes
```

---

## Lab Environment Setup

```bash
# Create lab namespace
kubectl create namespace labs

# Set as default namespace for labs
kubectl config set-context --current --namespace=labs
```

---

## 🧹 Cleanup After Labs

```bash
# Delete lab namespace and all resources
kubectl delete namespace labs

# Or delete the entire resource group
az group delete --name $RESOURCE_GROUP --yes --no-wait
```

---

## Helpful Aliases for Labs

```bash
alias k='kubectl'
alias ka='kubectl apply -f'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
```
