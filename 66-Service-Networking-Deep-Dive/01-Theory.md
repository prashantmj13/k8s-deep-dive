# 66 — Service Networking Deep Dive

## How kube-proxy Implements Services

Every Service gets a virtual ClusterIP that doesn't correspond to any real network interface. `kube-proxy` on every node watches Services/Endpoints and programs the **data plane** to make that virtual IP work.

---

## kube-proxy Modes

| Mode | Mechanism | Performance |
|------|-----------|-------------|
| `iptables` (default) | DNAT rules in netfilter | O(n) rule eval per packet, slows with many services |
| `ipvs` | Linux IP Virtual Server (hash tables) | O(1) lookup, better at scale |
| `userspace` | proxy process forwards traffic | deprecated, slow |
| `nftables` (newer) | nftables ruleset | replacement for iptables long-term |

```bash
# Check current mode
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
```

---

## iptables Mode Walkthrough

```
Packet to Service ClusterIP:Port
        ↓
KUBE-SERVICES chain (matches service IP:port)
        ↓
KUBE-SVC-XXXX chain (per-service, probabilistic load balancing)
        ↓
KUBE-SEP-XXXX chain (per-endpoint, DNAT to pod IP:port)
        ↓
Pod IP:Port
```

```bash
# View the rules (on a node)
iptables -t nat -L KUBE-SERVICES -n | head -20
iptables -t nat -L KUBE-SVC-<hash> -n
```

Each endpoint gets equal probability via `--probability` matches — this is how iptables "load balances" (statistically, not round robin).

---

## IPVS Mode

```bash
# Switch kube-proxy to IPVS (edit configmap then restart kube-proxy pods)
kubectl edit configmap kube-proxy -n kube-system
# set mode: "ipvs"
kubectl rollout restart daemonset kube-proxy -n kube-system

# View IPVS rules
ipvsadm -Ln
```

IPVS supports real load-balancing algorithms: round-robin, least connection, source hashing — configurable via `ipvs.scheduler` in the kube-proxy config.

---

## Endpoints and EndpointSlices

A Service's backing pods are tracked via `Endpoints` (legacy) or `EndpointSlices` (modern, scalable):

```bash
kubectl get endpoints my-svc
kubectl get endpointslices -l kubernetes.io/service-name=my-svc
```

EndpointSlices shard large endpoint lists (>100) into multiple objects to reduce API server load on updates — critical for Services backing thousands of pods.

---

## Session Affinity

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800
```

Routes requests from the same client IP to the same pod for the session duration — useful for stateful protocols without proper session storage.

---

## externalTrafficPolicy

Controls whether NodePort/LoadBalancer traffic is routed to a pod on ANY node, or only the **local** node:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # preserves client source IP, avoids extra hop
  # vs: Cluster (default) — load balances across all nodes, but SNATs source IP
```

| Policy | Source IP preserved? | Even load balancing? |
|--------|----------------------|----------------------|
| `Cluster` (default) | ❌ No (SNAT) | ✅ Yes |
| `Local` | ✅ Yes | ❌ No (only nodes with a pod get traffic) |

---

## Service Topology / Topology Aware Routing

```yaml
spec:
  trafficDistribution: PreferClose   # prefer same-zone endpoints (replaces old topologyKeys)
```

Reduces cross-zone traffic costs and latency in multi-AZ clusters.

---

## DNS-based Service Discovery Recap

```
<service-name>.<namespace>.svc.cluster.local → ClusterIP
my-svc.default.svc.cluster.local             → 10.96.x.x
```

Headless services (`clusterIP: None`) instead return **all pod IPs** directly via DNS A records — used heavily by StatefulSets for per-pod addressing:
```
pod-0.my-headless-svc.default.svc.cluster.local → pod-0's IP directly
```

---

## Debugging Service Connectivity

```bash
# Does the service have endpoints?
kubectl get endpoints my-svc

# Is kube-proxy running and healthy on the node?
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check iptables rules exist for the service
iptables -t nat -L | grep my-svc

# Test from inside the cluster
kubectl run debug --image=busybox --rm -it -- wget -qO- http://my-svc
```
