# 06 - Security

This module covers the essential security patterns for AKS: Role-Based Access Control (RBAC), Azure Managed Identity for workloads, and the Azure Key Vault CSI Driver for secrets management.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Configure Kubernetes RBAC with Roles, ClusterRoles, and Bindings
- Use Azure Managed Identity to grant workloads access to Azure resources
- Implement the Azure Key Vault CSI Driver to inject secrets into pods
- Apply Pod Security Standards (restricted, baseline, privileged)
- Configure cluster RBAC with Azure AD (Entra ID) groups

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-rbac.md](./01-rbac.md) | Roles, ClusterRoles, Bindings, ServiceAccounts | 45 min |
| [02-managed-identity.md](./02-managed-identity.md) | Workload Identity, pod-level Azure access | 30 min |
| [03-azure-keyvault-csi.md](./03-azure-keyvault-csi.md) | CSI Driver, SecretProviderClass, secrets sync | 30 min |

---

## Prerequisites

- Completed [05-storage](../05-storage/README.md)
- A running AKS cluster

---

## Security Best Practices Summary

| Practice | Action |
|---------|--------|
| Least privilege | Grant only the permissions needed |
| No secrets in code | Use Key Vault CSI or Workload Identity |
| Pod security | Run as non-root, read-only filesystem |
| Network isolation | Use Network Policies |
| Image security | Use ACR, scan images, use minimal base images |
| Audit logs | Enable AKS audit logging |

---

## Next Steps

After completing this module, proceed to [07 - Monitoring](../07-monitoring/README.md).
