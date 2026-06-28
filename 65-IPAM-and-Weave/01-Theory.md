# 65 — IP Address Management (IPAM) and Weave Net

## What is IPAM?

IP Address Management is how the cluster decides **which pod gets which IP**, ensuring no two pods get the same address, and that routing between pods works correctly. Every CNI plugin implements its own IPAM strategy (or delegates to a plugin like `host-local` or `whereabouts`).

---

## IPAM Models

| Model | How it works | Used by |
|-------|--------------|---------|
| **Host-local** | Each node owns a fixed CIDR block, allocates IPs sequentially | Flannel, Calico (default) |
| **Cluster-wide pool** | Central IPAM server hands out IPs from one pool | Calico (calico-ipam), Cilium |
| **Cloud-provider IPAM** | Pod IPs come from VPC subnet directly (no overlay) | AWS VPC CNI, GCP (Alias IP) |

---

## Host-local IPAM Example (Flannel/Calico default)

```
Cluster Pod CIDR: 10.244.0.0/16
   ├── Node 1: 10.244.0.0/24   (256 IPs)
   ├── Node 2: 10.244.1.0/24
   └── Node 3: 10.244.2.0/24
```

Each node's kubelet requests a `/24` block from the API server (`spec.podCIDR`), and the CNI plugin's IPAM allocates individual pod IPs from that local block.

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
```

---

## What is Weave Net?

**Weave Net** is a CNI plugin that creates a mesh overlay network between nodes. Each node runs a `weave` agent that:
- Encapsulates pod traffic and routes it across nodes via a mesh
- Implements its own IPAM allocating from a single cluster-wide pool
- Supports both **fast datapath** (kernel-level, faster) and **sleeve mode** (fallback, encrypted, NAT-traversal capable)

---

## Weave Architecture

```
Pod A (node1) → weave bridge → weave agent (node1)
                                      ↓ mesh (UDP 6783/6784)
                                weave agent (node2) → weave bridge → Pod B (node2)
```

- Default subnet: `10.32.0.0/12` (configurable)
- IPAM is **cluster-wide** — no per-node fixed block; Weave dynamically allocates and reclaims ranges between nodes via a gossip protocol

---

## Installing Weave Net

```bash
kubectl apply -f "https://github.com/weaveworks/weave/releases/download/latest_release/weave-daemonset-k8s.yaml"

# Verify
kubectl get pods -n kube-system | grep weave
```

---

## Weave CLI / Status

```bash
# Exec into a weave pod
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# Check connections between nodes
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status connections

# Check IPAM allocation
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status ipam
```

---

## Weave vs Other CNIs (IPAM Perspective)

| Plugin | IPAM Style | Overlay? | NetworkPolicy |
|--------|-----------|----------|---------------|
| Flannel | host-local, per-node CIDR | Yes (VXLAN) | ❌ |
| Calico | host-local or calico-ipam, BGP routed | No (usually) | ✅ |
| Weave | cluster-wide gossip pool | Yes (mesh) | ✅ |
| Cilium | cluster-wide, eBPF | Optional | ✅ (+ L7) |

---

## Custom IPAM Configuration (CNI Config)

```json
{
  "cniVersion": "0.3.1",
  "name": "mynet",
  "type": "bridge",
  "ipam": {
    "type": "host-local",
    "subnet": "10.244.1.0/24",
    "rangeStart": "10.244.1.10",
    "rangeEnd": "10.244.1.250",
    "gateway": "10.244.1.1"
  }
}
```

File location: `/etc/cni/net.d/10-mynet.conf` on each node.

---

## Diagnosing IP Exhaustion

```bash
# Check pod CIDR size per node
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Count running pods per node
kubectl get pods -A -o wide --field-selector=status.phase=Running | awk '{print $8}' | sort | uniq -c

# A /24 block = 256 IPs - if maxPods is set higher than available IPs, new pods fail with "no IP addresses available"
```

If a node runs out of IPs in its allocated block, new pods get stuck `ContainerCreating` with a CNI IPAM error in events.
