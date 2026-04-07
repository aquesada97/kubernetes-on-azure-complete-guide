# Azure Key Vault CSI Driver

The **Azure Key Vault Provider for Secrets Store CSI Driver** allows you to mount Azure Key Vault secrets, keys, and certificates directly into pods as files or Kubernetes secrets — without hardcoding credentials in your application.

---

## Why Use Key Vault CSI Driver?

| Approach | Problem |
|----------|---------|
| Kubernetes Secrets | Secrets stored in etcd (base64 only), limited access control |
| Environment variables | Secrets visible in pod spec, process list |
| Key Vault CSI Driver | ✅ Secrets from Key Vault, auto-rotated, mounted as files |

---

## Install the CSI Driver

### Via AKS Addon (Recommended)

```bash
# Enable on existing cluster
az aks enable-addons \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --addons azure-keyvault-secrets-provider

# Or enable during cluster creation
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-addons azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m \
  ...
```

```bash
# Verify installation
kubectl get pods -n kube-system -l app=secrets-store-csi-driver
kubectl get pods -n kube-system -l app=csi-secrets-store-provider-azure
```

---

## Create an Azure Key Vault

```bash
# Create Key Vault
az keyvault create \
  --name kv-aks-training \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --enable-rbac-authorization true    # Use RBAC instead of access policies

# Add secrets
az keyvault secret set \
  --vault-name kv-aks-training \
  --name "db-password" \
  --value "SuperSecretPassword123!"

az keyvault secret set \
  --vault-name kv-aks-training \
  --name "api-key" \
  --value "sk-abc123def456ghi789"

# Add a certificate
az keyvault certificate import \
  --vault-name kv-aks-training \
  --name my-cert \
  --file certificate.pfx
```

---

## Configure Access with Workload Identity

Grant the managed identity access to Key Vault:

```bash
# Grant Key Vault Secrets User role
IDENTITY_PRINCIPAL_ID=$(az identity show \
  --name mi-my-app \
  --resource-group $RESOURCE_GROUP \
  --query principalId -o tsv)

KV_SCOPE=$(az keyvault show \
  --name kv-aks-training \
  --resource-group $RESOURCE_GROUP \
  --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $IDENTITY_PRINCIPAL_ID \
  --scope $KV_SCOPE
```

---

## Create a SecretProviderClass

The `SecretProviderClass` defines which Key Vault secrets to retrieve and how to surface them.

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: kv-secrets
  namespace: default
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<IDENTITY_CLIENT_ID>"               # Managed identity client ID
    keyvaultName: "kv-aks-training"                # Key Vault name
    cloudName: ""                                  # Leave blank for Azure Public Cloud
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
          objectVersion: ""                        # Latest version
        - |
          objectName: api-key
          objectType: secret
          objectVersion: ""
        - |
          objectName: my-cert
          objectType: cert
          objectVersion: ""
    tenantId: "<AZURE_TENANT_ID>"                  # Azure Tenant ID
  # Optionally sync to Kubernetes Secrets
  secretObjects:
  - secretName: app-secrets-k8s           # K8s Secret name
    type: Opaque
    data:
    - objectName: db-password
      key: DB_PASSWORD
    - objectName: api-key
      key: API_KEY
```

```bash
# Get your tenant ID
az account show --query tenantId -o tsv

# Apply the SecretProviderClass
kubectl apply -f secretproviderclass.yaml
```

---

## Mount Secrets in a Pod

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
        azure.workload.identity/use: "true"     # Required for Workload Identity
    spec:
      serviceAccountName: my-app-sa
      volumes:
      - name: kv-secrets-vol
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: kv-secrets     # Reference the SecretProviderClass
      containers:
      - name: app
        image: myregistry.azurecr.io/my-app:1.0
        # Option 1: Mount as files
        volumeMounts:
        - name: kv-secrets-vol
          mountPath: "/mnt/secrets"
          readOnly: true
        # Option 2: Use synced Kubernetes Secret as env var
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets-k8s             # From SecretProviderClass.secretObjects
              key: DB_PASSWORD
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: app-secrets-k8s
              key: API_KEY
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
```

The secrets will be available as files:
- `/mnt/secrets/db-password`
- `/mnt/secrets/api-key`
- `/mnt/secrets/my-cert`

```bash
# Verify secrets are mounted
kubectl exec my-app-pod -- ls /mnt/secrets
kubectl exec my-app-pod -- cat /mnt/secrets/db-password

# Check synced K8s Secret
kubectl get secret app-secrets-k8s -o yaml
```

---

## Auto-Rotation of Secrets

When secret rotation is enabled, the CSI driver polls Key Vault for updated secret values and updates the mounted files automatically.

```bash
# Enable rotation on the AKS addon
az aks addon update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --addon azure-keyvault-secrets-provider \
  --enable-secret-rotation \
  --rotation-poll-interval 2m
```

> 📝 **Note:** Auto-rotation updates the mounted files, but if your application reads secrets at startup only, it won't pick up the new value. Use rotation-aware patterns (e.g., read secrets on each request, or use a config reloader).

---

## TLS Certificates from Key Vault

Mount a TLS certificate for NGINX or other services:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: kv-tls-cert
  namespace: ingress-nginx
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    clientID: "<IDENTITY_CLIENT_ID>"
    keyvaultName: "kv-aks-training"
    objects: |
      array:
        - |
          objectName: my-tls-cert
          objectType: secret           # For PFX/PEM cert+key combined
          objectVersion: ""
    tenantId: "<AZURE_TENANT_ID>"
  secretObjects:
  - secretName: my-tls-k8s-secret
    type: kubernetes.io/tls            # TLS secret type
    data:
    - objectName: my-tls-cert
      key: tls.crt
    - objectName: my-tls-cert
      key: tls.key
```

Then reference in Ingress:

```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: my-tls-k8s-secret     # Synced from Key Vault
```

---

## Verify and Troubleshoot

```bash
# Check CSI driver pods
kubectl get pods -n kube-system | grep secrets-store

# Describe SecretProviderClass
kubectl describe secretproviderclass kv-secrets

# Check SecretProviderClassPodStatus (created per pod mount)
kubectl get secretproviderclasspodstatus -n default

# View detailed status
kubectl describe secretproviderclasspodstatus <name>

# Common error: authentication failure
# → Verify managed identity has Key Vault Secrets User role
# → Verify federated credential is configured correctly
# → Verify ServiceAccount annotation has correct client ID

# Check CSI driver logs
kubectl logs -n kube-system -l app=csi-secrets-store-provider-azure --tail=50
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| CSI Driver | Mounts Key Vault secrets into pods as files |
| SecretProviderClass | Defines which secrets to mount, maps to K8s secrets |
| Workload Identity | Authentication method for Key Vault access |
| Secret Objects | Sync KV secrets to K8s Secrets for env var injection |
| Auto-rotation | CSI driver polls for updated values on configurable interval |

---

## Next Steps

Proceed to [07 - Monitoring](../07-monitoring/README.md) to learn about observability in AKS.
