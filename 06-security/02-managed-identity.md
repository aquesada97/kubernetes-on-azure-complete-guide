# Azure Managed Identity and Workload Identity

**Azure Workload Identity** allows pods to authenticate to Azure services (Key Vault, Storage, SQL, etc.) using Azure AD without storing credentials anywhere. It replaces the older pod-managed identity (AAD Pod Identity) approach.

---

## Why Workload Identity?

Without Workload Identity, applications access Azure services using:
- Connection strings with secrets → stored in ConfigMaps/Secrets → security risk
- Service principal credentials → need rotation, can be leaked

With Workload Identity:
- No credentials stored anywhere
- Pods use Kubernetes ServiceAccount tokens to exchange for Azure AD tokens
- Tokens are short-lived and automatically rotated
- Follows OAuth 2.0 federated credential flow

---

## How It Works

```
1. Pod uses ServiceAccount with Workload Identity annotation
2. Kubernetes issues ServiceAccount OIDC token
3. Azure SDK in pod exchanges token with AAD using federated credential
4. AAD issues an Azure access token
5. Pod accesses Azure resource using the Azure token
```

---

## Prerequisites

Workload Identity requires:
1. AKS cluster with OIDC Issuer enabled
2. AKS cluster with Workload Identity addon enabled
3. Azure User-Assigned Managed Identity
4. Federated credential on the managed identity
5. Annotated Kubernetes ServiceAccount

---

## Step 1: Enable on AKS Cluster

```bash
# New cluster
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-oidc-issuer \
  --enable-workload-identity \
  ...

# Existing cluster
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-oidc-issuer \
  --enable-workload-identity

# Get the OIDC issuer URL (needed for federated credential)
OIDC_ISSUER=$(az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

echo $OIDC_ISSUER
# https://eastus.oic.prod-aks.azure.com/<tenant-id>/<cluster-id>/
```

---

## Step 2: Create User-Assigned Managed Identity

```bash
# Create the managed identity
az identity create \
  --name mi-my-app \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Get identity details
IDENTITY_CLIENT_ID=$(az identity show \
  --name mi-my-app \
  --resource-group $RESOURCE_GROUP \
  --query clientId -o tsv)

IDENTITY_PRINCIPAL_ID=$(az identity show \
  --name mi-my-app \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

echo "Client ID: $IDENTITY_CLIENT_ID"
echo "Principal ID: $IDENTITY_PRINCIPAL_ID"
```

---

## Step 3: Assign Azure Role to the Identity

Grant the managed identity permissions to the Azure resource it needs to access.

```bash
# Example: Grant Key Vault Secrets User role
KEY_VAULT_ID=$(az keyvault show --name my-keyvault --resource-group $RESOURCE_GROUP --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $KEY_VAULT_ID

# Example: Grant Storage Blob Data Reader role
STORAGE_ID=$(az storage account show --name mystorageaccount --resource-group $RESOURCE_GROUP --query id -o tsv)

az role assignment create \
  --role "Storage Blob Data Reader" \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $STORAGE_ID
```

---

## Step 4: Create Federated Credential

Link the managed identity to the Kubernetes ServiceAccount:

```bash
# Variables
NAMESPACE="default"
SERVICE_ACCOUNT_NAME="my-app-sa"

# Create federated credential
az identity federated-credential create \
  --name "fc-my-app" \
  --identity-name "mi-my-app" \
  --resource-group $RESOURCE_GROUP \
  --issuer $OIDC_ISSUER \
  --subject "system:serviceaccount:${NAMESPACE}:${SERVICE_ACCOUNT_NAME}" \
  --audiences api://AzureADTokenExchange
```

---

## Step 5: Create Kubernetes ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: "<IDENTITY_CLIENT_ID>"    # From Step 2
  labels:
    azure.workload.identity/use: "true"
```

```bash
kubectl apply -f serviceaccount.yaml
```

---

## Step 6: Deploy Application Using Workload Identity

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
        azure.workload.identity/use: "true"      # Required label
    spec:
      serviceAccountName: my-app-sa              # ServiceAccount from Step 5
      containers:
      - name: app
        image: myregistry.azurecr.io/my-app:1.0
        env:
        - name: AZURE_CLIENT_ID
          value: "<IDENTITY_CLIENT_ID>"
        - name: KEYVAULT_URL
          value: "https://my-keyvault.vault.azure.net/"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

---

## Application Code: Using Workload Identity

The Azure SDK automatically handles token exchange when Workload Identity is configured.

### Python

```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient

credential = DefaultAzureCredential()
client = SecretClient(vault_url="https://my-keyvault.vault.azure.net/", credential=credential)

secret = client.get_secret("my-secret")
print(secret.value)
```

### Go

```go
import (
    "github.com/Azure/azure-sdk-for-go/sdk/azidentity"
    "github.com/Azure/azure-sdk-for-go/sdk/keyvault/azsecrets"
)

cred, _ := azidentity.NewDefaultAzureCredential(nil)
client, _ := azsecrets.NewClient("https://my-keyvault.vault.azure.net/", cred, nil)
secret, _ := client.GetSecret(ctx, "my-secret", "", nil)
```

### .NET

```csharp
var credential = new DefaultAzureCredential();
var client = new SecretClient(new Uri("https://my-keyvault.vault.azure.net/"), credential);

var secret = await client.GetSecretAsync("my-secret");
Console.WriteLine(secret.Value.Value);
```

---

## Verifying Workload Identity

```bash
# Check the Workload Identity webhook is running
kubectl get pods -n kube-system -l app=azure-wi-webhook-controller-manager

# Check pod environment for injected env vars
kubectl exec my-app-pod -- env | grep AZURE

# Expected:
# AZURE_CLIENT_ID=<client-id>
# AZURE_TENANT_ID=<tenant-id>
# AZURE_FEDERATED_TOKEN_FILE=/var/run/secrets/azure/tokens/azure-identity-token

# Verify the projected token file exists
kubectl exec my-app-pod -- ls /var/run/secrets/azure/tokens/
```

---

## System-Assigned vs User-Assigned Managed Identity

| Feature | System-Assigned | User-Assigned |
|---------|----------------|---------------|
| Tied to | AKS cluster | Standalone resource |
| Lifecycle | Deleted with cluster | Independent |
| Sharing | Cannot be shared | Can be shared across resources |
| Used for Workload Identity | No | Yes |
| Used for cluster operations | Yes (ACR, LB, etc.) | Yes |

The **AKS cluster's system-assigned identity** is used for:
- Creating Azure Load Balancers
- Creating managed disks
- Attaching ACR (with `--attach-acr`)
- Managing node resource group

---

## Summary

| Step | Action |
|------|--------|
| 1 | Enable OIDC Issuer + Workload Identity on cluster |
| 2 | Create User-Assigned Managed Identity |
| 3 | Grant Azure RBAC role to identity |
| 4 | Create federated credential linking SA to identity |
| 5 | Create annotated Kubernetes ServiceAccount |
| 6 | Deploy pod with SA and label |
| 7 | Use DefaultAzureCredential in app code |

---

## Next Steps

Continue to [03-azure-keyvault-csi.md](./03-azure-keyvault-csi.md) to learn how to mount Azure Key Vault secrets directly into pods as files or environment variables.
