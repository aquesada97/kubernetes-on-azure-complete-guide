# AKS Architecture

Azure Kubernetes Service (AKS) is a managed Kubernetes offering from Microsoft Azure. It simplifies cluster lifecycle management by handling the control plane on your behalf.

---

## Managed Control Plane

In AKS, the **control plane is fully managed by Microsoft**. You do not pay for, maintain, or directly access the control plane components (API server, etcd, scheduler, controller manager). This includes:

- Automatic Kubernetes version upgrades (when configured)
- Control plane high availability
- etcd backups
- Security patches for control plane components

### What You Manage vs. What Azure Manages

| Responsibility | Azure (Managed) | Customer |
|---------------|-----------------|----------|
| API Server | ✅ | |
| etcd | ✅ | |
| Scheduler | ✅ | |
| Controller Manager | ✅ | |
| Worker Nodes | | ✅ |
| Node OS patches | | ✅ (or Azure with node auto-upgrade) |
| Application workloads | | ✅ |
| Networking configuration | | ✅ |
| RBAC policies | | ✅ |

---

## AKS Resource Group Structure

When you create an AKS cluster, Azure creates two resource groups:

### 1. The Cluster Resource Group (you choose the name)

Contains the AKS cluster resource itself. This is the resource group you specify with `--resource-group`.

**Contains:**
- The AKS cluster resource
- Any explicitly created resources (ACR, Log Analytics Workspace)

### 2. The Node Resource Group (automatically created)

Also called the **MC_ resource group** (short for Managed Cluster). Named automatically as `MC_<resourceGroup>_<clusterName>_<region>`.

**Contains:**
- Virtual Machine Scale Sets (VMSS) for node pools
- Virtual Network (if not using custom VNet)
- Network Security Groups
- Azure Load Balancers
- Public IP addresses
- Managed Disks for nodes

> ⚠️ **Warning:** Do NOT manually modify resources in the MC_ resource group. AKS manages these resources and manual changes can cause cluster instability or be overwritten.

```bash
az group list --query "[?starts_with(name, 'MC_')]" --output table

az resource list \
  --resource-group "MC_rg-aks-training_aks-training-cluster_eastus" \
  --output table
```

---

## Node Pools

Node pools are groups of nodes with the same configuration (VM size, OS, Kubernetes version). AKS supports multiple node pools per cluster.

### System Node Pool

Every AKS cluster requires at least one **system node pool**. It:
- Runs critical Kubernetes system pods (CoreDNS, kube-proxy, metrics-server)
- Has a `CriticalAddonsOnly=true:NoSchedule` taint by default
- Should NOT run application workloads (use user node pools for that)
- Minimum: 1 node (3+ recommended for production)

### User Node Pool

**User node pools** run your application workloads. You can have multiple user node pools with different configurations:
- Different VM sizes (CPU-optimized, GPU, memory-optimized)
- Different OS (Linux or Windows)
- Different node counts and autoscaling settings

```bash
az aks nodepool add \
  --cluster-name aks-training-cluster \
  --resource-group rg-aks-training \
  --name userpool \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --mode User

az aks nodepool list \
  --cluster-name aks-training-cluster \
  --resource-group rg-aks-training \
  --output table
```

### Node Pool Comparison

| Feature | System Pool | User Pool |
|---------|-------------|-----------|
| Purpose | System workloads | Application workloads |
| Required | Yes (at least 1) | No |
| Taint | CriticalAddonsOnly | None (by default) |
| Min nodes (production) | 3 | 1+ |
| VM types | Standard | Any supported VM |
| Windows support | No | Yes |

### Spot Node Pools

Spot node pools use Azure Spot VMs — unused Azure capacity at reduced cost (up to 90% discount). Suitable for:
- Batch processing
- Development/test workloads
- Fault-tolerant applications

```bash
az aks nodepool add \
  --cluster-name aks-training-cluster \
  --resource-group rg-aks-training \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 10
```

---

## Supported Operating Systems

### Linux Nodes

- Default OS for AKS nodes
- Supports Ubuntu 22.04 and Azure Linux (CBL-Mariner)
- Runs all Kubernetes system components
- Required for system node pool

```bash
az aks nodepool add \
  --os-type Linux \
  --os-sku Ubuntu \
  --name linuxpool ...

az aks nodepool add \
  --os-type Linux \
  --os-sku AzureLinux \
  --name azlinuxpool ...
```

### Windows Nodes

- Supported in user node pools only
- Required for Windows Server containers
- Uses Windows Server 2022 (recommended) or 2019

```bash
az aks nodepool add \
  --cluster-name aks-training-cluster \
  --resource-group rg-aks-training \
  --name winpool \
  --os-type Windows \
  --os-sku Windows2022 \
  --node-count 2 \
  --node-vm-size Standard_D4s_v3
```

---

## AKS Regions

AKS is available in most Azure regions. Key considerations:

- Choose a region close to your users for lower latency
- Some features (e.g., Availability Zones) are only available in specific regions
- Paired regions are important for disaster recovery

```bash
az provider show \
  --namespace Microsoft.ContainerService \
  --query "resourceTypes[?resourceType=='managedClusters'].locations[]" \
  --output table
```

---

## SLA Tiers

AKS offers two tiers for the control plane:

### Free Tier
- No SLA for API server uptime
- Best for: dev/test, non-critical workloads

### Standard Tier (recommended for production)
- **99.9% SLA** for API server availability (without Availability Zones)
- **99.95% SLA** with Availability Zones enabled

```bash
az aks create \
  --name aks-training-cluster \
  --resource-group rg-aks-training \
  --tier standard \
  ...
```

### Uptime SLA Comparison

| Tier | API Server SLA | Availability Zones SLA | Cost |
|------|---------------|------------------------|------|
| Free | No SLA | No SLA | Free |
| Standard | 99.9% | 99.95% | ~$0.10/hour |

---

## Availability Zones

Deploy nodes across Azure Availability Zones for higher resilience within a region.

```bash
az aks create \
  --name aks-training-cluster \
  --resource-group rg-aks-training \
  --zones 1 2 3 \
  --node-count 3 \
  ...
```

With 3 nodes and 3 zones, each node lands in a different zone — your cluster survives a complete zone failure.

---

## Summary

| Concept | Key Points |
|---------|-----------|
| Managed Control Plane | Azure manages API server, etcd, scheduler |
| Node Resource Group | MC_ group holds VMSS, LB, NICs — don't modify manually |
| System Node Pool | Required, runs system pods, dedicated |
| User Node Pool | Runs app workloads, multiple allowed, supports Windows |
| SLA Tiers | Free (no SLA) vs Standard (99.9%/99.95%) |
| Availability Zones | Spread nodes across zones for HA |

---

## Next Steps

Continue to [03-aks-components.md](./03-aks-components.md) to understand the components running inside AKS nodes.
