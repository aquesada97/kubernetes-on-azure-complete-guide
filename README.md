# AKS Training

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Contributors](https://img.shields.io/github/contributors/your-org/aks-training)
![Last Commit](https://img.shields.io/github/last-commit/your-org/aks-training)

Welcome to the **AKS Training** repository — a comprehensive, hands-on guide to Azure Kubernetes Service (AKS). Whether you are a developer, DevOps engineer, or cloud architect, this training helps you master AKS from fundamentals to advanced production patterns.

---

## 📋 Table of Contents

| Module | Description |
|--------|-------------|
| [00 - Prerequisites](./00-prerequisites/README.md) | Tools, Azure subscription setup, environment variables |
| [01 - Fundamentals](./01-fundamentals/README.md) | Kubernetes & AKS architecture, core concepts |
| [02 - Getting Started](./02-getting-started/README.md) | Create your first AKS cluster, kubectl basics |
| [03 - Workloads](./03-workloads/README.md) | Deployments, Services, ConfigMaps, Secrets, StatefulSets |
| [04 - Networking](./04-networking/README.md) | CNI, Ingress, Network Policies |
| [05 - Storage](./05-storage/README.md) | Persistent Volumes, Storage Classes, Azure Disk & Files |
| [06 - Security](./06-security/README.md) | RBAC, Managed Identity, Azure Key Vault CSI |
| [07 - Monitoring](./07-monitoring/README.md) | Azure Monitor, Container Insights, Prometheus & Grafana |
| [08 - Scaling](./08-scaling/README.md) | HPA, VPA, Cluster Autoscaler |
| [09 - Advanced](./09-advanced/README.md) | Upgrades, Multi-tenancy, Disaster Recovery |
| [Labs](./labs/README.md) | Hands-on labs with step-by-step instructions |

---

## 📖 Description

This training repository provides structured, progressive learning for Azure Kubernetes Service. Each module builds on the previous one and includes:

- **Conceptual explanations** with diagrams and tables
- **CLI commands** with flag-by-flag explanations
- **Complete YAML manifests** ready to apply
- **Hands-on labs** with estimated completion times

---

## ✅ Prerequisites Summary

Before starting, ensure you have the following:

- An **Azure subscription** with Contributor access
- **Azure CLI** (v2.50+) installed and logged in
- **kubectl** (v1.27+)
- **Helm** (v3.12+)
- **VS Code** with Kubernetes and Azure extensions

> See [00-prerequisites/README.md](./00-prerequisites/README.md) for detailed installation instructions.

---

## 🚀 How to Use This Repository

1. **Start with Prerequisites** — install all required tools
2. **Work through modules in order** — each module builds on the last
3. **Complete the labs** — reinforce concepts with hands-on practice
4. **Refer back as needed** — use this as a reference during real-world work

### Recommended Learning Path

```
Prerequisites → Fundamentals → Getting Started → Workloads → Networking → Storage → Security → Monitoring → Scaling → Advanced → Labs
```

For a **quick start** (3–4 hours), focus on:
- Prerequisites, Fundamentals, Getting Started, Workloads, and Lab 01

For a **full course** (2–3 days), complete all modules and labs.

---

## 🤝 Contributing

We welcome contributions! Please see [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines.

---

## 📜 License

This project is licensed under the MIT License.


---

## 🔗 Related Training Guides

This guide is part of a series of Azure container platform training resources:

| Guide | Description |
|-------|-------------|
| 🐳 [Azure Container Registry (ACR)](https://github.com/aquesada97/azure-container-registry-complete-guide) | Build, store, and manage container images — integrates directly with AKS |
| 🔴 [Azure Red Hat OpenShift (ARO)](https://github.com/aquesada97/azure-redhat-openshift-complete-guide) | Enterprise OpenShift on Azure — security, S2I builds, Operator Framework |

> **Recommended learning path:** ACR → AKS → ARO
