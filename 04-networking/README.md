# 04 - Networking

This module covers Kubernetes networking fundamentals, the NGINX Ingress controller, and Network Policies for securing traffic in AKS.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Explain how Pod-to-Pod networking works in AKS
- Install and configure the NGINX Ingress controller
- Create Ingress resources with path-based and host-based routing
- Implement Network Policies to restrict traffic between Pods
- Configure TLS termination with cert-manager

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-networking-concepts.md](./01-networking-concepts.md) | Pod networking, CNI, DNS, Service networking | 30 min |
| [02-ingress.md](./02-ingress.md) | NGINX Ingress, path routing, TLS, annotations | 45 min |
| [03-network-policies.md](./03-network-policies.md) | Allow/deny traffic rules, namespace isolation | 30 min |

---

## Prerequisites

- Completed [03-workloads](../03-workloads/README.md)
- A running AKS cluster with kubectl access

---

## Key Networking Concepts

| Concept | Description |
|---------|-------------|
| Pod CIDR | IP range assigned to pods |
| Service CIDR | IP range for Service ClusterIPs |
| ClusterIP | Virtual IP for Service |
| NodePort | Port exposed on each node |
| Ingress | HTTP/HTTPS routing layer |
| Network Policy | Firewall rules for pod-to-pod traffic |

---

## Next Steps

After completing this module, proceed to [05 - Storage](../05-storage/README.md).
