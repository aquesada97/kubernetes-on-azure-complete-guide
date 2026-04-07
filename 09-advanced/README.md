# 09 - Advanced

This module covers advanced AKS topics for production operations: upgrade strategies, multi-tenancy patterns, and disaster recovery.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Plan and execute zero-downtime AKS cluster and node upgrades
- Implement multi-tenant cluster patterns with namespace isolation
- Design and implement a disaster recovery strategy for AKS
- Configure GitOps with Flux for continuous delivery

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-upgrade-strategy.md](./01-upgrade-strategy.md) | Cluster upgrades, node OS upgrades, auto-upgrade | 35 min |
| [02-multi-tenancy.md](./02-multi-tenancy.md) | Namespace isolation, quotas, LimitRanges, RBAC | 30 min |
| [03-disaster-recovery.md](./03-disaster-recovery.md) | Backup/restore, multi-region, RTO/RPO | 35 min |

---

## Prerequisites

- Completed all previous modules
- Production-like AKS cluster

---

## Advanced Topics Summary

| Topic | Key Tool / Pattern |
|-------|-------------------|
| Cluster upgrades | az aks upgrade + auto-upgrade channels |
| Node upgrades | Node OS upgrade channels + node image updates |
| Multi-tenancy | Namespaces + RBAC + ResourceQuotas |
| GitOps | Flux v2 + Azure Arc GitOps |
| Disaster Recovery | Velero + Azure Backup + multi-region |
| Service Mesh | Istio, Linkerd, Open Service Mesh |
| Policy | Azure Policy + OPA Gatekeeper |

---

## Next Steps

After completing all modules, proceed to the [Labs](../labs/README.md) for hands-on practice.
