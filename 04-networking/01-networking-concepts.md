# Networking Concepts

Understanding Kubernetes networking is essential for designing secure, efficient applications on AKS. This document covers how traffic flows between pods, services, and external clients.

---

## The Four Kubernetes Networking Rules

Kubernetes enforces these networking rules:

1. **Every Pod gets its own IP address**
2. **Pods on any node can communicate with all other pods on any node without NAT**
3. **Agents on a node can communicate with all pods on that node**
4. **Pods in the host network can communicate with all pods on all nodes without NAT**

These rules mean: if you know a Pod's IP, you can reach it directly from any other Pod in the cluster.

---

## Pod Networking

### IP Address Assignment

When a Pod is created, the CNI plugin assigns it an IP address from the Pod CIDR.

```bash
# View pod IP addresses
kubectl get pods -o wide

# NAME          READY   STATUS    IP            NODE
# web-app-0     1/1     Running   10.244.1.5    aks-nodepool-123
# web-app-1     1/1     Running   10.244.2.3    aks-nodepool-456
# web-app-2     1/1     Running   10.244.1.8    aks-nodepool-123
```

### Pod-to-Pod Communication

Pods can communicate directly without going through a Service:

```bash
# From inside a pod, reach another pod directly
kubectl exec -it web-app-0 -- curl http://10.244.2.3:8080

# Better: use Services (stable DNS) instead of pod IPs
kubectl exec -it web-app-0 -- curl http://my-service.default.svc.cluster.local:80
```

### Container-to-Container (Same Pod)

Containers in the same Pod share a network namespace, so they communicate via `localhost`:

```yaml
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8080
  - name: sidecar
    image: my-sidecar:1.0
    # Sidecar can reach app at localhost:8080
```

---

## Service Networking

### How ClusterIPs Work

1. You create a Service with a selector
2. The Endpoints controller creates an Endpoints object listing matching Pod IPs
3. kube-proxy on each node creates iptables/IPVS rules for the ClusterIP
4. Traffic to ClusterIP:Port is redirected (load balanced) to a Pod IP

```bash
# Inspect the iptables rules for a service (on a node)
iptables -t nat -L KUBE-SERVICES | grep my-service

# View endpoints
kubectl get endpoints my-service
```

### DNS Resolution Flow

```
App Pod → CoreDNS (kube-dns Service) → ClusterIP → Pod
            ↓
   my-service.default.svc.cluster.local → 10.0.0.50
```

```bash
# Check DNS from inside a pod
kubectl run -it --rm dnstest --image=busybox --restart=Never -- \
  nslookup my-service.default.svc.cluster.local
```

### Service CIDR vs Pod CIDR

```
Azure VNet: 10.0.0.0/8
  ├── Node Subnet: 10.240.0.0/16  (node VMs)
  ├── Pod CIDR: 192.168.0.0/16    (pod IPs, Azure CNI Overlay)
  └── Service CIDR: 10.0.0.0/16  (ClusterIP virtual IPs)
```

```bash
# Check cluster networking config
az aks show \
  --name $CLUSTER_NAME \
  --resource-group $RESOURCE_GROUP \
  --query networkProfile
```

---

## Azure CNI Overlay Deep Dive

With Azure CNI Overlay (recommended), pods get IPs from a private CIDR using overlay (VXLAN) encapsulation:

```
Pod CIDR: 192.168.0.0/16 (private, not in VNet)
Node VNet IP: 10.240.0.4

Traffic from Pod A (192.168.1.5) to Pod B (192.168.2.3):
1. Encapsulated in VXLAN at Node A
2. Transported as UDP over VNet (10.240.0.4 → 10.240.0.5)
3. Decapsulated at Node B
4. Delivered to Pod B
```

This provides:
- Pod density not limited by VNet IP space
- Full Azure CNI features (Network Policies, Windows nodes)
- Better performance than Kubenet (no NAT)

---

## DNS in AKS

### CoreDNS Configuration

```bash
# View CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS ConfigMap
kubectl get configmap coredns -n kube-system -o yaml

# Check DNS from a pod
kubectl run -it --rm dnstest --image=busybox --restart=Never -- sh
# Inside pod:
# nslookup kubernetes.default
# nslookup my-service.default.svc.cluster.local
# nslookup google.com
```

### DNS Search Domains

Pods automatically have DNS search domains configured:

```
search default.svc.cluster.local svc.cluster.local cluster.local
```

This means from a pod in `default` namespace, you can use:
- `my-service` → resolves to `my-service.default.svc.cluster.local`
- `my-service.other-ns` → resolves to `my-service.other-ns.svc.cluster.local`
- `google.com` → forwarded to upstream DNS

```bash
# View DNS config inside a pod
kubectl exec my-pod -- cat /etc/resolv.conf
# nameserver 10.0.0.10          ← CoreDNS ClusterIP
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

### Custom DNS Forwarding

Forward queries for specific domains to custom DNS servers:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  # Forward contoso.com queries to internal corporate DNS
  contoso.server: |
    contoso.com:53 {
        forward . 10.1.0.4 10.1.0.5
    }
  # Forward azure.com queries to Azure DNS
  azure.server: |
    azure.com:53 {
        forward . 168.63.129.16
    }
```

---

## Network Troubleshooting

### Common Tools

```bash
# Test connectivity from a pod
kubectl run -it --rm nettest \
  --image=nicolaka/netshoot \
  --restart=Never -- bash

# Inside netshoot:
# ping 10.244.1.5             ← ping a pod IP
# curl http://my-service:80   ← test service
# nslookup my-service         ← check DNS
# traceroute 10.244.1.5       ← trace route
# ss -tulnp                   ← check listening ports

# Test a specific endpoint
kubectl exec -it my-pod -- curl -v http://other-service:8080/health

# Check if service has endpoints
kubectl get endpoints my-service

# Describe service for event details
kubectl describe service my-service
```

### Diagnosing "Connection Refused"

```bash
# 1. Verify pods are running
kubectl get pods -l app=my-app

# 2. Verify service selector matches pod labels
kubectl describe service my-service | grep Selector
kubectl get pods -l <selector> --show-labels

# 3. Verify service has endpoints
kubectl get endpoints my-service

# 4. Verify correct port
kubectl describe service my-service | grep -A 5 Port
kubectl describe pod my-pod | grep -A 5 containerPort

# 5. Test directly to pod IP (bypass service)
POD_IP=$(kubectl get pod my-pod -o jsonpath='{.status.podIP}')
kubectl exec -it another-pod -- curl http://$POD_IP:8080
```

---

## Summary

| Concept | Key Points |
|---------|-----------|
| Pod IP | Every pod gets unique IP; direct pod-to-pod communication |
| ClusterIP | Virtual IP for Service; stable DNS name |
| kube-proxy | Implements Service networking via iptables |
| CoreDNS | Cluster DNS; resolves service names to ClusterIPs |
| Azure CNI Overlay | Pod IPs in overlay network; best of both worlds |
| ndots:5 | DNS search logic; understand for cross-namespace resolution |

---

## Next Steps

Continue to [02-ingress.md](./02-ingress.md) to learn how to expose multiple services via a single Ingress controller.
