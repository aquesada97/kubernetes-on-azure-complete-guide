# Deployments

A **Deployment** is the primary workload resource for running stateless applications on Kubernetes. It provides declarative updates, rolling deployments, and automatic rollback capabilities.

---

## What is a Deployment?

A Deployment manages a set of identical Pods via a ReplicaSet. It ensures:
- A specified number of Pods are always running
- Updates are applied gradually (rolling updates)
- Failed updates can be rolled back

```
Deployment
  └── ReplicaSet (current)
        ├── Pod 1
        ├── Pod 2
        └── Pod 3
```

---

## Basic Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default
  labels:
    app: web-app
    version: "1.0"
spec:
  replicas: 3                          # Desired number of pods
  selector:
    matchLabels:
      app: web-app                     # Must match pod template labels
  template:
    metadata:
      labels:
        app: web-app
        version: "1.0"
    spec:
      containers:
      - name: web
        image: nginx:1.25
        ports:
        - containerPort: 80
          name: http
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        readinessProbe:               # Pod is ready to receive traffic
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
          failureThreshold: 3
        livenessProbe:                # Restart pod if this fails
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
```

```bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods -l app=web-app
```

---

## Deployment Strategies

### RollingUpdate (Default)

Gradually replaces old Pods with new ones. Zero downtime if configured correctly.

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Max extra pods above desired count during update
      maxUnavailable: 0    # Max pods that can be unavailable during update
```

With `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 0`:
- During update: up to 4 pods exist, but always 3 serving traffic
- Old pods are removed only after new pods pass readiness checks

### Recreate

All existing Pods are killed before new ones are created. Results in downtime.

```yaml
spec:
  strategy:
    type: Recreate
```

Use when: new and old versions cannot run simultaneously (e.g., database schema migrations).

### Strategy Comparison

| Strategy | Downtime | Speed | Use Case |
|----------|----------|-------|----------|
| RollingUpdate | No | Slower | Most production apps |
| Recreate | Yes | Fastest | Breaking changes, DB migrations |

---

## Rolling Update Example

```bash
# Trigger a rolling update by changing the image
kubectl set image deployment/web-app web=nginx:1.26

# Watch the rollout
kubectl rollout status deployment/web-app

# View rollout history
kubectl rollout history deployment/web-app

# View details of a specific revision
kubectl rollout history deployment/web-app --revision=2

# Roll back to previous version
kubectl rollout undo deployment/web-app

# Roll back to specific revision
kubectl rollout undo deployment/web-app --to-revision=1
```

---

## Health Probes

Health probes are critical for zero-downtime deployments and self-healing.

### Readiness Probe

Determines if the Pod is ready to receive traffic. Pod is removed from Service endpoints when this fails.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10    # Wait before first probe
  periodSeconds: 5           # Check every 5 seconds
  successThreshold: 1        # Successes needed to become ready
  failureThreshold: 3        # Failures before marking not ready
  timeoutSeconds: 3          # Probe timeout
```

### Liveness Probe

Determines if the Pod is healthy. Pod is restarted when this fails.

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
```

### Startup Probe

Gives slow-starting containers time to initialize before liveness kicks in.

```yaml
startupProbe:
  httpGet:
    path: /health/startup
    port: 8080
  failureThreshold: 30    # 30 * 10s = 5 minutes to start
  periodSeconds: 10
```

### Probe Types

| Type | Uses | Example |
|------|------|---------|
| `httpGet` | HTTP endpoint returns 2xx/3xx | REST APIs, web servers |
| `tcpSocket` | TCP connection success | Databases, raw TCP services |
| `exec` | Command exits with code 0 | Custom health scripts |
| `grpc` | gRPC health check | gRPC services |

---

## Resource Requests and Limits

```yaml
resources:
  requests:
    cpu: "250m"       # 0.25 vCPU — guaranteed minimum
    memory: "256Mi"   # 256 MB — guaranteed minimum
  limits:
    cpu: "500m"       # 0.5 vCPU — throttled if exceeded
    memory: "512Mi"   # 512 MB — OOMKilled if exceeded
```

### CPU Units

- `1000m` = 1 vCPU core
- `500m` = 0.5 vCPU core
- `100m` = 0.1 vCPU core

### Memory Units

- `256Mi` = 256 Mebibytes (~268 MB)
- `1Gi` = 1 Gibibyte (~1.07 GB)

### Quality of Service Classes

| QoS Class | Condition | Eviction Priority |
|-----------|-----------|-------------------|
| Guaranteed | requests == limits | Last evicted |
| Burstable | requests < limits | Middle |
| BestEffort | No requests/limits | First evicted |

---

## Pod Scheduling Controls

### Node Affinity

Schedule pods on specific nodes based on labels:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values:
            - linux
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: agentpool
            operator: In
            values:
            - highcpu
```

### Pod Anti-Affinity

Spread pods across nodes (avoid SPOF):

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-app
          topologyKey: kubernetes.io/hostname
```

### Topology Spread Constraints

Evenly distribute pods across zones:

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-app
```

### Taints and Tolerations

Node taints repel pods. Tolerations allow pods to schedule on tainted nodes:

```yaml
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
```

---

## Init Containers

Init containers run to completion before app containers start. Used for:
- Database migration
- Configuration generation
- Waiting for dependencies

```yaml
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z db-service 5432; do echo waiting; sleep 2; done']
  containers:
  - name: app
    image: my-app:1.0
```

---

## Complete Production Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-api
  namespace: production
  labels:
    app: production-api
    tier: backend
spec:
  replicas: 3
  revisionHistoryLimit: 5          # Keep 5 old ReplicaSets for rollback
  selector:
    matchLabels:
      app: production-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: production-api
        tier: backend
    spec:
      serviceAccountName: production-api-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: api
        image: myregistry.azurecr.io/production-api:1.5.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: production-api
              topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: production-api
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| Deployment | Manages ReplicaSets, handles updates and rollbacks |
| RollingUpdate | Zero-downtime updates with configurable surge and unavailability |
| Readiness Probe | Gates traffic to ready pods only |
| Liveness Probe | Restarts unhealthy pods automatically |
| Resource Limits | Prevents resource starvation, defines QoS |
| Pod Anti-Affinity | Distributes pods across nodes/zones for resilience |

---

## Next Steps

Continue to [02-services.md](./02-services.md) to learn how to expose your Deployments.
