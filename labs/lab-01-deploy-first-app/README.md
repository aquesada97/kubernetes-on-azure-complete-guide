# Lab 01: Deploy Your First Application

In this lab, you will deploy a sample web application to AKS, expose it via a Service, and practice essential kubectl operations.

**Estimated Time:** 45 minutes  
**Difficulty:** Beginner

---

## 🎯 Lab Objectives

By completing this lab, you will be able to:

- Deploy a containerized application using a Kubernetes Deployment
- Expose the application using a ClusterIP and LoadBalancer Service
- Scale the deployment manually and verify changes
- Perform a rolling update and roll back
- Read logs and execute commands inside pods

---

## Prerequisites

- AKS cluster running with kubectl access
- Completed Modules 01-03

---

## Lab Setup

```bash
# Create a namespace for this lab
kubectl create namespace lab01
kubectl config set-context --current --namespace=lab01
```

---

## Part 1: Deploy the Application

The sample application is the Azure Voting App — a simple Python + Redis app.

### 1.1 Apply the Manifests

```bash
# Navigate to the lab directory
cd labs/lab-01-deploy-first-app

# Apply all manifests
kubectl apply -f manifests/

# Verify deployment
kubectl get deployments
kubectl get pods
kubectl get services
```

### 1.2 Watch Pod Creation

```bash
# Watch pods come up
kubectl get pods --watch

# Once Running, press Ctrl+C to stop watching
```

### 1.3 Inspect Resources

```bash
# Describe the deployment
kubectl describe deployment azure-vote-front

# Get pod details
kubectl get pods -o wide

# View pod logs
kubectl logs -l app=azure-vote-front

# Check service
kubectl get service azure-vote-front --watch
# Wait for EXTERNAL-IP to be assigned
```

---

## Part 2: Access the Application

```bash
# Get the External IP
EXTERNAL_IP=$(kubectl get service azure-vote-front -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Application URL: http://$EXTERNAL_IP"
```

Open `http://$EXTERNAL_IP` in your browser. You should see a voting interface for Cats vs Dogs.

```bash
# Or test from command line
curl -s http://$EXTERNAL_IP | grep -o "<title>.*</title>"
```

---

## Part 3: Scale the Deployment

### 3.1 Scale Up

```bash
# Scale the front-end to 5 replicas
kubectl scale deployment azure-vote-front --replicas=5

# Watch pods being created
kubectl get pods --watch

# Verify
kubectl get deployment azure-vote-front
```

### 3.2 Observe Pod Distribution

```bash
# See which nodes the pods are on
kubectl get pods -o wide
```

### 3.3 Scale Down

```bash
kubectl scale deployment azure-vote-front --replicas=2
kubectl get pods --watch
```

---

## Part 4: Rolling Update

### 4.1 Check Current Image

```bash
kubectl describe deployment azure-vote-front | grep Image
```

### 4.2 Update the Image

```bash
# Update to a different version (simulates a new release)
kubectl set image deployment/azure-vote-front azure-vote-front=mcr.microsoft.com/azuredocs/azure-vote-front:v2

# Watch the rolling update
kubectl rollout status deployment/azure-vote-front

# Watch pods being replaced
kubectl get pods --watch
```

### 4.3 View Rollout History

```bash
kubectl rollout history deployment/azure-vote-front
```

### 4.4 Roll Back

```bash
# Roll back to the previous version
kubectl rollout undo deployment/azure-vote-front

# Verify rollback
kubectl rollout status deployment/azure-vote-front
kubectl describe deployment azure-vote-front | grep Image
```

---

## Part 5: Pod Inspection

### 5.1 Execute Commands in a Pod

```bash
# Get a pod name
POD_NAME=$(kubectl get pods -l app=azure-vote-front -o jsonpath='{.items[0].metadata.name}')

# Open a shell in the pod
kubectl exec -it $POD_NAME -- /bin/bash

# Inside the pod:
# ls /app
# env | grep -i azure
# curl http://azure-vote-back:6379/ping
# exit
```

### 5.2 View Real-Time Logs

```bash
# Follow logs
kubectl logs -f $POD_NAME

# Or follow all pods with the label
kubectl logs -f -l app=azure-vote-front --all-containers
```

### 5.3 Port-Forward for Direct Access

```bash
# Port-forward to bypass the LoadBalancer
kubectl port-forward pod/$POD_NAME 8080:80

# In another terminal:
curl http://localhost:8080
```

---

## Part 6: Resource Management

### 6.1 Check Resource Usage

```bash
# Top pods
kubectl top pods

# Top nodes
kubectl top nodes
```

### 6.2 View Events

```bash
kubectl get events --sort-by=.metadata.creationTimestamp
```

---

## Cleanup

```bash
kubectl delete namespace lab01
```

---

## ✅ Lab Completion Checklist

- [ ] Deployed application using manifests
- [ ] Accessed application via LoadBalancer IP
- [ ] Scaled deployment to 5 replicas and back to 2
- [ ] Performed a rolling update and verified
- [ ] Rolled back the update
- [ ] Executed a command inside a pod
- [ ] Viewed pod logs

---

## 🤔 Discussion Questions

1. What happens to traffic during a rolling update?
2. Why does the LoadBalancer IP take time to be assigned?
3. What would happen if you set `replicas: 0`?
4. How does the Service know which pods to route traffic to?

---

## Next Lab

Proceed to [Lab 02: Configure Ingress](../lab-02-ingress/README.md).
