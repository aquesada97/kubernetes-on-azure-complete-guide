# 03 - Workloads

This module covers the core Kubernetes workload resources used to run applications on AKS, including Deployments, Services, ConfigMaps, Secrets, and StatefulSets.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Create and manage Deployments with rolling updates and rollbacks
- Expose applications using different Service types (ClusterIP, NodePort, LoadBalancer)
- Store and inject configuration using ConfigMaps and Secrets
- Deploy stateful applications using StatefulSets
- Apply resource requests and limits to pods

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-deployments.md](./01-deployments.md) | Deployment strategies, rolling updates, resource limits | 40 min |
| [02-services.md](./02-services.md) | ClusterIP, NodePort, LoadBalancer, ExternalName | 30 min |
| [03-configmaps-secrets.md](./03-configmaps-secrets.md) | Config injection, secret management, env vars, volumes | 30 min |
| [04-statefulsets.md](./04-statefulsets.md) | StatefulSets, headless services, ordered operations | 30 min |

---

## Prerequisites

- Completed [02-getting-started](../02-getting-started/README.md)
- A running AKS cluster with kubectl access

---

## Key Concepts to Master

Before proceeding to Module 04, ensure you can answer:

1. When would you choose a StatefulSet over a Deployment?
2. What is the difference between a ClusterIP and a LoadBalancer Service?
3. How do you reference a ConfigMap value as an environment variable in a Pod?
4. What is the difference between `maxSurge` and `maxUnavailable` in a rolling update strategy?
5. How do you safely store and consume secrets in a Pod?

---

## Quick Reference: Resource Requests and Limits

Always set resource requests and limits for production workloads:

```yaml
resources:
  requests:
    cpu: "250m"       # 0.25 CPU cores (guaranteed)
    memory: "256Mi"   # 256 MB RAM (guaranteed)
  limits:
    cpu: "500m"       # 0.5 CPU cores (maximum)
    memory: "512Mi"   # 512 MB RAM (maximum, OOMKilled if exceeded)
```

> 💡 **Tip:** Setting only `requests` (no limits) allows the pod to burst but risks node resource exhaustion. Setting both is a best practice.

---

## Next Steps

After completing this module, proceed to [04 - Networking](../04-networking/README.md).
