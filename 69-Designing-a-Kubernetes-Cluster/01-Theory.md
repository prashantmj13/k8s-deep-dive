# 69 — Designing a Kubernetes Cluster

## Key Design Decisions

Before building a cluster, decide on:

1. **Topology** — single control plane vs HA multi-master
2. **etcd placement** — stacked vs external
3. **Node sizing** — how many nodes, what specs
4. **Networking** — CNI plugin, pod/service CIDR ranges
5. **Storage** — which StorageClass / CSI driver
6. **Managed vs self-hosted** — EKS/GKE/AKS vs kubeadm/kops/k3s

---

## Control Plane Topology

### Single control-plane node (dev/test only)
```
[ Control Plane ] → [ Worker ] [ Worker ] [ Worker ]
```
Single point of failure — never use in production.

### Stacked etcd HA (most common)
```
[ CP1+etcd ] [ CP2+etcd ] [ CP3+etcd ]  ← odd number, 3 or 5
       ↓           ↓            ↓
[ Worker ] [ Worker ] [ Worker ] [ Worker ]
```
etcd runs on the same nodes as the control plane components. Simpler to operate, fewer nodes needed.

### External etcd HA (advanced/large scale)
```
[ etcd1 ] [ etcd2 ] [ etcd3 ]   ← separate etcd cluster
    ↑         ↑          ↑
[ CP1 ]    [ CP2 ]    [ CP3 ]
```
Decouples etcd failure domain from control plane failure domain — more resilient but more nodes to manage.

---

## Load Balancing the Control Plane

With multiple API servers, you need a load balancer in front:

```
kubectl / kubelets
        ↓
  Load Balancer (HAProxy, cloud LB, or keepalived+VIP)
        ↓        ↓        ↓
   API Server  API Server  API Server
```

```bash
# Example HAProxy config snippet
frontend k8s-api
    bind *:6443
    default_backend k8s-api-backend
backend k8s-api-backend
    balance roundrobin
    server cp1 192.168.1.10:6443 check
    server cp2 192.168.1.11:6443 check
    server cp3 192.168.1.12:6443 check
```

---

## Sizing Guidelines

| Component | Minimum | Recommended (medium cluster) |
|-----------|---------|------------------------------|
| Control plane CPU | 2 vCPU | 4+ vCPU |
| Control plane RAM | 2 GB | 8+ GB |
| etcd disk | any | SSD, low latency critical |
| Worker node | 2 vCPU / 2GB | depends on workload |
| Max pods per node | 110 (default) | tune based on node size |

> **Rule of thumb:** etcd write latency directly affects API responsiveness — always put etcd on fast SSDs, never on network-attached spinning disks.

---

## Network Planning

Choose CIDR ranges that **do not overlap** with your existing infrastructure:

```
Pod CIDR:     10.244.0.0/16   (~65k pod IPs)
Service CIDR: 10.96.0.0/12    (~1M service IPs, only need a few thousand)
Node subnet:  192.168.1.0/24  (your physical/VPC network)
```

Check for conflicts before cluster creation — changing CIDRs after init requires a cluster rebuild in most setups.

---

## Choosing a CNI Plugin

| Requirement | Suggested CNI |
|-------------|---------------|
| Simplicity, small cluster | Flannel |
| NetworkPolicy + performance | Calico |
| Deep observability + eBPF | Cilium |
| Multi-cloud overlay mesh | Weave |

---

## High Availability Checklist

- [ ] Odd number of control plane nodes (3 or 5)
- [ ] etcd on fast, dedicated SSD storage
- [ ] Load balancer in front of API servers
- [ ] Control plane nodes spread across failure domains (AZs/racks)
- [ ] Regular etcd backups (automated, tested restores)
- [ ] Worker nodes spread across multiple AZs
- [ ] PodDisruptionBudgets for critical workloads
- [ ] Cluster autoscaler or sufficient node headroom

---

## Managed vs Self-Hosted Decision Matrix

| Factor | Managed (EKS/GKE/AKS) | Self-hosted (kubeadm) |
|--------|------------------------|------------------------|
| Control plane maintenance | Provider handles it | You handle it |
| Upgrade complexity | Mostly automated | Manual, careful sequencing |
| Cost | Pay for control plane + nodes | Pay for nodes only (DIY ops cost) |
| Customization | Limited | Full control |
| On-prem/air-gapped | Not available | Required option |

---

## Reference Architecture (3-node HA, stacked etcd)

```
              [ Load Balancer :6443 ]
               /        |        \
        [ CP1+etcd ] [ CP2+etcd ] [ CP3+etcd ]
               \        |        /
         [ W1 ]    [ W2 ]    [ W3 ]    [ W4 ]
```

This is the topology `kubeadm` builds when you run `kubeadm init` on the first control plane node with `--control-plane-endpoint` pointing at the load balancer, then `kubeadm join --control-plane` on CP2 and CP3.
