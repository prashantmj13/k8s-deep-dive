# 65 — IPAM and Weave | Exercises

> **Cluster:** kubeadm cluster (you can install Weave as the CNI)
> **Estimated time:** 20 minutes

---

## Exercise 1 — Inspect current pod CIDR allocation

```bash
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"  "}{.spec.podCIDR}{"\n"}{end}'
kubectl cluster-info dump | grep -m1 "cluster-cidr"
```

---

## Exercise 2 — Install Weave Net (if not already using a CNI)

```bash
kubectl apply -f "https://github.com/weaveworks/weave/releases/download/latest_release/weave-daemonset-k8s.yaml"
kubectl get pods -n kube-system | grep weave
kubectl rollout status daemonset/weave-net -n kube-system
```

---

## Exercise 3 — Check Weave status

```bash
WEAVE_POD=$(kubectl get pods -n kube-system -l name=weave-net -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n kube-system $WEAVE_POD -c weave -- /home/weave/weave --local status
kubectl exec -n kube-system $WEAVE_POD -c weave -- /home/weave/weave --local status ipam
```

---

## Exercise 4 — Create pods and observe IP allocation

```bash
kubectl create deployment ip-test --image=busybox --replicas=4 -- sleep 3600
kubectl get pods -o wide -l app=ip-test
```

> Note how IPs are allocated — are they sequential within a node, or scattered?

---

## Exercise 5 — Simulate IP exhaustion (conceptual)

```bash
# Check the node's pod CIDR size
kubectl get node $(kubectl get nodes -o jsonpath='{.items[0].metadata.name}') -o jsonpath='{.spec.podCIDR}'

# A /24 = 254 usable IPs. Check current pod count on that node
kubectl get pods -A -o wide | grep <node-name> | wc -l
```

---

## Exercise 6 — Cleanup

```bash
kubectl delete deployment ip-test
```

---

## Challenge Questions

1. What is the difference between host-local and cluster-wide IPAM?
2. Why does Weave use a gossip protocol instead of a fixed per-node CIDR?
3. What happens when a node's pod CIDR block runs out of available IPs?
4. Where is CNI IPAM configuration stored on a node?
5. What are the two operating modes of Weave Net and how do they differ?
