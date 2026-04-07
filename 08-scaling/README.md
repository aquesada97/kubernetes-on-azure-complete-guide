# 08 - Scaling

This module covers the three scaling mechanisms available in AKS: Horizontal Pod Autoscaler (HPA), Vertical Pod Autoscaler (VPA), and the Cluster Autoscaler.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Configure HPA to scale pods based on CPU and custom metrics
- Configure VPA to automatically right-size pod resource requests
- Enable and tune the Cluster Autoscaler for node pool scaling
- Understand when to use each autoscaling mechanism
- Implement KEDA for event-driven scaling

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-hpa.md](./01-hpa.md) | HPA with CPU, memory, and custom metrics | 35 min |
| [02-vpa.md](./02-vpa.md) | VPA update modes, right-sizing recommendations | 25 min |
| [03-cluster-autoscaler.md](./03-cluster-autoscaler.md) | Node pool autoscaling, expander strategies | 30 min |

---

## Prerequisites

- Completed [07-monitoring](../07-monitoring/README.md)
- A running AKS cluster with metrics-server installed

---

## Scaling Decision Guide

```
Load increased?
├── Need more pod instances? → HPA (horizontal)
├── Pods need more CPU/memory? → VPA (vertical)
└── No nodes available for new pods? → Cluster Autoscaler
```

## Scaling Layers

| Layer | Mechanism | What Scales |
|-------|-----------|-------------|
| Pod replicas | HPA | Number of pods in a Deployment |
| Pod resources | VPA | CPU/memory requests per pod |
| Nodes | Cluster Autoscaler | Number of nodes in a node pool |
| Event-driven | KEDA | Pods based on queue depth, events |

---

## Next Steps

After completing this module, proceed to [09 - Advanced](../09-advanced/README.md).
