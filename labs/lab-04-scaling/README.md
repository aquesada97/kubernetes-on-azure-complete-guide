# Lab 04: Autoscaling

In this lab, you will configure Horizontal Pod Autoscaler (HPA) and observe the Cluster Autoscaler in action as you generate load on the application.

**Estimated Time:** 45 minutes  
**Difficulty:** Intermediate

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

- Configure HPA based on CPU utilization
- Generate load to trigger pod autoscaling
- Observe Cluster Autoscaler adding nodes
- Tune HPA scale-down behavior
- View autoscaling events and history

---

## Prerequisites

- Completed [Lab 03](../lab-03-storage/README.md)
- AKS cluster with Cluster Autoscaler enabled
- metrics-server running (enabled by default on AKS)

---

## Lab Setup

```bash
kubectl create namespace lab04
kubectl config set-context --current --namespace=lab04
```

---

## Part 1: Deploy the Target Application

```bash
# Deploy a CPU-intensive application
kubectl create deployment php-apache \
  --image=registry.k8s.io/hpa-example \
  --namespace=lab04

# Set resource requests (REQUIRED for HPA)
kubectl set resources deployment php-apache \
  --requests=cpu=200m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi \
  --namespace=lab04

# Expose as a service
kubectl expose deployment php-apache \
  --port=80 \
  --namespace=lab04

kubectl get pods,services -n lab04
```

---

## Part 2: Apply the HPA

```bash
kubectl apply -f manifests/hpa.yaml

# Check HPA status
kubectl get hpa -n lab04
# Initially shows <unknown>/50% — wait for metrics
sleep 30
kubectl get hpa -n lab04
# NAME         REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# php-apache   Deployment/php-apache 5%/50%    2         20        2          60s
```

---

## Part 3: Generate Load

### 3.1 Start a Load Generator

Open a **second terminal window** and run:

```bash
kubectl run -it --rm load-generator \
  --image=busybox \
  --namespace=lab04 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache.lab04.svc.cluster.local; done"
```

### 3.2 Watch HPA Scale Up (First Terminal)

```bash
# Watch HPA react in real-time
kubectl get hpa php-apache -n lab04 --watch
```

Observe:
1. CPU % climbs above 50%
2. HPA increases replicas
3. CPU % drops as more pods share the load

```bash
# Also watch pods being created
kubectl get pods -n lab04 --watch
```

### 3.3 Stop the Load Generator

In the second terminal, press `Ctrl+C` to stop the load generator. The `--rm` flag means the pod is automatically deleted.

---

## Part 4: Observe Scale-Down

```bash
# After stopping load, watch HPA scale down
kubectl get hpa php-apache -n lab04 --watch

# Scale-down has a cooldown (default 5 minutes)
# HPA will not scale down until CPU stays below threshold
```

---

## Part 5: View HPA Events

```bash
# See HPA events
kubectl describe hpa php-apache -n lab04

# Look for events like:
# "Scaled up replica count to X from Y"
# "Scaled down replica count to X from Y"

# Check recent events
kubectl get events -n lab04 --sort-by=.metadata.creationTimestamp | grep HorizontalPodAutoscaler
```

---

## Part 6: Cluster Autoscaler Observation

If your cluster has the Cluster Autoscaler enabled, generate enough load to exhaust current node capacity:

```bash
# Scale deployment to force node addition
kubectl scale deployment php-apache --replicas=50 -n lab04

# Watch for pending pods (CA trigger)
kubectl get pods -n lab04 | grep Pending

# Watch nodes being added
kubectl get nodes --watch

# Check CA status
kubectl get configmap cluster-autoscaler-status -n kube-system -o yaml | grep -A 20 scaleUp
```

```bash
# After observing, scale back down
kubectl scale deployment php-apache --replicas=2 -n lab04

# Watch nodes being removed (takes ~10 minutes)
kubectl get nodes --watch
```

---

## Part 7: HPA with Custom Behavior

Apply a modified HPA that scales up fast but scales down slowly:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-tuned
  namespace: lab04
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # Scale up immediately
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300      # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 2                           # Remove at most 2 pods at a time
        periodSeconds: 60
EOF
```

---

## Part 8: kubectl top for Verification

```bash
# Pod resource usage
kubectl top pods -n lab04

# Node resource usage
kubectl top nodes

# Sort by CPU
kubectl top pods -n lab04 --sort-by=cpu
```

---

## Cleanup

```bash
kubectl delete namespace lab04
```

---

## ✅ Lab Completion Checklist

- [ ] Deployed application with resource requests set
- [ ] Applied HPA targeting 50% CPU
- [ ] Generated load and observed HPA scaling up
- [ ] Stopped load and observed HPA scaling down
- [ ] Viewed HPA events and history
- [ ] Applied custom HPA behavior (fast up, slow down)

---

## 🤔 Discussion Questions

1. Why do resource requests need to be set for HPA to work?
2. What is the default scale-down stabilization window and why is it important?
3. How would you scale based on a custom metric (e.g., requests per second)?
4. When would KEDA be better than HPA?

---

## Next Lab

Proceed to [Lab 05: RBAC and Security](../lab-05-rbac/README.md).
