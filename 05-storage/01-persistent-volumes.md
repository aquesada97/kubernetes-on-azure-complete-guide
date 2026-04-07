# Persistent Volumes

Kubernetes provides a storage abstraction layer through **PersistentVolumes (PV)** and **PersistentVolumeClaims (PVC)** that decouples storage provisioning from storage consumption.

---

## The PV/PVC Relationship

```
Administrator provisions storage → PersistentVolume (PV)
Developer requests storage       → PersistentVolumeClaim (PVC)
Kubernetes binds PVC to PV       → Pod mounts PVC
```

- **PersistentVolume (PV)**: A piece of storage provisioned by an admin or dynamically by a StorageClass. It exists independently of any Pod.
- **PersistentVolumeClaim (PVC)**: A request for storage by a user. Specifies size and access mode. Kubernetes binds PVCs to matching PVs.

---

## Access Modes

| Mode | Short | Description | Supported By |
|------|-------|-------------|-------------|
| ReadWriteOnce | RWO | Read/write by a single node | Azure Disk, Azure Files |
| ReadOnlyMany | ROX | Read-only by many nodes | Azure Files, NFS |
| ReadWriteMany | RWX | Read/write by many nodes | Azure Files |
| ReadWriteOncePod | RWOP | Read/write by a single pod | Azure Disk (CSI) |

> 📝 **Note:** Azure Disk only supports `ReadWriteOnce` (one node at a time). For multi-pod shared storage, use Azure Files.

---

## Volume Lifecycle

```
Provisioning → Binding → Using → Releasing → Reclaiming
```

1. **Provisioning**: PV is created (static) or StorageClass creates it (dynamic)
2. **Binding**: PVC finds a matching PV and binds to it (1-to-1 mapping)
3. **Using**: Pod mounts the PVC as a volume
4. **Releasing**: Pod is deleted, PVC remains (unless deleted)
5. **Reclaiming**: Based on ReclaimPolicy (Retain, Delete, Recycle)

### Reclaim Policies

| Policy | Behavior after PVC deleted |
|--------|---------------------------|
| `Retain` | PV and data preserved. Manual cleanup required. |
| `Delete` | PV and underlying storage deleted automatically. |
| `Recycle` | Data scrubbed, PV available for reuse. Deprecated. |

---

## Dynamic Provisioning (Recommended)

With dynamic provisioning, PVs are automatically created when a PVC is submitted.

```yaml
# PVC requests storage from a StorageClass
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-csi         # AKS default StorageClass
  resources:
    requests:
      storage: 10Gi
```

```bash
kubectl apply -f pvc.yaml
kubectl get pvc my-pvc

# NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc    Bound    pvc-abc12345-...                           10Gi       RWO            managed-csi    30s
```

---

## Static Provisioning

Manually create a PV and then claim it with a PVC:

```yaml
# PersistentVolume (admin creates this)
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-static-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""                   # Empty for static binding
  csi:
    driver: disk.csi.azure.com
    readOnly: false
    volumeHandle: /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.Compute/disks/myDisk
    volumeAttributes:
      fsType: ext4
---
# PVC binds to the specific PV
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-static-pvc
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 50Gi
  volumeName: my-static-pv              # Bind to specific PV
```

---

## Using PVCs in Pods

### Single Pod with PVC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx:1.25
    ports:
    - containerPort: 80
    volumeMounts:
    - name: data-volume
      mountPath: /usr/share/nginx/html   # Mount point in container
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "250m"
        memory: "256Mi"
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: my-pvc                  # Reference the PVC
```

### Deployment with PVC (ReadWriteOnce)

> ⚠️ **Note:** For RWO volumes, the deployment must have `replicas: 1`, or all pods must run on the same node (which is not guaranteed). For multi-replica deployments, use `ReadWriteMany` (Azure Files).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: single-replica-db
spec:
  replicas: 1                            # Must be 1 for RWO
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          value: mypassword
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
```

### Multiple Pods Sharing Storage (ReadWriteMany)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-with-shared-storage
spec:
  replicas: 3                            # All replicas can mount same volume
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: web
        image: nginx:1.25
        volumeMounts:
        - name: shared-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-content
        persistentVolumeClaim:
          claimName: shared-azurefiles-pvc   # Must be ReadWriteMany
```

---

## Ephemeral Storage (emptyDir)

`emptyDir` volumes are created when a Pod starts and deleted when it terminates. Useful for:
- Cache storage
- Scratch space
- Sharing files between containers in a Pod

```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    volumeMounts:
    - name: cache
      mountPath: /tmp/cache
    - name: shared-data
      mountPath: /shared
  - name: sidecar
    image: my-sidecar:1.0
    volumeMounts:
    - name: shared-data
      mountPath: /input
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 500Mi           # Optional size limit
  - name: shared-data
    emptyDir: {}
```

---

## Expanding PVCs

Resize a PVC after creation (requires StorageClass with `allowVolumeExpansion: true`):

```bash
# Edit the PVC storage request
kubectl edit pvc my-pvc

# Or patch it
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Check expansion status
kubectl describe pvc my-pvc | grep -A 5 Conditions
```

---

## Useful PV/PVC Commands

```bash
# List PVCs
kubectl get pvc
kubectl get pvc -n my-namespace

# List PVs
kubectl get pv

# Describe PVC
kubectl describe pvc my-pvc

# Check PVC events (useful for debugging provisioning failures)
kubectl describe pvc my-pvc | grep -A 10 Events

# Delete PVC (careful — may delete data depending on reclaim policy)
kubectl delete pvc my-pvc

# List storage classes
kubectl get storageclass
kubectl get sc
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| PV | Admin-provisioned or dynamically created storage |
| PVC | User's request for storage; bound 1-to-1 to a PV |
| Dynamic provisioning | StorageClass creates PV automatically on PVC creation |
| Access modes | RWO (disk), RWX (files), ROX (files read-only) |
| Reclaim policy | Retain (keep data) vs Delete (destroy on PVC delete) |
| emptyDir | Ephemeral, pod-scoped, deleted when pod terminates |

---

## Next Steps

Continue to [02-storage-classes.md](./02-storage-classes.md) to learn about AKS-specific StorageClasses.
