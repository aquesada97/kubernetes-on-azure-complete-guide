# kubectl Basics

kubectl is the command-line interface for interacting with Kubernetes clusters. This document covers essential commands with real-world examples.

---

## Installation and Configuration

```bash
kubectl version --client
kubectl config view
kubectl config set-context --current --namespace=my-namespace
```

---

## Getting Resources

### kubectl get

```bash
kubectl get nodes
kubectl get nodes -o wide

kubectl get pods
kubectl get pods -n kube-system
kubectl get pods -A
kubectl get pods -o wide
kubectl get pods --watch
kubectl get pods -l app=nginx

kubectl get deployments
kubectl get deploy

kubectl get services
kubectl get svc

kubectl get all
kubectl get all -n my-namespace

kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

kubectl get pod my-pod -o yaml
kubectl get pod my-pod -o json
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
```

### Useful Output Formats

| Flag | Output |
|------|--------|
| (default) | Formatted table |
| `-o wide` | Table with extra columns |
| `-o yaml` | YAML format |
| `-o json` | JSON format |
| `-o jsonpath='{.field}'` | Extract specific field |
| `-o name` | Resource name only |

---

## Describing Resources

```bash
kubectl describe pod my-pod
kubectl describe pod my-pod -n kube-system
kubectl describe node aks-systempool-12345678-vmss000000
kubectl describe deployment my-deployment
kubectl describe service my-service
```

The `describe` output includes:
- Resource metadata (labels, annotations)
- Spec details
- Status and conditions
- Events (very useful for debugging)

---

## Viewing Logs

```bash
kubectl logs my-pod
kubectl logs my-pod -f
kubectl logs -f my-pod --since=1h
kubectl logs my-pod --previous
kubectl logs my-pod -c my-container
kubectl logs -l app=nginx --all-containers
kubectl logs my-pod --tail=100
kubectl logs deployment/my-deployment
```

---

## Executing Commands in Pods

```bash
kubectl exec -it my-pod -- /bin/bash
kubectl exec -it my-pod -- /bin/sh

kubectl exec my-pod -- ls /app
kubectl exec my-pod -- env
kubectl exec my-pod -- cat /etc/config/app.conf

kubectl exec -it my-pod -c my-container -- /bin/bash

kubectl exec -it my-pod -- curl http://other-service:8080/health
kubectl exec -it my-pod -- nslookup kubernetes.default
```

---

## Port Forwarding

Forward local port to a pod or service for testing without exposing publicly.

```bash
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward service/my-service 8080:80
kubectl port-forward deployment/my-deployment 8080:80
kubectl port-forward pod/my-pod 8080:80 --address=0.0.0.0
```

Then access at `http://localhost:8080`:

```bash
curl http://localhost:8080
```

---

## Applying Manifests

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server

kubectl diff -f deployment.yaml
```

---

## Deleting Resources

```bash
kubectl delete pod my-pod
kubectl delete deployment my-deployment
kubectl delete -f deployment.yaml
kubectl delete pods -l app=nginx
kubectl delete all --all -n my-namespace
kubectl delete pod my-pod --grace-period=0 --force
```

---

## Creating Resources

```bash
kubectl create deployment nginx --image=nginx:1.25
kubectl expose deployment nginx --port=80 --type=LoadBalancer
kubectl create namespace my-namespace
kubectl create configmap my-config --from-literal=key1=value1
kubectl create secret generic my-secret --from-literal=password=mypassword

# Generate YAML without applying
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml
```

---

## Scaling and Rolling Updates

```bash
kubectl scale deployment my-deployment --replicas=5

kubectl rollout status deployment my-deployment
kubectl rollout history deployment my-deployment
kubectl rollout undo deployment my-deployment
kubectl rollout undo deployment my-deployment --to-revision=2
kubectl rollout pause deployment my-deployment
kubectl rollout resume deployment my-deployment

kubectl set image deployment/my-deployment my-container=nginx:1.26
```

---

## Debugging Tools

```bash
# Run a temporary debug pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
kubectl run -it --rm debug --image=curlimages/curl --restart=Never -- sh

# Debug a node
kubectl debug node/aks-systempool-123-vmss000000 -it --image=mcr.microsoft.com/dotnet/nightly/runtime-deps:7.0-jammy-chiseled

# Debug a crashing pod (creates a copy with modifications)
kubectl debug my-pod -it --image=ubuntu --copy-to=debug-pod

# Resource usage
kubectl top nodes
kubectl top pods
kubectl top pods -n kube-system --sort-by=memory
```

---

## kubectl Aliases (Recommended)

Add these to your `~/.bashrc` or `~/.zshrc`:

```bash
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kl='kubectl logs'
alias ke='kubectl exec -it'
alias ka='kubectl apply -f'
alias kdel='kubectl delete'
alias kns='kubectl config set-context --current --namespace'

source <(kubectl completion bash)
complete -F __start_kubectl k
```

---

## Common Troubleshooting Commands

```bash
# Get recent events (sorted by time)
kubectl get events --sort-by=.metadata.creationTimestamp -n default

# Check pod resource usage
kubectl top pod my-pod --containers

# Check if a pod can reach a service
kubectl exec -it my-pod -- wget -O- http://my-service:80 --timeout=5

# Check DNS resolution
kubectl exec -it my-pod -- nslookup my-service.default.svc.cluster.local

# Inspect a failed container
kubectl describe pod my-pod | grep -A 10 "Last State"

# Get all non-running pods
kubectl get pods -A --field-selector=status.phase!=Running
```

---

## Next Steps

Continue to [03-connect-to-cluster.md](./03-connect-to-cluster.md) to learn about kubeconfig and managing multiple clusters.
