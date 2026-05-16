# 63 — Pod Networking and CNI Plugins

## The Kubernetes Networking Model

Kubernetes imposes these fundamental networking requirements:

1. **Every pod gets its own IP address** — no NAT between pods
2. **All pods can communicate with all other pods** — across nodes, without NAT
3. **Pods can communicate with nodes** — and vice versa
4. **Services have their own stable IP** — separate from pod IPs

This model is simple from the pod's perspective — it sees one flat network. The implementation details are handled by the CNI plugin.

---

## What is CNI?

**Container Network Interface (CNI)** is a specification and a set of libraries for configuring network interfaces in Linux containers. Kubernetes uses CNI plugins to:

- Assign IP addresses to pods
- Set up network interfaces inside pods
- Configure routing rules on the node
- Implement NetworkPolicy (optional, plugin-dependent)

---

## How a Pod Gets its Network

When a pod is created:

```
1. kubelet creates the pod's network namespace
         ↓
2. kubelet calls the CNI plugin: "set up networking for this pod"
         ↓
3. CNI plugin:
   - Creates a virtual ethernet pair (veth)
   - One end in pod namespace (eth0)
   - Other end on the host (vethXXXX)
   - Assigns IP from the pod CIDR
   - Sets up routes
         ↓
4. Pod has eth0 with an IP — can communicate
```

---

## veth Pair: How Pods Connect to the Node

```
┌─────────────────┐         ┌─────────────────────────┐
│   Pod Network   │         │       Node Network       │
│   Namespace     │         │                          │
│  ┌───────────┐  │         │  ┌────────────────────┐  │
│  │   eth0    │◄─┼──veth───┼─►│    vethXXXX        │  │
│  │10.244.0.5 │  │   pair  │  │    (host side)     │  │
│  └───────────┘  │         │  └────────────────────┘  │
└─────────────────┘         │            │              │
                            │         bridge/overlay    │
                            └─────────────────────────┘
```

---

## Cross-Node Pod Communication

Two approaches:

### Overlay Network (e.g. Flannel VXLAN, Weave)
Encapsulates pod traffic in UDP packets to cross node boundaries:

```
Pod A (node1) → veth → bridge → VXLAN encapsulation → eth0 (node1 NIC)
    → network → eth0 (node2 NIC) → VXLAN decapsulation → bridge → veth → Pod B (node2)
```

### Underlay / BGP (e.g. Calico BGP, Cilium)
Uses routing protocols — pod IPs are directly routable on the network:

```
Pod A (node1) → veth → node1 routes pod CIDR for node2 via BGP → Pod B (node2)
```
No encapsulation overhead — more performant.

---

## Popular CNI Plugins

| Plugin | Approach | NetworkPolicy | Performance | Notes |
|--------|---------|--------------|-------------|-------|
| **Flannel** | VXLAN overlay | ❌ No | Medium | Simple, no NetworkPolicy |
| **Calico** | BGP / VXLAN | ✅ Yes | High | Most popular, full NetworkPolicy |
| **Cilium** | eBPF | ✅ Yes | Very High | eBPF-based, advanced observability |
| **Weave** | Overlay | ✅ Yes | Medium | Easy setup |
| **Canal** | Flannel + Calico | ✅ Yes | Medium | Combines both |
| **Antrea** | OVS / eBPF | ✅ Yes | High | VMware, good for vSphere |

---

## Pod CIDR and Node CIDR

```bash
# Each node gets a subnet from the cluster pod CIDR
# Cluster CIDR: 10.244.0.0/16

# Node 1: 10.244.0.0/24  (pods get IPs from this range)
# Node 2: 10.244.1.0/24
# Node 3: 10.244.2.0/24

# Check pod CIDR
kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

# Check cluster CIDR (in kube-controller-manager config)
kubectl describe pod kube-controller-manager-controlplane -n kube-system | grep cluster-cidr
```

---

## CNI Plugin Files

```
/etc/cni/net.d/           ← CNI config files (which plugin to use)
/opt/cni/bin/             ← CNI plugin binaries
```

```bash
# Check which CNI is configured
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist   # Flannel example

# Check plugin binaries
ls /opt/cni/bin/
```

---

## Inspecting Pod Networking

```bash
# Pod IP
kubectl get pod mypod -o jsonpath='{.status.podIP}'

# All pods with IPs
kubectl get pods -o wide

# Pod's network namespace from inside the pod
kubectl exec mypod -- ip addr
kubectl exec mypod -- ip route
kubectl exec mypod -- cat /etc/resolv.conf

# On the node: see veth pairs
ip link show type veth

# On the node: see pod routes
ip route | grep 10.244
```

---

## Service Networking vs Pod Networking

| | Pod Network | Service Network |
|-|------------|----------------|
| IP range | `--cluster-cidr` | `--service-cluster-ip-range` |
| Implemented by | CNI plugin | kube-proxy (iptables/IPVS) |
| Routable on nodes | Yes (via CNI routes) | No (virtual, iptables only) |
| Example range | 10.244.0.0/16 | 10.96.0.0/12 |

---

## Troubleshooting Pod Networking

```bash
# Pod can't reach another pod
kubectl exec pod-a -- ping <pod-b-ip>
kubectl exec pod-a -- curl http://<pod-b-ip>:8080

# Check if CNI plugin is running
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium|weave"

# Check CNI logs
kubectl logs -n kube-system calico-node-xxx

# Node-level checks
ip route              # routing table on node
iptables -L -n -t nat  # iptables NAT rules (kube-proxy)
```
