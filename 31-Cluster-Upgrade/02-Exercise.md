# 31 — Cluster Upgrade | Exercises

> **Cluster:** kubeadm-based cluster (use killercoda.com or a local kubeadm VM)
> **Estimated time:** 30–40 minutes

---

## Exercise 1 — Check current versions

```bash
kubectl get nodes
kubectl version --short
kubeadm version
```

Note down the current version of each component.

---

## Exercise 2 — Check available upgrade versions

```bash
# On the control plane node
kubeadm upgrade plan
```

This shows:
- Current version
- Latest stable version
- Components that will be upgraded
- Required kubeadm version

---

## Exercise 3 — Upgrade kubeadm on control plane

```bash
# Check available kubeadm versions
apt-cache madison kubeadm | head -5

# Unhold and upgrade (replace version as needed)
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# Verify
kubeadm version
```

---

## Exercise 4 — Apply the cluster upgrade

```bash
kubeadm upgrade apply v1.29.0
```

> This upgrades: kube-apiserver, kube-controller-manager, kube-scheduler, kube-proxy, CoreDNS, etcd.

Confirm with `y` when prompted. Watch for `SUCCESS! Your cluster was upgraded to "v1.29.0"`.

---

## Exercise 5 — Upgrade kubelet and kubectl on control plane

```bash
kubectl drain controlplane --ignore-daemonsets

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon controlplane
kubectl get nodes
```

---

## Exercise 6 — Upgrade a worker node

SSH into node01 (or use a second terminal):

```bash
# On node01
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm
kubeadm upgrade node
```

From control plane:

```bash
kubectl drain node01 --ignore-daemonsets --delete-emptydir-data
```

Back on node01:

```bash
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet
```

From control plane:

```bash
kubectl uncordon node01
kubectl get nodes
```

---

## Exercise 7 — Verify all nodes upgraded

```bash
kubectl get nodes
```

All nodes should show `v1.29.0` in the VERSION column.

---

## Challenge Questions

1. Can you upgrade from v1.27 directly to v1.29?
2. What does `apt-mark hold kubeadm` do and why is it important?
3. Which component must be upgraded first — API server or kubelet?
4. What does `kubeadm upgrade node` do on a worker node?
5. What is the version skew policy between kubectl and kube-apiserver?
