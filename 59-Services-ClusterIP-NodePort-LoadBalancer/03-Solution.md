# 59 — Services | Solutions

---

## Exercise 2 — ClusterIP service

```bash
kubectl get svc backend-clusterip
```
```
NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
backend-clusterip    ClusterIP   10.96.100.50    <none>        80/TCP    5s
```

```bash
kubectl get endpoints backend-clusterip
```
```
NAME                ENDPOINTS                                       AGE
backend-clusterip   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80      5s
```

3 pod IPs in the Endpoints list — one per replica.

---

## Exercise 3 — ClusterIP curl test

```
<title>Welcome to nginx!</title>
```

Reached nginx through the ClusterIP service.

---

## Exercise 4 — NodePort

```bash
kubectl get svc backend-nodeport
```
```
NAME               TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
backend-nodeport   NodePort   10.96.100.51    <none>        80:30080/TCP   5s
```

`80:30080/TCP` means: port 80 internally, exposed on 30080 on every node.

---

## Exercise 6 — Headless DNS

```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      backend-headless.svc-lab.svc.cluster.local
Address 1: 10.244.0.5   ← individual pod IPs, not a VIP
Address 2: 10.244.0.6
Address 3: 10.244.0.7
```

No ClusterIP — DNS returns all 3 pod IPs directly.

---

## Exercise 8 — Endpoint changes with scaling

```bash
# After scale down to 1:
kubectl get endpoints backend-clusterip
# ENDPOINTS: 10.244.0.5:80   ← only 1

# After scale back to 3:
# ENDPOINTS: 10.244.0.5:80,10.244.0.6:80,10.244.0.7:80
```

Endpoints update dynamically as pods come and go.

---

## Challenge Answers

**1. port vs targetPort vs nodePort?**
- `port`: The port the Service itself listens on — what clients use to connect to the Service.
- `targetPort`: The port on the **pod** that traffic is forwarded to — what your app is actually listening on.
- `nodePort`: The port opened on **every node's IP** for external access (NodePort/LoadBalancer only, range 30000–32767).

**2. How Service selects pods?**
Via `spec.selector` — the Service watches for pods with matching labels. The Endpoints controller automatically maintains the list of pod IPs. When a pod dies or a new one starts, the Endpoints object updates within seconds.

**3. Headless service and use case?**
`clusterIP: None` — no virtual IP is assigned. DNS returns individual pod IPs. Used by StatefulSets so each pod is individually addressable (`pod-0.svc`, `pod-1.svc`). Also used when clients need to do their own load balancing or need to connect to specific instances (databases, Kafka brokers).

**4. NodePort range?**
30000–32767 (default). Configurable on the API server with `--service-node-port-range`.

**5. What implements Service routing on each node?**
`kube-proxy` — runs as a DaemonSet on every node. It watches Services and Endpoints and programs iptables (or IPVS) rules to forward traffic. When you connect to a ClusterIP, iptables/IPVS on the local node intercepts and redirects the packet to a pod IP.
