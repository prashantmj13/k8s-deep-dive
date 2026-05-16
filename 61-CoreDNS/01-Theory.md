# 61 — CoreDNS

## What is CoreDNS?

CoreDNS is the **DNS server for Kubernetes** — it resolves service names to ClusterIPs and pod names to pod IPs inside the cluster. It replaced kube-dns in Kubernetes 1.11+.

Every pod in the cluster has CoreDNS set as its DNS resolver (`/etc/resolv.conf`).

---

## How DNS Works in Kubernetes

```
Pod wants to connect to "backend-svc"
         ↓
Pod checks /etc/resolv.conf:
  nameserver 10.96.0.10       ← CoreDNS ClusterIP
  search default.svc.cluster.local svc.cluster.local cluster.local
         ↓
Pod queries CoreDNS: "backend-svc"
         ↓
CoreDNS adds search domain: "backend-svc.default.svc.cluster.local"
         ↓
CoreDNS returns: 10.96.100.50 (the Service ClusterIP)
         ↓
Pod connects to 10.96.100.50
```

---

## DNS Name Format for Services

```
<service-name>.<namespace>.svc.<cluster-domain>

Examples:
backend-svc.default.svc.cluster.local
mysql.production.svc.cluster.local
redis.cache.svc.cluster.local
```

**Short names that work from within the same namespace:**
```
backend-svc                           # works in same namespace
backend-svc.default                   # works from any namespace
backend-svc.default.svc               # works from any namespace
backend-svc.default.svc.cluster.local # always works
```

---

## DNS Name Format for Pods

Pod DNS is disabled by default. Enable with `subdomain` + headless service:

```yaml
# Pod spec
spec:
  hostname: my-pod
  subdomain: my-service      # must match a headless service name
```

Full DNS: `my-pod.my-service.default.svc.cluster.local`

For StatefulSets, pods are automatically named:
```
pod-0.my-statefulset-svc.default.svc.cluster.local
pod-1.my-statefulset-svc.default.svc.cluster.local
```

---

## CoreDNS Configuration (Corefile)

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

Default Corefile:
```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {
       max_concurrent 1000
    }
    cache 30
    loop
    reload
    loadbalance
}
```

Key plugins:
- `kubernetes`: handles `.cluster.local` queries (in-cluster DNS)
- `forward`: forwards external queries to upstream resolvers
- `cache`: caches DNS responses (30s default)
- `health`: health check endpoint
- `prometheus`: metrics endpoint

---

## Customising CoreDNS

### Add a stub zone (resolve custom domain internally)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        kubernetes cluster.local ...
        forward . /etc/resolv.conf
    }
    mycompany.internal:53 {
        forward . 10.10.0.1    # internal DNS server
    }
```

### Rewrite a DNS name

```
rewrite name backend-svc.default.svc.cluster.local new-backend-svc.default.svc.cluster.local
```

---

## Inspecting DNS from a Pod

```bash
# Check pod's DNS config
kubectl exec mypod -- cat /etc/resolv.conf

# Test DNS resolution
kubectl exec mypod -- nslookup backend-svc
kubectl exec mypod -- nslookup backend-svc.default.svc.cluster.local

# Test external DNS
kubectl exec mypod -- nslookup google.com

# Using dig
kubectl run dns-debug --image=tutum/dnsutils --restart=Never --rm -it \
  -- dig backend-svc.default.svc.cluster.local
```

---

## CoreDNS Pods and Service

```bash
# CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# CoreDNS service (ClusterIP 10.96.0.10 by default)
kubectl get svc -n kube-system kube-dns

# View logs
kubectl logs -n kube-system -l k8s-app=kube-dns
```

---

## DNS Policy for Pods

```yaml
spec:
  dnsPolicy: ClusterFirst    # default: use CoreDNS
  # Options:
  # ClusterFirst      — CoreDNS first, then upstream (default)
  # ClusterFirstWithHostNet — for hostNetwork pods
  # Default           — use node's DNS (not CoreDNS)
  # None              — custom dnsConfig only
  dnsConfig:
    nameservers:
    - 8.8.8.8
    searches:
    - mycompany.internal
    options:
    - name: ndots
      value: "5"
```

---

## Common DNS Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `nslookup` timeout | CoreDNS pod down | `kubectl rollout restart deploy/coredns -n kube-system` |
| Service not resolved | Wrong namespace in query | Use FQDN: `svc.namespace.svc.cluster.local` |
| External DNS fails | `forward` misconfigured | Check Corefile `forward` plugin |
| Slow DNS | `ndots` too high (many failed lookups) | Set `ndots: 2` in dnsConfig |
