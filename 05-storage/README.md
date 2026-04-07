# 05 - Storage

This module covers persistent storage in AKS, including PersistentVolumes, PersistentVolumeClaims, StorageClasses, and Azure-native storage integrations.

---

## 🎯 Learning Objectives

By the end of this module, you will be able to:

- Explain the difference between PersistentVolumes and PersistentVolumeClaims
- Create and use Storage Classes for dynamic provisioning
- Mount Azure Disk and Azure Files storage in pods
- Choose the right storage type for different workloads
- Configure storage for StatefulSets

---

## 📚 Topics Covered

| File | Topic | Est. Time |
|------|-------|-----------|
| [01-persistent-volumes.md](./01-persistent-volumes.md) | PV, PVC, access modes, volume lifecycle | 35 min |
| [02-storage-classes.md](./02-storage-classes.md) | StorageClasses, Azure Disk, Azure Files, CSI drivers | 30 min |

---

## Prerequisites

- Completed [04-networking](../04-networking/README.md)
- A running AKS cluster

---

## Storage Decision Guide

| Storage Type | Access Mode | Use Case |
|-------------|-------------|----------|
| Azure Disk (Premium SSD) | ReadWriteOnce | Databases, single-pod stateful apps |
| Azure Files | ReadWriteMany | Shared config, CMS, multi-pod access |
| Azure Disk (Standard HDD) | ReadWriteOnce | Dev/test, low IOPS workloads |
| Azure Files Premium | ReadWriteMany | High-throughput shared workloads |
| emptyDir | Per-pod | Temp storage, caches, scratch space |
| hostPath | Per-node | DaemonSets, node-level access |

---

## Next Steps

After completing this module, proceed to [06 - Security](../06-security/README.md).
