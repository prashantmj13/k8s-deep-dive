# 63 — Pod Networking and CNI | Solutions

---

## Exercise 2 — Pod CIDRs

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```
```
controlplane   10.244.0.0/24
node01         10.244.1.0/24
node02         10.244.2.0/24
```

Each node gets a /24 subnet. Pods on `node01` get IPs from `10.244.1.0/24`.

---

## Exercise 3 — Pod IPs

```bash
kubectl get pods -o wide
```
```
NAME    READY   STATUS    IP            NODE
pod-a   1/1     Running   10.244.0.5    controlplane
pod-b   1/1     Running   10.244.1.3    node01
```

Pod IPs come from their node's pod CIDR.

---

## Exercise 4 — Network inspection inside pod

```bash
kubectl exec pod-a -- ip addr
```
```
1: lo: <LOOPBACK> ...
    inet 127.0.0.1/8
3: eth0@if7: <BROADCAST,MULTICAST,UP,LOWER_UP>
    inet 10.244.0.5/32    ← pod's IP
```

```bash
kubectl exec pod-a -- ip route
```
```
default via 169.254.1.1 dev eth0      ← default gateway
169.254.1.1 dev eth0 scope link       ← Calico link-local gateway
```

```bash
kubectl exec pod-a -- curl -s http://10.244.1.3   # pod-b
# returns nginx HTML — direct pod-to-pod works across nodes
```

---

## Exercise 6 — Cross-namespace connectivity

```bash
kubectl exec pod-a -- curl -s http://$CROSS_IP
# Returns HTML — pod network is flat across namespaces
```

The pod network is a single flat network. Isolation is only enforced by NetworkPolicy, not by namespaces.

---

## Challenge Answers

**1. veth pair?**
A virtual ethernet (veth) pair is two connected virtual network interfaces — like a pipe. One end (`eth0`) lives inside the pod's network namespace. The other end (`vethXXXX`) lives in the host (node) network namespace. Packets sent into one end come out the other. The CNI plugin connects the host end to a bridge or routes it directly, linking the pod to the rest of the network.

**2. Overlay vs BGP routing?**
- Overlay (Flannel VXLAN, Weave): pod packets are wrapped (encapsulated) in UDP/VXLAN packets to cross node boundaries. The outer packet uses node IPs; the inner packet carries pod IPs. Overhead from encapsulation/decapsulation. Works on any network without special configuration.
- BGP (Calico): pod IPs are advertised as routes via BGP to all nodes and possibly the physical network. No encapsulation — the packet travels directly with pod source/dest IPs. Faster but requires the underlying network to support routing pod CIDRs.

**3. Why doesn't Flannel support NetworkPolicy?**
Flannel is deliberately simple — it only handles pod IP assignment and packet routing between nodes. It has no mechanism to inspect or filter individual packets based on labels or namespaces. NetworkPolicy enforcement requires kernel-level packet filtering (iptables rules or eBPF programs) keyed on pod metadata — that's what Calico, Cilium, and Weave add on top of networking.

**4. Pod CIDR division?**
The cluster is given a large CIDR (e.g. `10.244.0.0/16`). The controller manager divides it into per-node subnets (e.g. /24 each). Each node's kubelet tells the CNI plugin its assigned subnet, and the CNI assigns IPs from that range to pods scheduled on that node. Cross-node routing is set up so each node knows which subnet belongs to which node.

**5. Pod network CIDR vs service network CIDR?**
- Pod CIDR (`--cluster-cidr`): real IP addresses assigned to pods. Implemented and routed by the CNI plugin. Packets with pod IPs actually travel on the network.
- Service CIDR (`--service-cluster-ip-range`): virtual IP addresses assigned to Services (ClusterIPs). These are NOT routable — they only exist in iptables/IPVS rules managed by kube-proxy on each node. When a pod connects to a ClusterIP, iptables intercepts and rewrites the destination to a pod IP before the packet ever leaves the node.
