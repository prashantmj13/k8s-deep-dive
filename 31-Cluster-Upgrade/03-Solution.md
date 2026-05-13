# 31 — Cluster Upgrade | Solutions & Expected Outputs

---

## Exercise 2 — kubeadm upgrade plan output

```
Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
COMPONENT   CURRENT        TARGET
kubelet     2x v1.28.0     v1.29.0

Upgrade to the latest stable version:
COMPONENT                 CURRENT    TARGET
kube-apiserver            v1.28.0    v1.29.0
kube-controller-manager   v1.28.0    v1.29.0
kube-scheduler            v1.28.0    v1.29.0
kube-proxy                v1.28.0    v1.29.0
CoreDNS                   v1.10.1    v1.11.1
etcd                      3.5.9-0    3.5.10-0
```

---

## Exercise 4 — Upgrade apply success

```
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.0". Enjoy!
[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```

---

## Exercise 5 — After upgrading control plane kubelet

```bash
kubectl get nodes
```
```
NAME           STATUS                     VERSION
controlplane   Ready                      v1.29.0   ← upgraded
node01         Ready                      v1.28.0   ← still old
```

---

## Exercise 6 — After upgrading node01

```bash
kubectl get nodes
```
```
NAME           STATUS   VERSION
controlplane   Ready    v1.29.0
node01         Ready    v1.29.0   ← now upgraded
```

---

## Challenge Answers

**1. Skip from 1.27 to 1.29?**
No. Kubernetes only supports upgrading **one minor version at a time**. You must go 1.27 → 1.28 → 1.29. Attempting to skip will fail with an error from kubeadm.

**2. apt-mark hold?**
`apt-mark hold kubeadm` prevents apt from automatically upgrading kubeadm when you run `apt upgrade`. Without this, a routine system upgrade could accidentally bump Kubernetes versions. Always hold Kubernetes packages after installing them.

**3. Upgrade order?**
The **API server must always be upgraded first**. The version skew policy allows other components to be one minor version behind the API server. Upgrading kubelet before API server would violate the skew policy.

**4. kubeadm upgrade node on worker?**
`kubeadm upgrade node` on a worker node:
- Downloads the new kubelet configuration from the control plane
- Updates the `/var/lib/kubelet/config.yaml` 
- Does NOT upgrade the kubelet binary itself (you do that with apt)

**5. kubectl version skew?**
kubectl can be at most **one minor version ahead or behind** the API server. So with API server at v1.29, kubectl can be v1.28, v1.29, or v1.30. This is the most permissive skew policy of all components.
