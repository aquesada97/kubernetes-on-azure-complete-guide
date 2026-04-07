# Kubernetes Basics

## Containers vs Virtual Machines

Understanding the difference between containers and virtual machines is fundamental to understanding why Kubernetes exists.

### Virtual Machines

A virtual machine (VM) includes a full operating system (kernel + user space), virtualized hardware, and the application. VMs are isolated at the hypervisor level.

```
┌──────────────────────────────┐
│  ┌────────┐  ┌────────┐      │
│  │ App A  │  │ App B  │      │
│  ├────────┤  ├────────┤      │
│  │ OS     │  │ OS     │      │
│  └────────┘  └────────┘      │
│  ┌──────────────────────┐    │
│  │  Hypervisor          │    │
│  └──────────────────────┘    │
│  ┌──────────────────────┐    │
│  │  Physical Host       │    │
│  └──────────────────────┘    │
└──────────────────────────────┘
```

**Pros:** Strong isolation, run any OS, full OS access  
**Cons:** Heavy (GBs), slow startup (minutes), wasteful resource use

### Containers

A container shares the host OS kernel and uses Linux namespaces and cgroups for isolation. Containers package the application + its dependencies (libraries, runtime) but NOT the OS kernel.

```
┌────────────────────────────────┐
│  ┌──────────┐  ┌──────────┐   │
│  │  App A   │  │  App B   │   │
│  │  + libs  │  │  + libs  │   │
│  └──────────┘  └──────────┘   │
│  ┌────────────────────────┐   │
│  │  Container Runtime     │   │
│  ├────────────────────────┤   │
│  │  Host OS (shared)      │   │
│  └────────────────────────┘   │
└────────────────────────────────┘
```

**Pros:** Lightweight (MBs), fast startup (seconds), portable, consistent  
**Cons:** Weaker isolation than VMs, must use host OS kernel

### Comparison Table

| Feature | Container | Virtual Machine |
|---------|-----------|-----------------|
| Startup time | Seconds | Minutes |
| Size | MBs | GBs |
| OS | Shared host kernel | Full OS per VM |
| Isolation | Process-level (namespaces) | Hardware-level |
| Portability | Very high | Medium |
| Resource use | Efficient | Less efficient |
| Best for | Microservices, CI/CD | Legacy apps, full OS control |

---

## Kubernetes Architecture

Kubernetes is an open-source container orchestration platform. It automates deployment, scaling, and management of containerized applications.

### Control Plane

The **control plane** manages the cluster state and makes global decisions. It consists of:

#### API Server (`kube-apiserver`)
- The front end for the Kubernetes control plane
- Exposes the Kubernetes REST API
- All communication (kubectl, controllers, nodes) goes through the API server
- Validates and processes requests, then stores state in etcd

#### etcd
- A distributed key-value store
- Stores all cluster state (object definitions, configuration)
- The "source of truth" for the cluster
- Must be backed up for disaster recovery

#### Scheduler (`kube-scheduler`)
- Watches for newly created Pods with no assigned node
- Selects a node for the Pod based on:
  - Resource availability (CPU, memory)
  - Node affinity/anti-affinity rules
  - Taints and tolerations
  - Pod topology spread constraints

#### Controller Manager (`kube-controller-manager`)
- Runs controller loops that watch cluster state and reconcile desired vs. actual state
- Includes: Node controller, Replication controller, Endpoints controller, Service Account controller

#### Cloud Controller Manager
- Integrates Kubernetes with cloud provider APIs
- Manages: Load balancers, node lifecycle, volumes, routes
- In AKS, this handles Azure Load Balancer creation, Azure Disk provisioning, etc.

### Worker Nodes

Worker nodes run the actual application workloads. Each node contains:

#### kubelet
- An agent that runs on each node
- Communicates with the API server
- Ensures containers described in PodSpecs are running and healthy
- Reports node and Pod status back to the control plane

#### kube-proxy
- A network proxy running on each node
- Implements Kubernetes Service networking
- Maintains network rules that allow communication to Pods from inside and outside the cluster

#### Container Runtime
- The software that runs containers
- Kubernetes supports CRI (Container Runtime Interface) compatible runtimes
- AKS uses **containerd** (previously Docker)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                     Control Plane                        │
│  ┌──────────┐  ┌──────┐  ┌───────────┐  ┌──────────┐  │
│  │API Server│  │ etcd │  │ Scheduler │  │Controller│  │
│  └──────────┘  └──────┘  └───────────┘  │ Manager  │  │
│                                          └──────────┘  │
└─────────────────────────────────────────────────────────┘
         │              │              │
┌────────┴──┐    ┌──────┴────┐   ┌────┴──────┐
│  Node 1   │    │  Node 2   │   │  Node 3   │
│ ┌───────┐ │    │ ┌───────┐ │   │ ┌───────┐ │
│ │kubelet│ │    │ │kubelet│ │   │ │kubelet│ │
│ ├───────┤ │    │ ├───────┤ │   │ ├───────┤ │
│ │k-proxy│ │    │ │k-proxy│ │   │ │k-proxy│ │
│ ├───────┤ │    │ ├───────┤ │   │ ├───────┤ │
│ │runtime│ │    │ │runtime│ │   │ │runtime│ │
│ └───────┘ │    │ └───────┘ │   │ └───────┘ │
└───────────┘    └───────────┘   └───────────┘
```

---

## Core Kubernetes Objects

### Pod

The smallest deployable unit in Kubernetes. A Pod encapsulates one or more containers that share:
- Network namespace (same IP address)
- Storage volumes
- Lifecycle

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
  - name: my-container
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "200m"
        memory: "256Mi"
```

> 📝 **Note:** Pods are ephemeral — they are not self-healing. Use higher-level controllers (Deployment, StatefulSet) to manage Pods.

### ReplicaSet

Ensures a specified number of Pod replicas are running at any given time. Selects Pods using label selectors.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: my-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx:1.25
```

> 💡 **Tip:** In practice, you rarely create ReplicaSets directly — use Deployments instead.

### Deployment

Manages ReplicaSets and provides declarative updates for Pods. Supports rolling updates and rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-container
        image: nginx:1.25
        ports:
        - containerPort: 80
```

### Service

Provides a stable network endpoint to access a set of Pods. Pods come and go, but Services provide a consistent DNS name and IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Namespace

A mechanism for dividing cluster resources between multiple users or teams. Provides a scope for names.

```bash
kubectl create namespace my-team
kubectl get namespaces
kubectl get pods --namespace my-team
kubectl get pods -n my-team
```

Default namespaces in AKS:

| Namespace | Purpose |
|-----------|---------|
| `default` | Default namespace for user workloads |
| `kube-system` | Kubernetes system components |
| `kube-public` | Publicly accessible data |
| `kube-node-lease` | Node heartbeat leases |

### ConfigMap

Stores non-sensitive configuration data as key-value pairs. Decouples configuration from container images.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgresql://db:5432/mydb"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

### Secret

Stores sensitive data (passwords, tokens, keys) in base64-encoded form.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  username: bXl1c2Vy          # base64 encoded "myuser"
  password: bXlwYXNzd29yZA==  # base64 encoded "mypassword"
```

```bash
kubectl create secret generic app-secret \
  --from-literal=username=myuser \
  --from-literal=password=mypassword
```

---

## kubectl Intro Commands

kubectl is the command-line tool for interacting with Kubernetes clusters.

```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl get pods -A
kubectl describe pod my-pod
kubectl logs my-pod
kubectl logs my-pod -f
kubectl logs my-pod -c my-container
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- curl localhost:80
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/
kubectl delete pod my-pod
kubectl delete -f deployment.yaml
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl scale deployment my-deployment --replicas=5
kubectl rollout status deployment my-deployment
```

---

## Labels and Selectors

Labels are key-value pairs attached to objects. Selectors filter objects by labels.

```bash
kubectl label pod my-pod environment=production
kubectl get pods --selector app=my-app
kubectl get pods -l app=my-app,environment=production
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| Pod | Smallest unit, one or more containers, ephemeral |
| ReplicaSet | Maintains desired replica count |
| Deployment | Manages ReplicaSets, enables rolling updates |
| Service | Stable endpoint for Pod access |
| Namespace | Logical cluster partitioning |
| ConfigMap | Non-sensitive configuration storage |
| Secret | Sensitive data storage (base64 encoded) |

---

## Next Steps

Continue to [02-aks-architecture.md](./02-aks-architecture.md) to learn how AKS builds on these Kubernetes concepts.
