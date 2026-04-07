# AKS Components

This document covers the key components that run within an AKS cluster, including networking, DNS, container runtime, and Azure integrations.

---

## CoreDNS

**CoreDNS** is the default DNS server in AKS. It provides DNS-based service discovery within the cluster.

### How It Works

Every Pod in the cluster gets a DNS resolver pointing to the CoreDNS Service (`kube-dns` in the `kube-system` namespace). CoreDNS resolves:

- **Service names**: `my-service` → ClusterIP
- **Fully qualified names**: `my-service.my-namespace.svc.cluster.local`
- **External names**: forwarded to upstream DNS

### DNS Resolution Pattern

```
<service-name>.<namespace>.svc.<cluster-domain>
```

Example: `my-api.production.svc.cluster.local`

```bash
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup kubernetes.default
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup my-service.default.svc.cluster.local
```

### CoreDNS Configuration

CoreDNS is configured via a ConfigMap in `kube-system`:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Custom forwarding (e.g., forward company domain to internal DNS):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns-custom
  namespace: kube-system
data:
  contoso.server: |
    contoso.com:53 {
        forward . 10.1.0.4
    }
```

---

## kube-proxy

**kube-proxy** runs on every node and implements Kubernetes Service networking. It maintains network rules (iptables or IPVS) so that packets destined for a Service ClusterIP are redirected to a Pod.

### Modes

| Mode | Description | Default in AKS |
|------|-------------|---------------|
| `iptables` | Uses iptables rules | Yes |
| `ipvs` | Uses IP Virtual Server (better for large clusters) | Available |

### What kube-proxy Does

1. Watches the API server for new Services and Endpoints
2. Creates iptables rules for each Service
3. When a packet hits a Service VIP, iptables redirects it to one of the backing Pods

```bash
kubectl get pods -n kube-system -l component=kube-proxy
kubectl logs -n kube-system -l component=kube-proxy --tail=20
```

---

## Container Runtime: containerd

AKS uses **containerd** as the container runtime (Docker was removed in Kubernetes 1.24+). containerd is a CNCF graduated project and implements the Container Runtime Interface (CRI).

### containerd vs Docker

| Feature | containerd | Docker |
|---------|------------|--------|
| CRI compliance | Native | Via dockershim (removed) |
| Overhead | Lightweight | Higher |
| Image compatibility | Docker images work | Docker images work |
| Used in AKS | Yes (1.19+) | No (deprecated) |

### containerd Tools

```bash
# On the node (via kubectl debug or SSH)
crictl ps
crictl pull nginx:1.25
crictl inspect <container-id>
```

---

## Azure CNI vs Kubenet

The Container Network Interface (CNI) plugin determines how Pod networking works. AKS supports multiple CNI options.

### Kubenet

Pods get IPs from a separate `podCIDR` that is NOT routable in the Azure VNet.

```
Azure VNet: 10.0.0.0/16
  Node IPs: 10.0.1.0/24
  Pod IPs (separate): 192.168.0.0/16 (non-routable in VNet)
```

**Pros:** Simpler setup, conserves VNet IP space  
**Cons:** Pods not directly accessible from VNet, limited to 400 nodes, no Windows node pool support

### Azure CNI

Pods get IPs directly from the Azure VNet subnet.

```
Azure VNet: 10.0.0.0/8
  Node IPs: 10.240.0.0/16
  Pod IPs: 10.244.0.0/16 (same VNet, routable)
```

**Pros:** Pods directly accessible from VNet, no NAT overhead, Windows node pool support  
**Cons:** Uses more IP addresses, requires larger subnet planning

### Azure CNI Overlay (Recommended for New Clusters)

Azure CNI Overlay combines the best of both: pods get IPs from a private CIDR without consuming VNet IPs.

```bash
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 192.168.0.0/16 \
  ...
```

### CNI Comparison

| Feature | Kubenet | Azure CNI | Azure CNI Overlay |
|---------|---------|-----------|-------------------|
| Pod IP in VNet | No | Yes | No (overlay) |
| IP consumption | Low | High | Low |
| Max nodes | 400 | 1000+ | 1000+ |
| Windows support | No | Yes | Yes |
| Network Policies | Limited | Full | Full |
| Performance | Good | Best | Very good |

---

## Azure Load Balancer Integration

AKS integrates with **Azure Load Balancer** to expose Services of type `LoadBalancer`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-public-service
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-sku: standard
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: my-app
```

### Internal Load Balancer

For services that should only be accessible within the VNet:

```yaml
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "my-internal-subnet"
```

---

## Node Resource Group Components

### Virtual Machine Scale Sets (VMSS)

Each node pool is backed by a VMSS. AKS manages the VMSS — do not modify it directly.

```bash
az vmss list \
  --resource-group MC_rg-aks-training_aks-training-cluster_eastus \
  --output table
```

### Azure Managed Disks

OS disks for nodes are Azure Managed Disks. The disk type depends on VM size and node pool configuration.

### Network Security Groups (NSG)

Automatically configured NSGs control inbound/outbound traffic to nodes. AKS manages NSG rules for:
- Node-to-node communication
- Load balancer health probes
- SSH access (if enabled)

---

## Summary

| Component | Purpose |
|-----------|---------|
| CoreDNS | Cluster DNS, service discovery |
| kube-proxy | Service networking via iptables/IPVS |
| containerd | Container runtime (CRI) |
| Azure CNI | Pod networking with VNet IPs |
| Kubenet | Pod networking with private CIDR + NAT |
| Azure Load Balancer | External Service exposure |
| VMSS | Backing compute for node pools |

---

## Next Steps

Proceed to [02 - Getting Started](../02-getting-started/README.md) to create your first AKS cluster.
