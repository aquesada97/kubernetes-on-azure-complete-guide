# Cluster Autoscaler

The **Cluster Autoscaler** automatically adjusts the number of nodes in a node pool when pods cannot be scheduled (scale up) or nodes are underutilized (scale down).

---

## How It Works

### Scale Up Trigger
1. A Pod is in `Pending` state because no node has sufficient resources
2. Cluster Autoscaler detects the pending pod
3. Evaluates which node pool can accommodate the pod
4. Adds a new node to the node pool
5. Pod is scheduled on the new node

### Scale Down Trigger
1. A node is underutilized (all pods using < 50% of requested resources)
2. All pods on the node can be moved to other nodes
3. Cluster Autoscaler safely drains the node
4. Removes the node from the pool

---

## Enable Cluster Autoscaler

### During Cluster Creation

```bash
az aks create \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10 \
  --node-count 3 \
  ...
```

### On Existing Cluster

```bash
# Enable on existing system pool
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10

# Update min/max counts
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --update-cluster-autoscaler \
  --min-count 3 \
  --max-count 15

# Disable
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --disable-cluster-autoscaler
```

### On a Specific Node Pool

```bash
az aks nodepool update \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name userpool \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20
```

---

## Cluster Autoscaler Profile

Fine-tune autoscaler behavior with the cluster autoscaler profile:

```bash
az aks update \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --cluster-autoscaler-profile \
    scan-interval=10s \
    scale-down-delay-after-add=10m \
    scale-down-unneeded-time=10m \
    scale-down-utilization-threshold=0.5 \
    max-graceful-termination-sec=600 \
    balance-similar-node-groups=true \
    expander=least-waste \
    skip-nodes-with-system-pods=true
```

### Key Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `scan-interval` | 10s | How often CA checks for pending pods |
| `scale-down-delay-after-add` | 10m | Wait after scale-up before considering scale-down |
| `scale-down-unneeded-time` | 10m | Node must be unneeded for this long before removal |
| `scale-down-utilization-threshold` | 0.5 | Node CPU/memory utilization threshold for scale-down |
| `max-graceful-termination-sec` | 600 | Max time to wait for pod termination |
| `balance-similar-node-groups` | false | Balance nodes across similar node pools |
| `expander` | random | Strategy for selecting node pool to scale up |
| `skip-nodes-with-system-pods` | true | Don't scale down nodes with system pods |
| `max-node-provision-time` | 15m | Max time to wait for a node to be provisioned |

### Expander Strategies

| Expander | Description | When to Use |
|----------|-------------|-------------|
| `random` | Random node group selection | Simple setups |
| `least-waste` | Minimizes unused CPU/memory after scale-up | Most efficient |
| `most-pods` | Maximizes number of schedulable pods | Maximize throughput |
| `price` | Lowest cost (requires cloud provider support) | Cost optimization |
| `priority` | Uses node group priorities | Custom ordering |

---

## Multiple Node Pools with Autoscaler

Configure different autoscaling settings per pool:

```bash
# System pool — always keep nodes available
az aks nodepool update \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name systempool \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5

# CPU-optimized user pool — scales aggressively
az aks nodepool update \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name cpupool \
  --enable-cluster-autoscaler \
  --min-count 0 \           # Scale to zero when not needed
  --max-count 50

# Spot pool for batch workloads — large range
az aks nodepool add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name spotpool \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 100 \
  --node-count 0
```

---

## Nodes That Cannot Be Scaled Down

The Cluster Autoscaler won't scale down nodes that have:

- Pods with local storage (hostPath, emptyDir with non-empty data)
- Pods with `PodDisruptionBudget` violations
- Pods that are not managed by a controller (standalone pods)
- Pods with annotation `cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`
- System pods (if `skip-nodes-with-system-pods: true`)

### Mark Pods as Safe to Evict

```yaml
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

### Prevent Node Scale-Down

```yaml
# On the node (set by node annotation)
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"
```

---

## Scale to Zero

Node pools can scale to zero when `min-count: 0`. This is useful for:
- Spot instance pools used only for batch jobs
- GPU pools that are not always needed
- Environment-specific pools (e.g., test workloads)

```bash
# Node pool that can scale to zero
az aks nodepool add \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name batchpool \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 20 \
  --node-count 0
```

For scale-from-zero to work, pods must have `nodeSelector`, `nodeAffinity`, or `tolerations` that match the empty node pool, so CA knows which pool to use.

---

## Monitoring Cluster Autoscaler

```bash
# Check CA status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml

# View CA logs
kubectl logs -n kube-system -l app=cluster-autoscaler --tail=100

# Check node pool counts
az aks nodepool list \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "[].{name:name, count:count, minCount:minCount, maxCount:maxCount}" \
  --output table

# Check pending pods (CA will trigger scale-up)
kubectl get pods -A --field-selector=status.phase=Pending
```

---

## Node Surge for Upgrades

Configure extra nodes during upgrades to maintain availability:

```bash
az aks nodepool update \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --name systempool \
  --max-surge 1              # Add 1 extra node during upgrades
```

---

## Summary

| Feature | Key Points |
|---------|-----------|
| Scale up trigger | Pending pods with unschedulable resources |
| Scale down trigger | Underutilized nodes (< utilization threshold) |
| Scale to zero | min-count=0 for demand-based pools |
| Expander strategies | least-waste is most efficient |
| PDB respect | CA won't evict pods violating PDB |
| Profile tuning | Adjust scan-interval, delay, threshold |

---

## Next Steps

Proceed to [09 - Advanced](../09-advanced/README.md) to learn about cluster upgrades, multi-tenancy, and disaster recovery.
