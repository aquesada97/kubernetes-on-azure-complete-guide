# Vertical Pod Autoscaler (VPA)

The **Vertical Pod Autoscaler** automatically adjusts the CPU and memory requests (and optionally limits) for containers based on actual resource usage. It helps right-size pods for efficiency.

---

## HPA vs VPA

| Feature | HPA | VPA |
|---------|-----|-----|
| What scales | Number of replicas | Resource requests per pod |
| Trigger | CPU/memory utilization | Actual usage over time |
| Effect | More/fewer pods | Larger/smaller pods |
| Use with StatefulSets | Yes | Yes |
| Can run together? | ⚠️ Carefully (not on same metric) | Same workload is fine |

> ⚠️ **Warning:** Do NOT use HPA on CPU/memory AND VPA on the same Deployment. VPA changing requests will cause HPA to rescale, creating an update loop. Use HPA for replicas and VPA for sizing, or use VPA in `Off` or `Initial` mode alongside HPA.

---

## Install VPA

```bash
# Clone the VPA repo
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler

# Install VPA components
./hack/vpa-up.sh

# Or install via Helm
helm repo add cowboysysop https://cowboysysop.github.io/charts/
helm install vpa cowboysysop/vertical-pod-autoscaler \
  --namespace kube-system

# Verify
kubectl get pods -n kube-system | grep vpa
# vpa-admission-controller-xxx  Running
# vpa-recommender-xxx           Running
# vpa-updater-xxx               Running
```

### VPA Components

| Component | Function |
|-----------|---------|
| `vpa-recommender` | Analyzes resource usage, generates recommendations |
| `vpa-updater` | Evicts pods when VPA wants to update resources |
| `vpa-admission-controller` | Injects recommended resources on pod creation |

---

## VPA Modes

| Mode | Description |
|------|-------------|
| `Off` | Only generates recommendations, no automatic changes |
| `Initial` | Applies recommendations only when pod is first created |
| `Recreation` | Evicts and recreates pods to apply new recommendations |
| `Auto` | Applies updates when safe (may evict pods) |

---

## VPA Configuration

### Off Mode (Recommendation Only)

Use this to get sizing recommendations without any changes:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"          # Only recommend, never change
```

```bash
# Apply VPA
kubectl apply -f vpa.yaml

# Wait a few minutes, then view recommendations
kubectl describe vpa my-app-vpa

# Output:
# Recommendation:
#   Container Recommendations:
#     Container Name:  my-app
#     Lower Bound:
#       Cpu:     25m
#       Memory:  262144k
#     Target:
#       Cpu:     100m
#       Memory:  524288k
#     Uncapped Target:
#       Cpu:     100m
#       Memory:  524288k
#     Upper Bound:
#       Cpu:     1500m
#       Memory:  4Gi
```

### Initial Mode (Apply at Pod Creation)

```yaml
spec:
  updatePolicy:
    updateMode: "Initial"
```

New pods get the recommended resources injected. Existing pods are NOT changed until they're naturally restarted.

### Auto Mode (Full Automatic)

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa-auto
  namespace: default
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: my-app
      minAllowed:
        cpu: 50m                 # VPA won't recommend below this
        memory: 64Mi
      maxAllowed:
        cpu: 2                   # VPA won't recommend above this
        memory: 4Gi
      controlledResources:
      - cpu
      - memory
```

---

## Inspecting VPA Recommendations

```bash
# Get VPA recommendations in YAML
kubectl get vpa my-app-vpa -o yaml

# View all VPAs
kubectl get vpa -A

# Describe for detailed output
kubectl describe vpa my-app-vpa
```

### Recommendation Fields

| Field | Meaning |
|-------|---------|
| `Target` | Recommended values to apply |
| `LowerBound` | Minimum recommended (below may cause OOM or throttling) |
| `UpperBound` | Maximum recommended (above is wasteful) |
| `UncappedTarget` | Recommendation ignoring min/max constraints |

---

## Using VPA Alongside HPA

To use both HPA and VPA without conflict:

```yaml
# VPA in Initial mode (only for sizing new pods)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Initial"           # Only applies at pod creation
  resourcePolicy:
    containerPolicies:
    - containerName: my-app
      controlledResources: ["cpu", "memory"]
---
# HPA for replica scaling (not on CPU to avoid conflict)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second    # Custom metric, not CPU
      target:
        type: AverageValue
        averageValue: "100"
```

---

## VPA Best Practices

1. **Start with `Off` mode** to gather recommendations without disruption
2. **Use `minAllowed` and `maxAllowed`** to prevent extreme values
3. **Not suitable for single-replica StatefulSets** — eviction causes downtime
4. **Combine with PodDisruptionBudgets** to limit disruption during updates
5. **Monitor VPA events** — too-frequent evictions indicate unstable workloads
6. **Review recommendations weekly** for new workloads

---

## PodDisruptionBudget with VPA

Protect availability when VPA evicts pods:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2          # Always keep at least 2 pods running
  selector:
    matchLabels:
      app: my-app
```

```bash
kubectl get pdb
```

---

## Summary

| Mode | Use Case |
|------|---------|
| Off | Gather recommendations, no changes |
| Initial | Apply recommendations on new pods only |
| Recreation | Recreate pods with new resources (some downtime) |
| Auto | Fully automatic (suitable with PDB) |

---

## Next Steps

Continue to [03-cluster-autoscaler.md](./03-cluster-autoscaler.md) to learn about node-level autoscaling.
