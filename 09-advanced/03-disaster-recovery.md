# Disaster Recovery for AKS

A comprehensive disaster recovery (DR) strategy for AKS addresses both the cluster configuration/state and the application data. This document covers backup, restoration, and multi-region architectures.

---

## RPO and RTO Targets

| Metric | Definition | Target (Example) |
|--------|-----------|-----------------|
| **RPO** (Recovery Point Objective) | Max acceptable data loss | 1 hour |
| **RTO** (Recovery Time Objective) | Max acceptable downtime | 4 hours |

DR strategy complexity increases as RPO/RTO requirements decrease.

---

## What to Back Up

| Component | Tool | Content |
|-----------|------|---------|
| Kubernetes state (manifests) | GitOps (Flux/ArgoCD) | All YAML manifests |
| Kubernetes API objects | Velero | Deployments, Services, etc. |
| Persistent Volumes | Velero + Azure Backup | Application data |
| Container Images | Azure Container Registry geo-replication | Docker images |
| Secrets | Azure Key Vault | Credentials, certificates |
| Cluster configuration | IaC (Bicep/Terraform) | Cluster definition |

---

## Install Velero for Kubernetes Backup

Velero backs up and restores Kubernetes objects and persistent volumes.

### Install Velero CLI

```bash
# Windows
choco install velero

# macOS
brew install velero

# Linux
curl -L https://github.com/vmware-tanzu/velero/releases/latest/download/velero-linux-amd64.tar.gz | tar xz
sudo mv velero /usr/local/bin/
```

### Create Azure Storage for Backups

```bash
BACKUP_RESOURCE_GROUP="rg-aks-backups"
STORAGE_ACCOUNT="aksbackupstorage$(date +%s | cut -c1-8)"
CONTAINER_NAME="velero-backups"
AZURE_BACKUP_SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Create resource group and storage
az group create --name $BACKUP_RESOURCE_GROUP --location $LOCATION

az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $BACKUP_RESOURCE_GROUP \
  --sku Standard_GRS \                    # Geo-redundant storage
  --encryption-services blob \
  --kind BlobStorage \
  --access-tier Hot

az storage container create \
  --name $CONTAINER_NAME \
  --public-access off \
  --account-name $STORAGE_ACCOUNT
```

### Install Velero on AKS

```bash
# Create service principal for Velero
az ad sp create-for-rbac \
  --name velero-sp \
  --role Contributor \
  --scopes /subscriptions/$AZURE_BACKUP_SUBSCRIPTION_ID/resourceGroups/$BACKUP_RESOURCE_GROUP \
           /subscriptions/$AZURE_BACKUP_SUBSCRIPTION_ID/resourceGroups/MC_${RESOURCE_GROUP}_${CLUSTER_NAME}_${LOCATION}

# Install Velero
velero install \
  --provider azure \
  --plugins velero/velero-plugin-for-microsoft-azure:v1.9.0 \
  --bucket $CONTAINER_NAME \
  --secret-file ./credentials-velero \
  --backup-location-config \
    resourceGroup=$BACKUP_RESOURCE_GROUP,\
    storageAccount=$STORAGE_ACCOUNT,\
    storageAccountKeyEnvVar=AZURE_STORAGE_ACCOUNT_ACCESS_KEY,\
    subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID \
  --snapshot-location-config \
    apiTimeout=5m,\
    resourceGroup=$BACKUP_RESOURCE_GROUP,\
    subscriptionId=$AZURE_BACKUP_SUBSCRIPTION_ID \
  --use-node-agent              # For file-level PV backup
```

---

## Creating Backups

### Manual Backup

```bash
# Backup entire cluster
velero backup create full-cluster-backup \
  --include-namespaces '*' \
  --wait

# Backup specific namespace
velero backup create production-backup \
  --include-namespaces production \
  --wait

# Backup with volume data
velero backup create production-with-volumes \
  --include-namespaces production \
  --default-volumes-to-fs-backup \
  --wait

# Check backup status
velero backup describe production-backup
velero backup logs production-backup
```

### Scheduled Backups

```bash
# Daily backup at 2 AM UTC, keep 30 days
velero schedule create daily-production \
  --schedule="0 2 * * *" \
  --include-namespaces production \
  --default-volumes-to-fs-backup \
  --ttl 720h0m0s

# List schedules
velero schedule get

# List backups
velero backup get
```

---

## Restoring from Backup

```bash
# List available backups
velero backup get

# Restore from backup
velero restore create --from-backup production-backup

# Restore to a different namespace
velero restore create \
  --from-backup production-backup \
  --namespace-mappings production:production-restored

# Check restore status
velero restore get
velero restore describe production-backup-<timestamp>
```

---

## Azure Backup for AKS (Native)

Azure Backup now supports native AKS backup (GA in some regions):

```bash
# Enable Azure Backup for AKS
az aks trustedaccess rolebinding create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name backup-rolebinding \
  --source-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.DataProtection/backupVaults/<vault> \
  --roles Microsoft.DataProtection/backupVaults/backup-operator

# Create Backup Vault
az dataprotection backup-vault create \
  --vault-name aks-backup-vault \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION \
  --type SystemAssigned \
  --storage-setting '[{"datastore-type":"VaultStore","type":"LocallyRedundant"}]'
```

---

## GitOps for Cluster State Backup

Store all cluster configuration in Git. This is the most important DR strategy — if you lose the cluster, you can recreate everything from Git.

### Install Flux

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux on cluster
flux bootstrap github \
  --owner=your-org \
  --repository=cluster-config \
  --branch=main \
  --path=./clusters/production \
  --personal

# After bootstrap, Flux continuously syncs Git → cluster
```

### Directory Structure

```
cluster-config/
├── clusters/
│   ├── production/
│   │   ├── flux-system/          # Flux components
│   │   └── kustomizations/       # What to deploy
│   └── staging/
├── apps/
│   ├── web-app/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── ingress.yaml
│   └── api/
└── infrastructure/
    ├── ingress-nginx/
    ├── cert-manager/
    └── monitoring/
```

---

## Multi-Region Active-Passive Architecture

For mission-critical workloads:

```
Region 1 (Primary - Active)
  ├── AKS Cluster + Applications
  ├── Azure SQL / Cosmos DB (Primary)
  ├── Azure Files / Blob (Primary)
  └── Azure Traffic Manager (Weighted: 100%)
                    ↕ (Replication)
Region 2 (Secondary - Passive)
  ├── AKS Cluster (stopped or minimal)
  ├── Azure SQL / Cosmos DB (Read Replica)
  ├── Azure Files / Blob (GRS/GZRS secondary)
  └── Azure Traffic Manager (Weighted: 0%)
```

### Failover Steps

```bash
# 1. Scale up secondary cluster
az aks scale \
  --name $CLUSTER_NAME_DR \
  --resource-group $RESOURCE_GROUP_DR \
  --node-count 3

# 2. Connect to secondary cluster
az aks get-credentials \
  --name $CLUSTER_NAME_DR \
  --resource-group $RESOURCE_GROUP_DR

# 3. Apply latest manifests from Git
flux reconcile kustomization cluster-apps --with-source

# 4. Update Traffic Manager to route to secondary
az network traffic-manager endpoint update \
  --profile-name my-traffic-manager \
  --resource-group $RESOURCE_GROUP \
  --name primary-endpoint \
  --type azureEndpoints \
  --weight 0

az network traffic-manager endpoint update \
  --profile-name my-traffic-manager \
  --resource-group $RESOURCE_GROUP \
  --name secondary-endpoint \
  --type azureEndpoints \
  --weight 100
```

---

## DR Runbook Template

```markdown
## AKS Disaster Recovery Runbook

### Scenario: Complete Region Failure

#### Pre-conditions
- [ ] Velero backups are current (< RPO hours old)
- [ ] GitOps repository is up to date
- [ ] DR cluster exists in secondary region

#### Steps

1. **Declare incident** — notify team, open incident ticket
2. **Assess** — determine scope (cluster? region? data?)
3. **Activate DR cluster** — `az aks scale --node-count 3`
4. **Redirect traffic** — update Traffic Manager or DNS
5. **Restore data** — `velero restore create --from-backup <latest>`
6. **Verify** — run smoke tests against DR environment
7. **Notify stakeholders** — update status page

#### Estimated Time
- Traffic redirection: 5 minutes
- Cluster scaling: 5-10 minutes
- Data restore: 15-60 minutes (depends on data volume)
- Smoke tests: 15 minutes
- **Total RTO: ~60-90 minutes**
```

---

## Summary

| DR Component | Tool/Strategy |
|-------------|--------------|
| Cluster configuration | GitOps (Flux/ArgoCD) + IaC |
| K8s objects (non-persistent) | Velero object backup |
| Persistent Volume data | Velero + Azure Backup |
| Container images | ACR geo-replication |
| Secrets | Azure Key Vault (multi-region) |
| Traffic routing | Azure Traffic Manager / Front Door |
| Multi-region | Active-Passive or Active-Active |

---

## Next Steps

Proceed to the [Labs](../labs/README.md) to apply all these concepts with hands-on exercises.
