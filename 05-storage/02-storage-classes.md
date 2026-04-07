# Storage Classes

**StorageClasses** define how storage is provisioned dynamically in Kubernetes. Each StorageClass specifies a provisioner, parameters, and a reclaim policy. AKS includes several built-in StorageClasses and supports adding custom ones.

---

## AKS Default Storage Classes

AKS ships with these built-in StorageClasses:

```bash
kubectl get storageclass
```

| StorageClass | Provisioner | Disk Type | Use Case |
|-------------|-------------|-----------|----------|
| `default` | disk.csi.azure.com | Standard HDD | Dev/test, cost-sensitive |
| `managed` | disk.csi.azure.com | Standard HDD | Same as default |
| `managed-csi` | disk.csi.azure.com | Standard SSD | General purpose |
| `managed-csi-premium` | disk.csi.azure.com | Premium SSD | Production databases |
| `managed-premium` | disk.csi.azure.com | Premium SSD | Production (legacy) |
| `azurefile` | file.csi.azure.com | Azure Files Standard | Shared storage |
| `azurefile-csi` | file.csi.azure.com | Azure Files Standard | Shared storage (CSI) |
| `azurefile-csi-premium` | file.csi.azure.com | Azure Files Premium | High-throughput shared |

---

## Azure Disk StorageClass

### Built-in Premium SSD

```yaml
# View default managed-csi-premium StorageClass
kubectl get storageclass managed-csi-premium -o yaml
```

### Custom Azure Disk StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-ssd-lrs
provisioner: disk.csi.azure.com
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer    # Wait until pod is scheduled to pick zone
allowVolumeExpansion: true
parameters:
  skuName: Premium_LRS                     # Premium SSD, locally redundant
  kind: managed
  cachingmode: ReadOnly                    # ReadOnly caching improves read performance
  # For disk encryption with customer key:
  # diskEncryptionSetID: /subscriptions/.../diskEncryptionSets/myDES
```

### Azure Disk SKU Options

| SKU | Description | IOPS | Throughput |
|-----|-------------|------|-----------|
| `Standard_LRS` | Standard HDD | Low | Low |
| `StandardSSD_LRS` | Standard SSD | Medium | Medium |
| `Premium_LRS` | Premium SSD | High | High |
| `UltraSSD_LRS` | Ultra Disk | Very High | Very High |
| `Premium_ZRS` | Premium SSD, zone-redundant | High | High |

---

## Azure Files StorageClass

### Use Cases for Azure Files

- Multiple pods need to read/write the same files
- Shared configuration or content across pod replicas
- CMS systems, media storage
- CI/CD artifact storage

### Custom Azure Files StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azure-files-premium
provisioner: file.csi.azure.com
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
parameters:
  skuName: Premium_LRS                     # Premium Files (NFS supported)
  # For NFS protocol (better performance):
  protocol: nfs
  # For SMB (default):
  # protocol: smb
mountOptions:
- dir_mode=0777
- file_mode=0777
- uid=0
- gid=0
- mfsymlinks
- cache=strict
- actimeo=30
```

### Azure Files PVC Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage-pvc
spec:
  accessModes:
  - ReadWriteMany                          # Shared across many pods
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 100Gi
```

---

## volumeBindingMode

| Mode | Behavior | When to Use |
|------|----------|-------------|
| `Immediate` | PV created as soon as PVC is created | Azure Files, single-zone clusters |
| `WaitForFirstConsumer` | PV created when a pod using the PVC is scheduled | Azure Disk (zone-aware), multi-zone clusters |

> 💡 **Tip:** Use `WaitForFirstConsumer` for Azure Disk in multi-zone clusters to ensure the disk is created in the same zone as the pod.

---

## Setting a Default StorageClass

Only one StorageClass can be the default (used when PVC doesn't specify a `storageClassName`).

```bash
# Set a custom StorageClass as default
kubectl patch storageclass my-custom-sc \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default annotation from another class
kubectl patch storageclass managed-csi \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Check which class is default
kubectl get storageclass | grep default
```

---

## Azure Container Storage (Preview)

Azure Container Storage is the next-generation storage platform for AKS, providing local NVMe disk pooling.

```bash
# Enable Azure Container Storage
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-azure-container-storage azureDisk \
  ...

# Create a storage pool
kubectl apply -f - <<EOF
apiVersion: containerstorage.azure.com/v1
kind: StoragePool
metadata:
  name: azuredisk
  namespace: acstor
spec:
  poolType:
    azureDisk: {}
  resources:
    requests:
      storage: 1Ti
EOF
```

---

## Choosing the Right Storage

```
Need persistent storage?
├── YES: Need to share between pods?
│   ├── YES: Use Azure Files (ReadWriteMany)
│   │   ├── High performance required? → Premium_LRS with NFS protocol
│   │   └── Cost-sensitive? → Standard azurefile-csi
│   └── NO: Single pod/node at a time
│       ├── Production database? → Premium SSD (managed-csi-premium)
│       ├── General purpose? → Standard SSD (managed-csi)
│       └── Dev/test? → Standard HDD (default)
└── NO: Temp data only?
    ├── Per-pod temp → emptyDir
    └── Per-node access → hostPath (DaemonSets only)
```

---

## Complete Example: WordPress with MySQL

```yaml
# MySQL PVC — Premium SSD, single node
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi-premium
  resources:
    requests:
      storage: 20Gi
---
# WordPress uploads PVC — Azure Files, shared
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-uploads-pvc
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 50Gi
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
# WordPress Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
spec:
  replicas: 3                 # Multiple replicas — needs RWX storage
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
      - name: wordpress
        image: wordpress:6.4
        env:
        - name: WORDPRESS_DB_HOST
          value: mysql:3306
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: uploads
          mountPath: /var/www/html/wp-content/uploads
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: wordpress-uploads-pvc    # Shared ReadWriteMany
```

---

## Summary

| StorageClass | Type | Access | Best For |
|-------------|------|--------|---------|
| `managed-csi` | Azure Disk SSD | RWO | General purpose |
| `managed-csi-premium` | Azure Disk Premium | RWO | Production DBs |
| `azurefile-csi` | Azure Files | RWX | Shared storage |
| `azurefile-csi-premium` | Azure Files Premium | RWX | High-throughput shared |

---

## Next Steps

Proceed to [06 - Security](../06-security/README.md) to learn about RBAC, managed identity, and secrets management.
