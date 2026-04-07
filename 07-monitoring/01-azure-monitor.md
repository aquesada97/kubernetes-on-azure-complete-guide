# Azure Monitor and Container Insights

Azure Monitor Container Insights provides native monitoring for AKS, collecting metrics and logs from your cluster and sending them to a Log Analytics Workspace.

---

## Enable Container Insights

### During Cluster Creation

```bash
# Create Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group $RESOURCE_GROUP \
  --workspace-name law-aks-training \
  --location $LOCATION

LAW_ID=$(az monitor log-analytics workspace show \
  --resource-group $RESOURCE_GROUP \
  --workspace-name law-aks-training \
  --query id -o tsv)

# Create cluster with monitoring enabled
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-addons monitoring \
  --workspace-resource-id $LAW_ID \
  ...
```

### On Existing Cluster

```bash
az aks enable-addons \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --addons monitoring \
  --workspace-resource-id $LAW_ID
```

### Verify Installation

```bash
# Check monitoring addon pods
kubectl get pods -n kube-system -l component=oms-agent
kubectl get pods -n kube-system | grep omsagent

# Check daemonset
kubectl get daemonset ama-logs -n kube-system
```

---

## Container Insights in Azure Portal

Navigate to: **Azure Portal → Your AKS Cluster → Monitoring → Insights**

### Key Views

#### Cluster Overview
- Node count and health
- Pod count (running, pending, failed)
- Node CPU and memory utilization trends
- Active container count

#### Node View
- Per-node CPU and memory utilization
- Node conditions
- Disk I/O per node

#### Controllers View
- Deployment, DaemonSet, StatefulSet health
- Pod restarts per controller

#### Containers View
- Per-container CPU and memory
- Container restart count and reason
- Live logs

---

## Log Analytics and KQL

Container Insights stores data in Log Analytics tables. Query them with **Kusto Query Language (KQL)**.

### Key Tables

| Table | Contains |
|-------|---------|
| `ContainerLog` | Container stdout/stderr logs |
| `ContainerLogV2` | Container logs (newer format, recommended) |
| `KubePodInventory` | Pod metadata and status |
| `KubeNodeInventory` | Node metadata and status |
| `KubeEvents` | Kubernetes events |
| `ContainerInventory` | Container metadata |
| `Perf` | CPU and memory metrics |

### Common KQL Queries

#### View Container Logs

```kql
// Recent logs from a specific container
ContainerLogV2
| where ContainerName == "my-app"
| where TimeGenerated > ago(1h)
| order by TimeGenerated desc
| take 100
```

```kql
// Error logs across all containers
ContainerLogV2
| where TimeGenerated > ago(1h)
| where LogMessage contains "ERROR" or LogMessage contains "FATAL"
| project TimeGenerated, PodName, ContainerName, LogMessage
| order by TimeGenerated desc
```

#### Pod Status

```kql
// Pods not in Running state
KubePodInventory
| where TimeGenerated > ago(5m)
| where PodStatus != "Running" and PodStatus != "Succeeded"
| distinct PodName, PodStatus, Namespace, ContainerStatus
| order by PodName
```

```kql
// Pod restart counts
KubePodInventory
| where TimeGenerated > ago(1h)
| summarize RestartCount = max(ContainerRestartCount) by PodName, ContainerName, Namespace
| where RestartCount > 0
| order by RestartCount desc
```

#### Node Health

```kql
// Node CPU utilization
Perf
| where ObjectName == "K8SNode"
| where CounterName == "cpuUsageNanoCores"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

```kql
// Node memory utilization
Perf
| where ObjectName == "K8SNode"
| where CounterName == "memoryRssBytes"
| where TimeGenerated > ago(1h)
| summarize AvgMemoryGB = avg(CounterValue) / (1024*1024*1024) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

#### Kubernetes Events

```kql
// Warning events in the last hour
KubeEvents
| where TimeGenerated > ago(1h)
| where Level == "Warning"
| project TimeGenerated, Namespace, Name, Reason, Message
| order by TimeGenerated desc
```

```kql
// OOMKilled events
KubeEvents
| where TimeGenerated > ago(24h)
| where Reason == "OOMKilling"
| project TimeGenerated, Namespace, Name, Message
```

#### Container Resource Usage

```kql
// Top containers by CPU
Perf
| where ObjectName == "K8SContainer"
| where CounterName == "cpuUsageNanoCores"
| where TimeGenerated > ago(5m)
| summarize AvgCPU = avg(CounterValue) / 1000000 by InstanceName
| top 10 by AvgCPU desc
```

---

## Configuring Log Collection

Customize what Container Insights collects using the `ConfigMap`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: container-azm-ms-agentconfig
  namespace: kube-system
data:
  schema-version: v1
  config-version: ver1
  log-data-collection-settings: |-
    [log_collection_settings]
       [log_collection_settings.stdout]
          enabled = true
          exclude_namespaces = ["kube-system", "gatekeeper-system"]
       [log_collection_settings.stderr]
          enabled = true
          exclude_namespaces = ["kube-system", "gatekeeper-system"]
       [log_collection_settings.env_var]
          enabled = true
       [log_collection_settings.enrich_container_logs]
          enabled = false
  agent-settings: |-
    [agent_settings.restart_settings]
       max_restarts = 6
```

---

## Azure Monitor Alerts

### Create a Metric Alert (Portal)

1. Navigate to: **Azure Portal → Your AKS Cluster → Monitoring → Alerts**
2. Click **+ Create** → **Alert rule**
3. Configure:
   - **Signal**: Select metric (e.g., "Nodes Count", "Pod Count", "CPU Usage")
   - **Condition**: Define threshold
   - **Action Group**: Configure notification (email, webhook, Teams)

### Create Alerts with Azure CLI

```bash
# Alert when pending pods > 0 for > 5 minutes
az monitor alert create \
  --name "aks-pending-pods" \
  --resource-group $RESOURCE_GROUP \
  --target $(az aks show --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --query id -o tsv) \
  --condition "count 'insights.container/pods' 'podStatus = \"Pending\"' > 0" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action-group my-action-group

# Log Analytics alert using KQL
az monitor scheduled-query create \
  --name "aks-oom-kills" \
  --resource-group $RESOURCE_GROUP \
  --scopes $LAW_ID \
  --condition "count > 0" \
  --condition-query "KubeEvents | where Reason == 'OOMKilling' | count" \
  --window-duration PT5M \
  --evaluation-frequency PT5M \
  --severity 2
```

### Recommended Alerts

| Alert | KQL/Metric | Severity |
|-------|-----------|---------|
| Pod in CrashLoopBackOff | KubePodInventory ContainerStatus contains "CrashLoopBackOff" | Sev 1 |
| Node NotReady | KubeNodeInventory Status contains "NotReady" | Sev 1 |
| High CPU > 80% | Perf cpuUsageNanoCores | Sev 2 |
| High Memory > 80% | Perf memoryRssBytes | Sev 2 |
| OOMKill events | KubeEvents Reason == "OOMKilling" | Sev 2 |
| Persistent volume filling | Perf volumeFsUsedBytes > 80% | Sev 2 |

---

## Live Container Logs

View real-time container logs directly from the Azure Portal:

1. Navigate to: **AKS Cluster → Insights → Containers**
2. Select a container
3. Click **Live Data** tab

Or using kubectl:

```bash
kubectl logs -f my-pod -c my-container
kubectl logs -f -l app=my-app --all-containers
```

---

## Summary

| Feature | Key Points |
|---------|-----------|
| Container Insights | Azure-native monitoring, auto-installed with addon |
| Log Analytics | Stores logs and metrics; query with KQL |
| ContainerLogV2 | Recommended table for container logs |
| KubePodInventory | Pod status and restart counts |
| Azure Monitor Alerts | Alert on metrics or log query results |

---

## Next Steps

Continue to [02-prometheus-grafana.md](./02-prometheus-grafana.md) to deploy the Prometheus + Grafana stack.
