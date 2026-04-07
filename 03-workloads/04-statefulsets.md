# StatefulSets

A **StatefulSet** is a workload resource for managing stateful applications that require stable identities, ordered deployment, and persistent storage. Unlike Deployments, StatefulSet Pods have predictable names and stable network identities.

---

## When to Use StatefulSet vs Deployment

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod naming | Random (e.g., `app-7f9d8-xkz`) | Ordered (e.g., `app-0`, `app-1`) |
| Pod identity | Interchangeable | Unique, stable |
| Startup/shutdown order | Parallel (by default) | Ordered (0, 1, 2, ...) |
| Persistent storage | Shared or none | Individual PVC per Pod |
| DNS | Service only | Stable per-pod DNS |
| Use cases | Stateless web apps, APIs | Databases, message queues, caches |

**Use StatefulSet for:**
- Databases: PostgreSQL, MySQL, MongoDB
- Message queues: Kafka, RabbitMQ
- Distributed caches: Redis Cluster, Cassandra
- Any app requiring stable network identity or ordered operations

---

## Core Concepts

### Stable Pod Identity

StatefulSet Pods are named with an ordinal index: `<statefulset-name>-<ordinal>`.

For a StatefulSet named `postgres` with 3 replicas:
- `postgres-0` — primary (first, always)
- `postgres-1` — replica
- `postgres-2` — replica

Pods maintain their identity across restarts and rescheduling.

### Headless Service

StatefulSets require a **headless Service** (clusterIP: None) to provide stable DNS addresses per Pod.

DNS pattern: `<pod-name>.<headless-service>.<namespace>.svc.cluster.local`

Examples:
- `postgres-0.postgres-headless.default.svc.cluster.local`
- `postgres-1.postgres-headless.default.svc.cluster.local`

### Ordered Operations

By default:
- **Scale up**: Pods created in order (0 → 1 → 2). Next pod not started until previous is Running and Ready.
- **Scale down**: Pods deleted in reverse order (2 → 1 → 0).
- **Updates**: Applied in reverse order (2 → 1 → 0).

---

## Basic StatefulSet Example

```yaml
# Headless service — required for StatefulSet DNS
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  labels:
    app: postgres
spec:
  clusterIP: None             # Headless
  selector:
    app: postgres
  ports:
  - name: postgres
    port: 5432
    targetPort: 5432
---
# Client-facing service (optional — for external access)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
    role: primary              # Only route to primary
  ports:
  - port: 5432
    targetPort: 5432
---
# StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless   # Must match headless service name
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
          name: postgres
        env:
        - name: POSTGRES_DB
          value: mydb
        - name: POSTGRES_USER
          value: myuser
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
        readinessProbe:
          exec:
            command: ["pg_isready", "-U", "myuser", "-d", "mydb"]
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          exec:
            command: ["pg_isready", "-U", "myuser", "-d", "mydb"]
          initialDelaySeconds: 30
          periodSeconds: 10
  # VolumeClaimTemplate — creates one PVC per Pod
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-premium    # Azure Premium SSD
      resources:
        requests:
          storage: 10Gi
```

---

## VolumeClaimTemplates

The `volumeClaimTemplates` section automatically creates a PersistentVolumeClaim for each Pod:

- `postgres-data-postgres-0` → bound to Pod `postgres-0`
- `postgres-data-postgres-1` → bound to Pod `postgres-1`
- `postgres-data-postgres-2` → bound to Pod `postgres-2`

```bash
# View PVCs created for a StatefulSet
kubectl get pvc -l app=postgres

# NAME                      STATUS   VOLUME                    CAPACITY
# postgres-data-postgres-0  Bound    pvc-abc123...             10Gi
# postgres-data-postgres-1  Bound    pvc-def456...             10Gi
# postgres-data-postgres-2  Bound    pvc-ghi789...             10Gi
```

> ⚠️ **Warning:** Deleting a StatefulSet does NOT delete its PVCs by default. You must delete PVCs manually if you want to remove the data.

---

## Update Strategies

### RollingUpdate (Default)

Updates Pods one at a time, in reverse ordinal order.

```yaml
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0      # Only update pods with ordinal >= partition (0 = all)
```

**Canary with partition:**

```yaml
# Update only pod-2 first (partition: 2 means only ordinal >= 2)
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 2
```

```bash
# Update the image
kubectl set image statefulset/postgres postgres=postgres:16

# Only postgres-2 is updated (partition: 2)
# Verify, then reduce partition to update all
kubectl patch statefulset postgres -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

### OnDelete

Pods are only updated when you manually delete them:

```yaml
spec:
  updateStrategy:
    type: OnDelete
```

---

## Pod Management Policy

```yaml
spec:
  podManagementPolicy: OrderedReady    # Default: ordered start/stop
  # or
  podManagementPolicy: Parallel        # Start all pods simultaneously
```

Use `Parallel` when ordering is not required (e.g., caching clusters) for faster scaling.

---

## StatefulSet Operations

```bash
# Deploy
kubectl apply -f statefulset.yaml

# Check status
kubectl get statefulset postgres
kubectl describe statefulset postgres

# Watch pod creation (ordered)
kubectl get pods -l app=postgres --watch

# Scale up (new pods created in order: 3, 4, ...)
kubectl scale statefulset postgres --replicas=5

# Scale down (pods removed in reverse order: 4, 3, ...)
kubectl scale statefulset postgres --replicas=3

# Update image
kubectl set image statefulset/postgres postgres=postgres:16

# Check rollout
kubectl rollout status statefulset/postgres

# Rollback
kubectl rollout undo statefulset/postgres
```

---

## Accessing Individual Pods

```bash
# Access a specific StatefulSet pod by DNS
kubectl exec -it postgres-0 -- psql -U myuser -d mydb

# From another pod
kubectl exec -it my-app-pod -- psql -h postgres-0.postgres-headless.default.svc.cluster.local -U myuser -d mydb

# Connect to primary only via regular service
kubectl exec -it my-app-pod -- psql -h postgres.default.svc.cluster.local -U myuser -d mydb
```

---

## Real-World Example: Redis Cluster

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 6          # 3 masters + 3 replicas in Redis cluster mode
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2
        command: ["redis-server"]
        args:
        - /etc/redis/redis.conf
        - --cluster-enabled yes
        - --cluster-config-file /data/nodes.conf
        - --cluster-node-timeout 5000
        - --appendonly yes
        ports:
        - containerPort: 6379
          name: redis
        - containerPort: 16379
          name: cluster
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        volumeMounts:
        - name: redis-data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: managed-csi
      resources:
        requests:
          storage: 5Gi
```

---

## Summary

| Feature | Key Points |
|---------|-----------|
| Pod Names | Stable, ordered: `name-0`, `name-1`, `name-2` |
| Headless Service | Required for per-pod DNS |
| VolumeClaimTemplates | One PVC per Pod, survives Pod deletion |
| Ordered Operations | Sequential start, reverse sequential stop |
| Update Strategy | RollingUpdate (reverse order) or OnDelete |
| Partition | Canary updates with controlled rollout |

---

## Next Steps

Proceed to [04 - Networking](../04-networking/README.md) to learn about Kubernetes networking and Ingress.
