# 65 — IPAM and Weave | Solutions

---

## Exercise 1
```
controlplane  10.244.0.0/24
node01        10.244.1.0/24
```

## Exercise 3
```
Status: ready
Protocol: weave 1.2
Peers: 2 (with 2 established connections)
IPAM: 256 IPs (00.0%) free
```

## Exercise 4
```
NAME                       READY   STATUS    IP             NODE
ip-test-xxx-aaa            1/1     Running   10.32.0.2      node01
ip-test-xxx-bbb            1/1     Running   10.32.0.3      node01
ip-test-xxx-ccc            1/1     Running   10.32.0.4      controlplane
ip-test-xxx-ddd            1/1     Running   10.32.0.5      controlplane
```
With Weave, IPs come from one shared cluster-wide pool — not strictly sequential per node.

---

## Challenge Answers

**1. Host-local vs cluster-wide IPAM?**
Host-local pre-assigns a fixed CIDR block (e.g. /24) to each node; the node's local IPAM hands out IPs from that block only. Cluster-wide IPAM (Weave, Calico-ipam in cross-subnet mode) shares one pool across all nodes — nodes dynamically claim and release smaller chunks as needed, balancing usage better but requiring coordination (gossip or central API).

**2. Why gossip protocol?**
Fixed per-node CIDRs waste IPs if pod density is uneven across nodes (one busy node runs out while another sits empty). Weave's gossip protocol lets nodes dynamically negotiate and reclaim IP ranges from each other without a central coordinator, adapting to actual usage and surviving network partitions.

**3. Node's pod CIDR exhausted?**
New pods scheduled to that node get stuck in `ContainerCreating` with a CNI error like `failed to allocate IP address: no IP addresses available in range`. Existing pods are unaffected, but no new pods can start there until IPs are freed (pods deleted) or the CIDR is enlarged (often impossible without rebuilding the node).

**4. CNI IPAM config location?**
`/etc/cni/net.d/*.conf` (or `.conflist`) on each node — contains the `ipam` section specifying type, subnet, range. The CNI binary at `/opt/cni/bin/` reads this when invoked by kubelet.

**5. Weave's two modes?**
- **Fast datapath**: uses the kernel's Open vSwitch datapath for near-native performance, used when nodes can talk directly via UDP.
- **Sleeve mode**: a fallback in userspace, slightly slower but supports encryption and works through NAT/firewalls where fast datapath can't establish a direct connection.
