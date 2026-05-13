# 30 — Cluster Maintenance: OS Upgrade | Exercises

> **Cluster:** Multi-node cluster (kubeadm, kind with multiple nodes, or GKE)
> **Estimated time:** 20 minutes
> **Namespace:** `maintenance-lab`

---

## Exercise 1 — Setup: deploy a workload across nodes

```bash
kubectl create namespace maintenance-lab
kubectl config set-context --current --namespace=maintenance-lab

kubectl create deployment nginx --image=nginx --replicas=4
kubectl get pods -o wide
```

> Note which nodes the pods are running on.

---

## Exercise 2 — Cordon a node

Pick a worker node from the output above:

```bash
kubectl cordon <node-name>
kubectl get nodes
```

Now try to add more replicas:

```bash
kubectl scale deployment nginx --replicas=6
kubectl get pods -o wide
```

> **Observe:** New pods land only on non-cordoned nodes.

---

## Exercise 3 — Drain the node

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide
```

> **Observe:** Pods from the drained node are rescheduled on other nodes.

---

## Exercise 4 — Simulate OS maintenance (just wait or echo)

```bash
echo "Simulating OS upgrade on <node-name>..."
sleep 10
echo "Node rebooted and ready"
```

In a real scenario you would SSH in, upgrade, and reboot.

---

## Exercise 5 — Uncordon the node

```bash
kubectl uncordon <node-name>
kubectl get nodes
kubectl get pods -o wide
```

> **Observe:** Node is back as `Ready`. New pods can now schedule on it (existing pods don't move back automatically).

---

## Exercise 6 — Test drain with a bare pod (no controller)

```bash
kubectl run bare-pod --image=nginx --restart=Never
kubectl get pod bare-pod -o wide

kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

> **Observe:** Drain fails with a message about the bare pod. Use `--force` to proceed:

```bash
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
kubectl get pods
```

> **Observe:** `bare-pod` is gone — not rescheduled because it has no controller.

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace maintenance-lab
kubectl uncordon <node-name>
```

---

## Challenge Questions

1. What is the difference between `cordon` and `drain`?
2. What happens if a node comes back online within 5 minutes after going offline?
3. Why are DaemonSet pods ignored during drain?
4. What is a PodDisruptionBudget and how does it affect drain?
5. After uncordoning, do pods automatically move back to the recovered node?
