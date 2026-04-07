# Lab 03: Persistent Storage

In this lab, you will work with Kubernetes persistent storage: creating PersistentVolumeClaims backed by Azure Disk and Azure Files, and mounting them in pods.

**Estimated Time:** 40 minutes  
**Difficulty:** Intermediate

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

- Create PersistentVolumeClaims using different StorageClasses
- Mount Azure Disk storage in a single-pod deployment
- Mount Azure Files storage shared across multiple pods
- Verify data persists across pod restarts
- Understand storage expansion

---

## Prerequisites

- Completed [Lab 02](../lab-02-ingress/README.md)
- AKS cluster running

---

## Lab Setup

```bash
kubectl create namespace lab03
kubectl config set-context --current --namespace=lab03
```

---

## Part 1: Explore Storage Classes

```bash
# List available storage classes
kubectl get storageclass

# Describe the default storage class
kubectl describe storageclass managed-csi

# See which is default (marked with "(default)")
kubectl get storageclass | grep default
```

---

## Part 2: Azure Disk (ReadWriteOnce)

### 2.1 Create the PVC

```bash
kubectl apply -f manifests/pvc.yaml
kubectl get pvc -n lab03 --watch
# Wait for STATUS to change from Pending to Bound
```

> 💡 **Note:** Azure Disk PVCs may stay in `Pending` state if using `WaitForFirstConsumer` binding mode. This is normal — the PV is created when a pod using it is scheduled.

### 2.2 Deploy a Pod with the PVC

```bash
kubectl apply -f manifests/pod-with-pvc.yaml
kubectl get pods -n lab03 --watch
```

### 2.3 Write Data to the Volume

```bash
POD_NAME=$(kubectl get pods -n lab03 -l app=storage-test -o jsonpath='{.items[0].metadata.name}')

# Write some data
kubectl exec -it $POD_NAME -- sh -c "echo 'Hello from AKS Storage Lab!' > /data/lab03.txt"
kubectl exec -it $POD_NAME -- cat /data/lab03.txt

# Check disk usage
kubectl exec -it $POD_NAME -- df -h /data
```

### 2.4 Test Persistence

```bash
# Delete the pod
kubectl delete pod $POD_NAME

# Wait for new pod to be created by the Deployment
kubectl get pods --watch

# Check the data is still there
NEW_POD=$(kubectl get pods -n lab03 -l app=storage-test -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $NEW_POD -- cat /data/lab03.txt
# Data should still be present!
```

---

## Part 3: Azure Files (ReadWriteMany)

### 3.1 Create the Azure Files PVC

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefiles-pvc
  namespace: lab03
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: azurefile-csi
  resources:
    requests:
      storage: 5Gi
EOF

kubectl get pvc azurefiles-pvc --watch
```

### 3.2 Deploy Multiple Pods Sharing Storage

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-storage-app
  namespace: lab03
spec:
  replicas: 3
  selector:
    matchLabels:
      app: shared-storage
  template:
    metadata:
      labels:
        app: shared-storage
    spec:
      containers:
      - name: app
        image: nginx:1.25
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html/shared
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: azurefiles-pvc
EOF

kubectl get pods -l app=shared-storage --watch
```

### 3.3 Write From One Pod, Read From Another

```bash
# Get pod names
PODS=($(kubectl get pods -l app=shared-storage -o jsonpath='{.items[*].metadata.name}'))

# Write from first pod
kubectl exec -it ${PODS[0]} -- sh -c "echo 'Written from ${PODS[0]}' > /usr/share/nginx/html/shared/hello.txt"

# Read from second pod (different pod, same data!)
kubectl exec -it ${PODS[1]} -- cat /usr/share/nginx/html/shared/hello.txt

# Read from third pod
kubectl exec -it ${PODS[2]} -- cat /usr/share/nginx/html/shared/hello.txt
```

---

## Part 4: Storage Inspection

```bash
# View PVCs
kubectl get pvc -n lab03

# View PVs (cluster-level resources)
kubectl get pv

# Describe PVC (see which PV it's bound to)
kubectl describe pvc azure-disk-pvc -n lab03

# View the underlying Azure Disk
DISK_NAME=$(kubectl get pv $(kubectl get pvc azure-disk-pvc -n lab03 -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}' | awk -F/ '{print $NF}')
echo "Azure Disk name: $DISK_NAME"
```

---

## Part 5: Expand a PVC

```bash
# Check current size
kubectl get pvc azure-disk-pvc -n lab03

# Expand (patch the storage request)
kubectl patch pvc azure-disk-pvc -n lab03 \
  -p '{"spec":{"resources":{"requests":{"storage":"15Gi"}}}}'

# Watch expansion
kubectl describe pvc azure-disk-pvc -n lab03 | grep -A 5 Conditions

# May need to restart pod for filesystem resize to take effect
kubectl delete pod -l app=storage-test -n lab03
sleep 30
kubectl exec -it $(kubectl get pods -l app=storage-test -n lab03 -o jsonpath='{.items[0].metadata.name}') -- df -h /data
```

---

## Cleanup

```bash
kubectl delete namespace lab03
```

---

## ✅ Lab Completion Checklist

- [ ] Listed available StorageClasses
- [ ] Created Azure Disk PVC (ReadWriteOnce)
- [ ] Wrote data to the volume and confirmed it persists after pod restart
- [ ] Created Azure Files PVC (ReadWriteMany)
- [ ] Verified shared access — wrote from one pod, read from another
- [ ] Expanded a PVC

---

## 🤔 Discussion Questions

1. Why can't you use ReadWriteOnce with 3 replicas on different nodes?
2. What happens to the Azure Disk when you delete the PVC?
3. When would you choose NFS protocol for Azure Files over SMB?

---

## Next Lab

Proceed to [Lab 04: Autoscaling](../lab-04-scaling/README.md).
