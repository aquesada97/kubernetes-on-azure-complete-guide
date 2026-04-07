# 01 - Fundamentals

This module establishes a solid foundation in Kubernetes concepts and AKS-specific architecture before you start working with clusters.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Explain the difference between containers and virtual machines
- Describe the Kubernetes control plane and worker node architecture
- Identify core Kubernetes objects and their purpose
- Explain AKS-specific architecture including managed control planes and node pools
- Understand the components that run on AKS worker nodes

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-kubernetes-basics.md](./01-kubernetes-basics.md) | Containers, K8s architecture, core objects | 45 min |
| [02-aks-architecture.md](./02-aks-architecture.md) | AKS managed control plane, node pools, SLAs | 30 min |
| [03-aks-components.md](./03-aks-components.md) | CoreDNS, CNI, containerd, Azure integration | 30 min |

---

## Prerequisites

- Completed [00-prerequisites](../00-prerequisites/README.md)
- Basic familiarity with Linux command line
- Understanding of what an API is

---

## Key Concepts to Master

Before proceeding to Module 02, ensure you can answer:

1. What is the role of the Kubernetes API Server?
2. What is the difference between a Pod and a Deployment?
3. Why does AKS use a managed control plane?
4. What is the difference between Azure CNI and Kubenet?
5. What node pool types exist in AKS and when do you use each?

---

## Next Steps

After completing this module, proceed to [02 - Getting Started](../02-getting-started/README.md).
