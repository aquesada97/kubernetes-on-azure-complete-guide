# 00 - Prerequisites

Before starting the AKS Training modules, you need to set up your local environment and Azure account. This document walks you through every required tool and configuration step.

---

## 🎯 Learning Objectives

After completing this section, you will be able to:

- Install and configure all required CLI tools
- Authenticate to Azure using the Azure CLI
- Create a resource group for training exercises
- Set environment variables used throughout the training

---

## Required Tools

| Tool | Minimum Version | Purpose |
|------|----------------|---------|
| Azure CLI | 2.50+ | Manage Azure resources and AKS |
| kubectl | 1.27+ | Interact with Kubernetes clusters |
| Helm | 3.12+ | Install Kubernetes applications |
| VS Code | Latest | Code editor with Kubernetes support |
| kubelogin | Latest | Azure AD authentication for kubectl |
| Git | 2.40+ | Clone this repository |

---

## 1. Install Azure CLI

### Windows

```powershell
# Using winget (recommended)
winget install Microsoft.AzureCLI

# Or download the MSI installer
# https://aka.ms/installazurecliwindows
```

### macOS

```bash
brew update && brew install azure-cli
```

### Linux (Ubuntu/Debian)

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Verify installation:

```bash
az version
```

Expected output should show version 2.50.0 or higher.

---

## 2. Install kubectl

### Windows

```powershell
# Using Azure CLI (easiest method)
az aks install-cli

# Or using winget
winget install Kubernetes.kubectl

# Or using Chocolatey
choco install kubernetes-cli
```

### macOS

```bash
brew install kubectl
```

### Linux

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Verify installation:

```bash
kubectl version --client
```

---

## 3. Install Helm

### Windows

```powershell
# Using winget
winget install Helm.Helm

# Or using Chocolatey
choco install kubernetes-helm
```

### macOS

```bash
brew install helm
```

### Linux

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

Verify installation:

```bash
helm version
```

---

## 4. Install kubelogin

kubelogin is required for Azure AD (Entra ID) authentication with AKS clusters.

### Windows

```powershell
# Using Azure CLI (installs automatically with az aks install-cli)
az aks install-cli

# Or using winget
winget install Microsoft.Azure.Kubelogin
```

### macOS

```bash
brew install Azure/kubelogin/kubelogin
```

### Linux

```bash
curl -LO https://github.com/Azure/kubelogin/releases/latest/download/kubelogin-linux-amd64.zip
unzip kubelogin-linux-amd64.zip
sudo mv bin/linux_amd64/kubelogin /usr/local/bin/
```

Verify:

```bash
kubelogin --version
```

---

## 5. Install VS Code and Extensions

Download VS Code from [https://code.visualstudio.com](https://code.visualstudio.com).

Install the following extensions:

```bash
code --install-extension ms-kubernetes-tools.vscode-kubernetes-tools
code --install-extension ms-azuretools.vscode-docker
code --install-extension ms-azure-devops.azure-pipelines
code --install-extension hashicorp.terraform
code --install-extension redhat.vscode-yaml
```

Recommended extensions summary:

| Extension | Publisher | Purpose |
|-----------|-----------|---------|
| Kubernetes | Microsoft | Browse clusters, edit manifests |
| Docker | Microsoft | Dockerfile and compose support |
| YAML | Red Hat | YAML validation and completion |
| Azure Tools | Microsoft | Azure resource management |
| Terraform | HashiCorp | Infrastructure as code |

---

## 6. Azure Subscription Setup

### Login to Azure

```bash
az login
```

This opens a browser for authentication. After login, list your subscriptions:

```bash
az account list --output table
```

Set the subscription you want to use for this training:

```bash
az account set --subscription "YOUR-SUBSCRIPTION-ID-OR-NAME"
```

Confirm the active subscription:

```bash
az account show --output table
```

### Register Required Resource Providers

```bash
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.KeyVault

# Check registration status (wait until all show "Registered")
az provider show --namespace Microsoft.ContainerService --query registrationState
```

---

## 7. Create Resource Group

All training resources will be created in a single resource group for easy cleanup.

```bash
RESOURCE_GROUP="rg-aks-training"
LOCATION="eastus"

az group create \
  --name $RESOURCE_GROUP \
  --location $LOCATION \
  --tags purpose=training owner=yourname
```

> 💡 **Tip:** Use a location close to you for better latency. Run `az account list-locations --output table` to see all available regions.

---

## 8. Environment Variables

Set these environment variables in your shell. Add them to `~/.bashrc`, `~/.zshrc`, or Windows Environment Variables for persistence.

### Bash / Zsh

```bash
export RESOURCE_GROUP="rg-aks-training"
export LOCATION="eastus"
export CLUSTER_NAME="aks-training-cluster"
export NODE_COUNT=3
export NODE_VM_SIZE="Standard_D2s_v3"
export KUBERNETES_VERSION="1.29"
export ACR_NAME="acrtraining$(az account show --query id -o tsv | cut -c1-8)"
```

### PowerShell

```powershell
$env:RESOURCE_GROUP = "rg-aks-training"
$env:LOCATION = "eastus"
$env:CLUSTER_NAME = "aks-training-cluster"
$env:NODE_COUNT = 3
$env:NODE_VM_SIZE = "Standard_D2s_v3"
$env:KUBERNETES_VERSION = "1.29"
```

---

## 9. Verify Everything

Run this checklist to confirm your environment is ready:

```bash
az version --output table
az account show --query "[name, id]" --output table
kubectl version --client --output=yaml | grep gitVersion
helm version --short
kubelogin --version
```

Expected results: all tools installed, logged in to Azure, resource group created.

---

## 🧹 Cleanup (After Training)

When you are done with all training modules, delete the resource group to avoid charges:

```bash
az group delete --name rg-aks-training --yes --no-wait
```

> ⚠️ **Warning:** This permanently deletes all resources in the resource group.

---

## Next Steps

Proceed to [01 - Fundamentals](../01-fundamentals/README.md) to learn Kubernetes and AKS architecture concepts.
