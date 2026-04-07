# 02 - Getting Started

This module walks you through creating your first AKS cluster, learning kubectl fundamentals, and connecting to your cluster.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Create an AKS cluster using Azure CLI
- Understand every flag used in the cluster creation command
- Use kubectl to inspect and manage cluster resources
- Configure kubeconfig and manage multiple cluster contexts

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-create-cluster.md](./01-create-cluster.md) | az aks create, Portal, Bicep/Terraform | 30 min |
| [02-kubectl-basics.md](./02-kubectl-basics.md) | kubectl get, describe, logs, exec, apply | 30 min |
| [03-connect-to-cluster.md](./03-connect-to-cluster.md) | kubeconfig, contexts, multi-cluster | 20 min |

---

## Prerequisites

- Completed [00-prerequisites](../00-prerequisites/README.md)
- Completed [01-fundamentals](../01-fundamentals/README.md)
- Azure subscription with Contributor access
- Environment variables set from Module 00

---

## Quick Reference

```bash
# Create cluster
az aks create --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP ...

# Connect kubectl
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP

# Verify connection
kubectl get nodes
```

---

## Next Steps

After completing this module, proceed to [03 - Workloads](../03-workloads/README.md).
