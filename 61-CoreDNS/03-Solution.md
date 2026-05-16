# 61 — CoreDNS | Solutions

---

## Exercise 2 — Pod resolv.conf

```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

`10.96.0.10` is the CoreDNS ClusterIP. The `search` domains are appended when a short name is looked up.

---

## Exercise 3 — DNS resolution forms

```bash
kubectl exec dns-pod -- nslookup nginx-svc
# Server: 10.96.0.10
# Address: 10.96.0.10#53
# Name: nginx-svc.default.svc.cluster.local
# Address: 10.96.xxx.xxx    ← ClusterIP

kubectl exec dns-pod -- nslookup nginx-svc.default
# Same result — search domains fill in the rest

kubectl exec dns-pod -- nslookup nginx-svc.default.svc.cluster.local
# Same result — explicit FQDN
```

All three resolve to the same ClusterIP.

---

## Exercise 4 — Cross-namespace DNS

```bash
kubectl exec dns-pod -- nslookup httpd-svc
# ** server can't find httpd-svc.default.svc.cluster.local: NXDOMAIN
# (Service is in dns-test namespace, not default)

kubectl exec dns-pod -- nslookup httpd-svc.dns-test
# Name: httpd-svc.dns-test.svc.cluster.local
# Address: 10.96.xxx.xxx    ← works!
```

---

## Exercise 7 — Custom DNS config

```
nameserver 8.8.8.8
nameserver 8.8.4.4
search mycompany.internal
options ndots:2
```

CoreDNS is NOT used — queries go directly to Google DNS.

---

## Challenge Answers

**1. Full DNS name for redis in cache namespace?**
`redis.cache.svc.cluster.local`

**2. Why does a short name work in the same namespace?**
The pod's `/etc/resolv.conf` contains `search default.svc.cluster.local ...`. When the resolver tries `nginx-svc`, it appends each search domain in order. `nginx-svc.default.svc.cluster.local` resolves successfully, so it returns that answer.

**3. What does the search field do?**
It defines a list of domain suffixes that are automatically appended when a non-FQDN (no trailing dot, fewer dots than `ndots`) is looked up. Avoids typing the full qualified name for every query.

**4. ndots and slow DNS?**
`ndots:5` means: if the query has fewer than 5 dots, try all search domains first before treating it as absolute. Querying `google.com` (1 dot) triggers 3 failed lookups (`google.com.default.svc.cluster.local`, `google.com.svc.cluster.local`, `google.com.cluster.local`) before resolving the real `google.com`. Setting `ndots:2` reduces these wasted lookups and speeds up external DNS resolution.

**5. Forward mycompany.internal to internal DNS?**
Edit the CoreDNS ConfigMap to add a stub zone:
```
kubectl edit configmap coredns -n kube-system
```
Add:
```
mycompany.internal:53 {
    forward . 10.10.0.1    # your internal DNS server
}
```
Then restart CoreDNS:
```bash
kubectl rollout restart deployment coredns -n kube-system
```
