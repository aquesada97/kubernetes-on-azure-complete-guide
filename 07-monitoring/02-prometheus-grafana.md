# Prometheus and Grafana

This document covers deploying the Prometheus + Grafana observability stack on AKS for collecting custom metrics, creating dashboards, and configuring alerting.

---

## Overview

```
Pods → Prometheus Scraper → Prometheus TSDB → Grafana → Dashboards/Alerts
                ↑                    ↑
     /metrics endpoints    AlertManager → PagerDuty/Slack
```

- **Prometheus**: Pulls (scrapes) metrics from applications and stores them as time series
- **Grafana**: Visualizes Prometheus data in dashboards
- **AlertManager**: Routes Prometheus alerts to notification channels

---

## Install kube-prometheus-stack

The `kube-prometheus-stack` Helm chart installs Prometheus, Grafana, AlertManager, and all necessary exporters in one deployment.

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword="YourSecureGrafanaPassword123!" \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=managed-csi \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=20Gi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=managed-csi \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.resources.requests.storage=5Gi

# Verify installation
kubectl get pods -n monitoring
kubectl get services -n monitoring
```

---

## Access Grafana

```bash
# Port-forward to access Grafana locally
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Open http://localhost:3000
# Username: admin
# Password: YourSecureGrafanaPassword123!
```

Or expose via LoadBalancer (not recommended for production without auth):

```bash
kubectl patch svc kube-prometheus-stack-grafana -n monitoring \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc kube-prometheus-stack-grafana -n monitoring --watch
```

---

## Pre-built Dashboards

The kube-prometheus-stack includes ready-to-use Grafana dashboards:

| Dashboard | Description |
|-----------|-------------|
| Kubernetes / Cluster Overview | Cluster-wide CPU, memory, pod counts |
| Kubernetes / Nodes | Per-node resource utilization |
| Kubernetes / Pods | Per-pod resource usage |
| Kubernetes / Deployments | Deployment health and replicas |
| Kubernetes / Persistent Volumes | PVC usage |

Access dashboards at: **Grafana → Dashboards → Browse**

---

## Scraping Custom Application Metrics

### Instrument Your Application

Applications must expose a `/metrics` endpoint in Prometheus format.

#### Python (prometheus_client)

```python
from prometheus_client import Counter, Histogram, start_http_server
import time

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency')

@REQUEST_LATENCY.time()
def handle_request(method, endpoint):
    REQUEST_COUNT.labels(method=method, endpoint=endpoint).inc()
    # ... your app logic
    
if __name__ == '__main__':
    start_http_server(9090)  # Serve metrics on :9090/metrics
```

#### .NET (prometheus-net)

```csharp
using Prometheus;

var app = WebApplication.Create(args);
app.MapMetrics();  // Serves /metrics endpoint

// Custom counter
var counter = Metrics.CreateCounter("myapp_requests_total", "Total requests", "method");
counter.WithLabels("GET").Inc();
```

### Add Prometheus Annotations to Pods

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    prometheus.io/path: "/metrics"
```

### Create a ServiceMonitor (Preferred Method)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app-monitor
  namespace: monitoring               # Must be in monitoring namespace or configure namespaceSelector
  labels:
    release: kube-prometheus-stack    # Must match Prometheus selector
spec:
  namespaceSelector:
    matchNames:
    - default                         # Watch services in 'default' namespace
  selector:
    matchLabels:
      app: my-app                     # Selector for the Service
  endpoints:
  - port: metrics                     # Port name in the Service
    interval: 30s
    path: /metrics
```

```yaml
# Service must have a named port
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
  - name: http
    port: 8080
  - name: metrics                    # Named port for ServiceMonitor
    port: 9090
```

---

## PromQL Queries

**PromQL** (Prometheus Query Language) is used to query metrics.

### Common Queries

```promql
# CPU usage by pod (rate over last 5 min)
sum(rate(container_cpu_usage_seconds_total{namespace="default"}[5m])) by (pod)

# Memory usage by pod
sum(container_memory_working_set_bytes{namespace="default"}) by (pod)

# HTTP request rate
sum(rate(http_requests_total[5m])) by (method, endpoint)

# Error rate as percentage
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m])) 
  / sum(rate(http_requests_total[5m]))

# P95 latency
histogram_quantile(0.95, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)

# Pending pods
kube_pod_status_phase{phase="Pending"} == 1

# Container restarts in last hour
sum(increase(kube_pod_container_status_restarts_total[1h])) by (pod, container) > 0

# Node CPU utilization percentage
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (node))

# Disk space available (percentage)
100 * node_filesystem_avail_bytes / node_filesystem_size_bytes
```

---

## Creating Custom Grafana Dashboards

### Add a Dashboard

1. Grafana → **+** → **New Dashboard**
2. **Add new panel**
3. Enter PromQL query in the query field
4. Configure visualization (Time series, Gauge, Bar chart, Table)
5. Save dashboard

### Dashboard JSON Example

```json
{
  "title": "My App Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "timeseries",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (method)",
          "legendFormat": "{{method}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "gauge",
      "targets": [
        {
          "expr": "100 * sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))"
        }
      ],
      "thresholds": {
        "steps": [
          {"color": "green", "value": 0},
          {"color": "yellow", "value": 1},
          {"color": "red", "value": 5}
        ]
      }
    }
  ]
}
```

---

## AlertManager Configuration

### Define Prometheus Rules

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: my-app
    rules:
    - alert: HighErrorRate
      expr: |
        100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
        / sum(rate(http_requests_total[5m])) > 5
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

    - alert: PodCrashLooping
      expr: |
        increase(kube_pod_container_status_restarts_total[15m]) > 3
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Container {{ $labels.container }} in pod {{ $labels.pod }} restarted {{ $value }} times in 15 minutes"

    - alert: HighMemoryUsage
      expr: |
        container_memory_working_set_bytes{container!=""}
        / on(pod, namespace) kube_pod_container_resource_limits{resource="memory"}
        > 0.9
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Container {{ $labels.container }} memory near limit"
        description: "Container is using {{ $value | humanizePercentage }} of its memory limit"
```

### Configure AlertManager Routes

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-kube-prometheus-stack
  namespace: monitoring
type: Opaque
stringData:
  alertmanager.yaml: |
    global:
      resolve_timeout: 5m
      slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
    
    route:
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: default
      routes:
      - match:
          severity: critical
        receiver: critical-alerts
      - match:
          severity: warning
        receiver: warning-alerts
    
    receivers:
    - name: default
      slack_configs:
      - channel: '#aks-alerts'
        title: '{{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
    
    - name: critical-alerts
      slack_configs:
      - channel: '#aks-critical'
        title: '🚨 CRITICAL: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
        send_resolved: true
    
    - name: warning-alerts
      slack_configs:
      - channel: '#aks-warnings'
        title: '⚠️ WARNING: {{ .CommonAnnotations.summary }}'
        text: '{{ .CommonAnnotations.description }}'
```

---

## Azure Managed Prometheus (Preview)

Azure now offers a managed Prometheus service that eliminates the need to manage Prometheus yourself.

```bash
# Enable Azure Managed Prometheus
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/microsoft.monitor/accounts/myAMW

# Enable managed Grafana
az grafana create \
  --name myGrafana \
  --resource-group $RESOURCE_GROUP

az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --grafana-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/microsoft.dashboard/grafana/myGrafana
```

---

## Summary

| Component | Purpose |
|-----------|---------|
| Prometheus | Time-series metric scraping and storage |
| Grafana | Visualization dashboards |
| AlertManager | Alert routing and notification |
| ServiceMonitor | Tells Prometheus what to scrape |
| PrometheusRule | Defines alert conditions |
| PromQL | Query language for metrics |

---

## Next Steps

Proceed to [08 - Scaling](../08-scaling/README.md) to learn about automatic scaling in AKS.
