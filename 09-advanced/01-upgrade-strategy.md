# AKS Upgrade Strategy

Keeping your AKS cluster up to date is critical for security, feature access, and support. This document covers all upgrade dimensions: Kubernetes version, node images, and OS patches.

---

## Upgrade Dimensions

| What | How | Frequency |
|------|-----|-----------|
| Kubernetes control plane version | `az aks upgrade` | Every 3-6 months |
| Kubernetes node pool version | `az aks nodepool upgrade` | After control plane |
| Node OS image | Node image upgrade | Weekly (security patches) |
| Node OS patches | NodeImage channel auto-upgrade | Automatic |

---

## Check Available Upgrades

```bash
# Check available Kubernetes versions in region
az aks get-versions --location eastus --output table

# Check available upgrades for your cluster
az aks get-upgrades \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table

# Check current cluster version
az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[kubernetesVersion, currentKubernetesVersion]"

# Check node pool versions
az aks nodepool list \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[].{name:name, k8sVersion:currentOrchestratorVersion, nodeImageVersion:nodeImageVersion}" \
  --output table
```

---

## Kubernetes Version Upgrade

### Manual Upgrade Process

The recommended order:
1. Upgrade the control plane
2. Upgrade each node pool (one at a time)

```bash
# Step 1: Upgrade control plane only
az aks upgrade \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --kubernetes-version 1.30.0 \
  --control-plane-only     # Don't upgrade node pools yet

# Verify control plane upgraded
az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query kubernetesVersion

# Step 2: Upgrade system node pool
az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name systempool \
  --kubernetes-version 1.30.0

# Step 3: Upgrade user node pool
az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name userpool \
  --kubernetes-version 1.30.0
```

### Upgrade with Max Surge

Max surge controls how many extra nodes are added during upgrade:

```bash
az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name userpool \
  --kubernetes-version 1.30.0 \
  --max-surge 33%         # Add 33% extra nodes (e.g., 3 nodes → 1 extra)
```

With `max-surge: 33%` and 3 nodes:
1. Add 1 new node (now 4 total)
2. Cordon and drain node 1
3. New pod scheduling on new node
4. Remove old node 1
5. Repeat for nodes 2 and 3

```bash
# Set max surge on node pool permanently
az aks nodepool update \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name userpool \
  --max-surge 1
```

---

## Auto-Upgrade Channels

Configure automatic Kubernetes version upgrades:

```bash
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --auto-upgrade-channel stable
```

### Channels

| Channel | Description | Recommended For |
|---------|-------------|-----------------|
| `none` | No automatic upgrades | Manual control needed |
| `patch` | Latest patch for current minor (1.29.3 → 1.29.5) | Production |
| `stable` | Latest stable minor - 1 (stays one version behind latest) | Production |
| `rapid` | Latest supported minor | Dev/test |
| `node-image` | Only upgrade node OS images | Node patching only |

---

## Node OS Image Upgrades

Node image upgrades update the OS image on nodes (new security patches, kernel updates) without changing the Kubernetes version.

```bash
# Check available node image versions
az aks nodepool get-upgrades \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --nodepool-name systempool \
  --output table

# Upgrade node images in a pool
az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name systempool \
  --node-image-only           # Only upgrade OS image, not K8s version
```

### Auto Node Image Upgrade

```bash
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --node-os-upgrade-channel NodeImage

# Options:
# None         - No automatic node image upgrades
# Unmanaged    - OS patching via unattended-upgrade (Linux) or Windows Update
# SecurityPatch - Only security patches (not full node image)
# NodeImage    - Full node image upgrade (recommended)
```

---

## Planned Maintenance Windows

Schedule upgrades during off-peak hours:

```bash
# Create maintenance configuration
az aks maintenanceconfiguration add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name default \
  --weekday Saturday \
  --start-hour 2           # 2:00 AM - 3:00 AM UTC

# View maintenance window
az aks maintenanceconfiguration list \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP
```

---

## Pre-Upgrade Checklist

Before upgrading a production cluster:

```bash
# 1. Back up important resources
kubectl get all --all-namespaces -o yaml > cluster-backup.yaml

# 2. Check cluster health
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed

# 3. Check PodDisruptionBudgets
kubectl get pdb -A

# 4. Check deprecated API usage (use kubectl-convert or pluto)
# Install pluto
curl -L https://github.com/FairwindsOps/pluto/releases/latest/download/pluto_linux_amd64.tar.gz | tar xz
./pluto detect-all-in-cluster

# 5. Verify rollback plan
# Note current version
az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query kubernetesVersion

# 6. Test upgrade in dev/staging first
```

---

## Rollback Considerations

> ⚠️ **Important:** Kubernetes does NOT support downgrading a cluster version. Plan carefully.

Mitigation strategies:
1. **Blue-Green clusters** — Run old version in parallel, switch DNS
2. **Thorough staging testing** — Always test in lower environment first
3. **Incremental upgrades** — Never skip more than one minor version
4. **Node image rollback** — Node image upgrades CAN be rolled back

```bash
# If node image upgrade causes issues, re-image the node pool
az aks nodepool upgrade \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name userpool \
  --node-image-only \
  --snapshot-id <previous-snapshot-id>
```

---

## Version Skew Policy

Kubernetes supports a maximum version skew of **±2 minor versions** between components:

```
Control plane: 1.30
Node pools: 1.28, 1.29, or 1.30 (all within ±2 of control plane)
```

Rule: Always upgrade the control plane before node pools.

---

## Summary

| Action | Command |
|--------|---------|
| Check available versions | `az aks get-versions` |
| Check available upgrades | `az aks get-upgrades` |
| Upgrade control plane | `az aks upgrade --control-plane-only` |
| Upgrade node pool | `az aks nodepool upgrade` |
| Node image only | `az aks nodepool upgrade --node-image-only` |
| Auto-upgrade channel | `az aks update --auto-upgrade-channel` |
| Maintenance window | `az aks maintenanceconfiguration add` |

---

## Next Steps

Continue to [02-multi-tenancy.md](./02-multi-tenancy.md) to learn about running multiple teams on a shared cluster.
