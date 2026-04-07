# Horizontal Pod Autoscaler (HPA)

The **Horizontal Pod Autoscaler** automatically scales the number of Pod replicas in a Deployment, ReplicaSet, or StatefulSet based on observed metrics.

---

## Prerequisites

HPA requires the **metrics-server** to be running:

```bash
# Check if metrics-server is installed
kubectl get pods -n kube-system -l k8s-app=metrics-server

# Install metrics-server if not present
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system

# Verify
kubectl top pods -n default
kubectl top nodes
```

---

## Basic HPA (CPU-Based)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app               # Target Deployment to scale
  minReplicas: 2                # Never scale below this
  maxReplicas: 10               # Never scale above this
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Target 70% CPU utilization
```

```bash
kubectl apply -f hpa.yaml

# View HPA status
kubectl get hpa web-app-hpa
# NAME          REFERENCE         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# web-app-hpa   Deployment/web-app  45%/70%   2         10        3          5m

# Describe for detailed info
kubectl describe hpa web-app-hpa
```

### Requirements for HPA

The target Deployment **must** have CPU requests defined:

```yaml
resources:
  requests:
    cpu: "200m"     # Required — HPA calculates utilization as actual/request
```

---

## HPA with Memory

```yaml
spec:
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80    # Scale when avg memory > 80% of request
```

Or using absolute value:

```yaml
spec:
  metrics:
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: "200Mi"     # Scale when avg memory > 200Mi per pod
```

---

## HPA with Multiple Metrics

HPA evaluates all metrics and scales to satisfy **all** of them (takes the maximum):

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60      # Wait 60s before scaling up again
      policies:
      - type: Percent
        value: 100                         # Can double replicas at once
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300     # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1                           # Remove at most 1 pod per minute
        periodSeconds: 60
```

---

## Custom Metrics HPA

Scale based on application metrics from Prometheus via the **prometheus-adapter**.

### Install Prometheus Adapter

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus-adapter prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local \
  --set prometheus.port=9090
```

### Configure Custom Metric

```yaml
# ConfigMap for prometheus-adapter
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_per_second{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_per_second$"
        as: "${1}_per_second"
      metricsQuery: 'sum(rate(http_requests_total{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'
```

### HPA Using Custom Metric

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 50
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"        # Scale to maintain 100 req/s per pod
```

---

## KEDA: Event-Driven Autoscaling

**KEDA** (Kubernetes Event-Driven Autoscaling) extends HPA to support scaling based on external events: Azure Service Bus, Storage Queues, Event Hubs, Kafka, etc.

### Install KEDA

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace
```

### Scale on Azure Service Bus Queue

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: azure-servicebus-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: message-processor
  minReplicaCount: 0            # Can scale to zero!
  maxReplicaCount: 30
  pollingInterval: 30           # Poll every 30s
  cooldownPeriod: 300
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: my-orders-queue
      namespace: myservicebus.servicebus.windows.net
      messageCount: "5"         # 1 pod per 5 messages
    authenticationRef:
      name: azure-servicebus-trigger-auth
---
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: azure-servicebus-trigger-auth
spec:
  podIdentity:
    provider: azure-workload
```

### Scale on Azure Storage Queue

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: storage-queue-scaler
  namespace: default
spec:
  scaleTargetRef:
    name: worker-deployment
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: azure-queue
    metadata:
      queueName: work-items
      accountName: mystorageaccount
      queueLength: "10"
    authenticationRef:
      name: storage-trigger-auth
```

---

## Load Testing HPA

```bash
# Deploy a test deployment
kubectl create deployment php-apache \
  --image=registry.k8s.io/hpa-example \
  --requests=cpu=200m

kubectl expose deployment php-apache --port=80

# Create HPA
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=1 \
  --max=10

# Generate load (in a separate terminal)
kubectl run -it --rm load-generator \
  --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# Watch HPA react (in another terminal)
kubectl get hpa php-apache --watch
```

---

## HPA Best Practices

1. **Always set resource requests** — HPA cannot function without them
2. **Set minReplicas >= 2** — for zero-downtime deployments
3. **Use scale-down stabilization** — prevent rapid scale-down oscillation
4. **Don't set minReplicas = 0** unless using KEDA (HPA can't scale from 0)
5. **Monitor HPA events** — `kubectl describe hpa` shows why scaling was triggered
6. **Combine with Cluster Autoscaler** — scale pods AND nodes together

---

## Summary

| Feature | Key Points |
|---------|-----------|
| CPU-based HPA | Most common; requires resource.requests.cpu |
| Memory-based HPA | Good for memory-heavy workloads |
| Custom metrics HPA | Application metrics via prometheus-adapter |
| KEDA | Event-driven scaling; can scale to/from zero |
| Behavior config | Control scale-up speed and scale-down cooldown |

---

## Next Steps

Continue to [02-vpa.md](./02-vpa.md) to learn about Vertical Pod Autoscaler.
