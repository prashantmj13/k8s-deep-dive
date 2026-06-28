# 80 — Troubleshooting Worker Node and Network | Exercises

> **Cluster:** Multi-node kubeadm cluster with node SSH access
> **Estimated time:** 35 minutes

---

## Exercise 1 — Inspect node conditions

```bash
kubectl get nodes
kubectl describe node <node> | grep -A15 "Conditions:"
kubectl get node <node> -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

---

## Exercise 2 — Simulate kubelet down (recoverable test)

```bash
# On a worker node
sudo systemctl stop kubelet
```

From control plane:
```bash
kubectl get nodes -w
# Wait and observe the node transition to NotReady after ~40s (node-monitor-grace-period)
```

```bash
# Restore
ssh <node> 'sudo systemctl start kubelet'
kubectl get nodes -w
```

---

## Exercise 3 — Observe pod eviction timing

```bash
kubectl create deployment evict-test --image=nginx --replicas=2
kubectl get pods -o wide | grep evict-test
# Note which node they're on, then stop kubelet on that node (as above)
# Wait 5+ minutes and watch:
kubectl get pods -o wide -w
```

> **Observe:** after `pod-eviction-timeout` (default 5 min), pods are evicted and rescheduled to healthy nodes.

---

## Exercise 4 — DNS resolution test

```bash
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 --rm -it --restart=Never -- nslookup kubernetes.default
```

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc -n kube-system kube-dns
```

---

## Exercise 5 — Endpoints diagnosis

```bash
kubectl create deployment web --image=nginx
kubectl expose deployment web --port=80

# Break it: scale to 0 and check endpoints
kubectl scale deployment web --replicas=0
kubectl get endpoints web
# Should be empty - no pods to back the service

kubectl scale deployment web --replicas=2
kubectl get endpoints web
```

---

## Exercise 6 — Pod IP vs Service connectivity test

```bash
POD_IP=$(kubectl get pods -l app=web -o jsonpath='{.items[0].status.podIP}')
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=3 http://$POD_IP
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=3 http://web
```

---

## Exercise 7 — Cross-node pod connectivity

```bash
kubectl get nodes
kubectl run net-a --image=busybox --overrides='{"spec":{"nodeName":"<node1>"}}' -- sleep 3600
kubectl run net-b --image=busybox --overrides='{"spec":{"nodeName":"<node2>"}}' -- sleep 3600
sleep 5
NET_B_IP=$(kubectl get pod net-b -o jsonpath='{.status.podIP}')
kubectl exec net-a -- ping -c3 $NET_B_IP
```

---

## Exercise 8 — NetworkPolicy blocking traffic (diagnosis)

```bash
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-test
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF

kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=3 http://web
# Should now fail

kubectl delete networkpolicy deny-all-test
kubectl run debug --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=3 http://web
# Should work again
```

---

## Exercise 9 — Cleanup

```bash
kubectl delete deployment web evict-test
kubectl delete svc web
kubectl delete pod net-a net-b 2>/dev/null; true
```

---

## Challenge Questions

1. What is `node-monitor-grace-period` and how does it relate to a node showing NotReady?
2. If pod-to-pod traffic fails only across nodes (same-node works fine), what component is most likely at fault?
3. Why would `kubectl get endpoints <svc>` returning empty be a normal, non-error state?
4. What's the practical difference between testing connectivity to a pod IP directly vs through its Service?
5. How would you confirm a NetworkPolicy — not something else — is the actual cause of a connectivity failure?

---

# Solutions

**Exercise 2:** Node shows `NotReady` after the grace period; `kubectl describe node` shows `KubeletNotReady` condition with a stale heartbeat. Restarting kubelet brings it back to `Ready` within seconds of the service restarting.

**Exercise 3:** Pods stay `Running` (still shown on the dead node) for up to 5 minutes, then transition through `Terminating` and get rescheduled as new pods on healthy nodes.

**Exercise 5:** With 0 replicas, `kubectl get endpoints web` shows `<none>` — completely normal, not a bug, since there are no pods to back the service.

**Exercise 8:** After applying `deny-all-test`, the wget to the service times out. After deleting it, connectivity is restored immediately — confirming the NetworkPolicy was the cause.

## Challenge Answers

**1. node-monitor-grace-period?**
A kube-controller-manager setting (default 40s) — if the controller manager doesn't receive a kubelet heartbeat within this window, it marks the node `NotReady`. This is distinct from `pod-eviction-timeout` (default 5 min), which is how long the control plane waits AFTER marking a node NotReady before it actually evicts and reschedules that node's pods elsewhere.

**2. Cross-node only failure?**
The CNI plugin's cross-node routing mechanism (VXLAN tunnel for Flannel, BGP routes for Calico, etc.) — same-node traffic only needs the local bridge/veth setup, which clearly works. Cross-node failures point at the CNI agent's inter-node data plane (check CNI pod logs, firewall rules between nodes on the overlay/BGP ports).

**3. Empty endpoints — normal state?**
Yes — an empty Endpoints/EndpointSlice simply means there are currently no Ready pods matching the Service's selector. This happens normally during a scale-to-zero, a rolling deployment's brief transition window, or before any matching pods have passed their readiness probe. It's only a problem if you EXPECT running pods and don't see them listed.

**4. Pod IP vs Service test — why both?**
Testing the pod IP directly isolates whether the APPLICATION/POD itself is reachable (bypassing kube-proxy/iptables entirely). Testing via the Service additionally exercises kube-proxy's DNAT rules and DNS resolution. If pod IP works but Service doesn't, the problem is in the Service layer (kube-proxy, endpoints, or DNS) — not the pod or CNI.

**5. Confirming NetworkPolicy is the cause?**
The cleanest method: temporarily delete (or use `--dry-run` to preview) the suspected NetworkPolicy and retest connectivity immediately. If connectivity is restored the instant the policy is removed, and breaks again when reapplied, that's a definitive confirmation — much faster than reading through complex selector logic, especially with multiple overlapping policies in a namespace.
