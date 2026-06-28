# 66 — Service Networking Deep Dive | Exercises

> **Cluster:** kubeadm cluster with root/sudo access to nodes
> **Estimated time:** 25 minutes

---

## Exercise 1 — Check kube-proxy mode

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep -A2 "mode:"
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

---

## Exercise 2 — Inspect iptables rules for a service

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80

kubectl get svc web
# On the node (or via kubectl debug node):
sudo iptables -t nat -L KUBE-SERVICES -n | grep web
sudo iptables -t nat -L -n | grep -A5 "KUBE-SVC"
```

---

## Exercise 3 — EndpointSlices

```bash
kubectl get endpoints web
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl describe endpointslice -l kubernetes.io/service-name=web
```

---

## Exercise 4 — externalTrafficPolicy comparison

```yaml
# svc-local.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-local
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f svc-local.yaml
kubectl get svc web-local
# Test from a node WITHOUT a web pod — should fail/timeout
# Test from a node WITH a web pod — should succeed
```

---

## Exercise 5 — Session affinity

```bash
kubectl patch svc web -p '{"spec":{"sessionAffinity":"ClientIP"}}'
kubectl get svc web -o jsonpath='{.spec.sessionAffinity}'
```

---

## Exercise 6 — Cleanup

```bash
kubectl delete deployment web
kubectl delete svc web web-local
```

---

## Challenge Questions

1. Why does IPVS scale better than iptables mode for clusters with many services?
2. What problem do EndpointSlices solve that the legacy Endpoints object had?
3. What is the tradeoff of `externalTrafficPolicy: Local`?
4. How does a headless service's DNS response differ from a normal ClusterIP service?
5. Why doesn't `iptables` mode do true round-robin load balancing?

---

# Solutions

**Exercise 2 output (abbreviated):**
```
KUBE-SVC-XXXX  tcp -- 0.0.0.0/0  10.96.5.20  /* default/web */ tcp dpt:80
  -> KUBE-SEP-AAA  (probability 0.33333)
  -> KUBE-SEP-BBB  (probability 0.50000)
  -> KUBE-SEP-CCC  (probability 1.00000)
```

**Exercise 3 output:**
```
NAME       ADDRESSTYPE   PORTS   ENDPOINTS
web-xk2p9  IPv4          80      10.244.0.5,10.244.1.3,10.244.1.4
```

---

## Challenge Answers

**1. IPVS vs iptables scaling?**
iptables rules are evaluated sequentially (linked list) — with thousands of services, packet processing time grows linearly. IPVS uses hash tables for O(1) lookup regardless of service count, and supports real LB algorithms (round robin, least conn) rather than probability-based chaining.

**2. EndpointSlices problem solved?**
A single Endpoints object listing all pods for a service became a bottleneck for services with hundreds/thousands of pods — any pod change rewrote the entire object, causing massive API server/etcd update traffic. EndpointSlices shard endpoints into groups of ~100, so only the affected slice updates.

**3. externalTrafficPolicy: Local tradeoff?**
Preserves the original client source IP (no SNAT) and avoids an extra network hop, but only nodes that actually have a matching pod will accept traffic — if your LB sends traffic to a node with no local pod, the connection fails. Requires the LB to do health checks per-node to route correctly.

**4. Headless service DNS?**
Normal Service: DNS returns ONE ClusterIP (a virtual IP) regardless of how many pods exist. Headless Service (`clusterIP: None`): DNS returns the IPs of ALL backing pods directly — clients connect straight to a pod IP. Used by StatefulSets for stable per-pod DNS names (`pod-0.svc...`).

**5. Why not true round robin in iptables?**
iptables rules use the `statistic` module's `--mode random --probability` matching — each rule independently has a probability of matching. This approximates even distribution statistically over many requests but isn't a strict round-robin sequence, and can be uneven with few requests or few endpoints.
