# Create an AKS Cluster

This document covers multiple methods to create an AKS cluster, with detailed explanations of each configuration option.

---

## Method 1: Azure CLI (Recommended)

### Prerequisites

```bash
az account show

RESOURCE_GROUP="rg-aks-training"
CLUSTER_NAME="aks-training-cluster"
LOCATION="eastus"
```

### Basic Cluster Creation

```bash
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --kubernetes-version 1.29 \
  --node-count 3 \
  --node-vm-size Standard_D2s_v3 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --tier standard
```

### Flag-by-Flag Explanation

| Flag | Value | Explanation |
|------|-------|-------------|
| `--name` | `$CLUSTER_NAME` | Cluster name. Must be unique in the resource group. |
| `--resource-group` | `$RESOURCE_GROUP` | Resource group where the AKS resource is created. |
| `--location` | `$LOCATION` | Azure region. Defaults to resource group location if omitted. |
| `--kubernetes-version` | `1.29` | Kubernetes version. Use `az aks get-versions` to list available. |
| `--node-count` | `3` | Initial node count for the default system pool. |
| `--node-vm-size` | `Standard_D2s_v3` | VM size for nodes. Standard_D2s_v3 = 2 vCPU, 8 GB RAM. |
| `--network-plugin` | `azure` | Use Azure CNI (vs `kubenet`). |
| `--network-plugin-mode` | `overlay` | Azure CNI Overlay — pods get IPs from private CIDR. |
| `--pod-cidr` | `192.168.0.0/16` | Private CIDR for pod IPs (only used with overlay mode). |
| `--enable-managed-identity` | (flag) | Use system-assigned managed identity instead of service principal. |
| `--enable-addons monitoring` | `monitoring` | Install Azure Monitor / Container Insights addon. |
| `--generate-ssh-keys` | (flag) | Auto-generate SSH keys for node access. |
| `--tier` | `standard` | SLA tier: `free` (no SLA) or `standard` (99.9% SLA). |

### Production-Ready Cluster

```bash
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --kubernetes-version 1.29 \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --generate-ssh-keys \
  --tier standard \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  --nodepool-name systempool \
  --os-sku AzureLinux \
  --auto-upgrade-channel patch \
  --node-os-upgrade-channel NodeImage \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-azure-rbac \
  --aad-admin-group-object-ids <GROUP-ID>
```

### Viewing Available Kubernetes Versions

```bash
az aks get-versions \
  --location eastus \
  --output table
```

### Listing Available VM Sizes

```bash
az vm list-sizes \
  --location eastus \
  --query "[?contains(name, 'Standard_D')]" \
  --output table
```

---

## Method 2: Azure Portal

1. Navigate to [portal.azure.com](https://portal.azure.com)
2. Search for **Kubernetes services** → **Create** → **Kubernetes cluster**
3. Fill in the **Basics** tab:
   - Subscription, Resource Group, Region
   - Cluster name, Kubernetes version, Tier
4. Configure **Node pools**:
   - System pool: VM size, node count, availability zones
   - Add user node pools as needed
5. **Networking** tab: Choose Azure CNI or Kubenet
6. **Integrations** tab: Enable Container Insights, ACR
7. Review + Create

The portal provides a guided experience and is good for initial exploration, but the CLI/IaC approach is preferred for reproducible environments.

---

## Method 3: Bicep

```bicep
// aks-cluster.bicep
param clusterName string = 'aks-training-cluster'
param location string = resourceGroup().location
param nodeCount int = 3
param nodeVmSize string = 'Standard_D2s_v3'
param kubernetesVersion string = '1.29'

resource logAnalyticsWorkspace 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: '${clusterName}-law'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

resource aks 'Microsoft.ContainerService/managedClusters@2023-07-01' = {
  name: clusterName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: clusterName
    agentPoolProfiles: [
      {
        name: 'systempool'
        count: nodeCount
        vmSize: nodeVmSize
        mode: 'System'
        osSKU: 'AzureLinux'
        availabilityZones: ['1', '2', '3']
      }
    ]
    networkProfile: {
      networkPlugin: 'azure'
      networkPluginMode: 'overlay'
      podCidr: '192.168.0.0/16'
    }
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalyticsWorkspace.id
        }
      }
    }
  }
}

output clusterName string = aks.name
output clusterFqdn string = aks.properties.fqdn
```

```bash
az deployment group create \
  --resource-group rg-aks-training \
  --template-file aks-cluster.bicep
```

---

## Method 4: Terraform

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.80"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "aks" {
  name     = "rg-aks-training"
  location = "East US"
}

resource "azurerm_kubernetes_cluster" "aks" {
  name                = "aks-training-cluster"
  location            = azurerm_resource_group.aks.location
  resource_group_name = azurerm_resource_group.aks.name
  dns_prefix          = "akstraining"
  kubernetes_version  = "1.29"

  default_node_pool {
    name       = "systempool"
    node_count = 3
    vm_size    = "Standard_D2s_v3"
  }

  identity {
    type = "SystemAssigned"
  }

  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"
    pod_cidr            = "192.168.0.0/16"
  }

  tags = {
    Environment = "Training"
  }
}

output "cluster_name" {
  value = azurerm_kubernetes_cluster.aks.name
}

output "kube_config" {
  value     = azurerm_kubernetes_cluster.aks.kube_config_raw
  sensitive = true
}
```

```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

---

## Verify Cluster Creation

```bash
az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table

az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query kubernetesVersion

az aks nodepool list \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --output table
```

---

## Next Steps

Continue to [02-kubectl-basics.md](./02-kubectl-basics.md) to learn how to interact with your cluster.
