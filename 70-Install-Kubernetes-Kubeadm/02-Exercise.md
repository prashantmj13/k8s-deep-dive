# 70 — Install Kubernetes with kubeadm | Exercises

> **Environment:** 2+ Ubuntu VMs (1 control plane, 1+ workers) — or use killercoda.com / play-with-k8s for a free sandbox
> **Estimated time:** 45 minutes

---

## Exercise 1 — Prepare all nodes

On EVERY node (control plane + workers), run:
```bash
sudo swapoff -a
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

---

## Exercise 2 — Install containerd on every node

```bash
sudo apt-get update && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

---

## Exercise 3 — Install kubeadm/kubelet/kubectl on every node

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Exercise 4 — Initialize control plane

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

> Save the printed `kubeadm join` command for Exercise 6.

---

## Exercise 5 — Install Calico CNI

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
kubectl get nodes -w
```

---

## Exercise 6 — Join a worker node

On the worker node, run the saved join command:
```bash
sudo kubeadm join <control-plane-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Back on the control plane:
```bash
kubectl get nodes
```

---

## Exercise 7 — Verify cluster health

```bash
kubectl get pods -A
kubectl get componentstatuses 2>/dev/null || kubectl get --raw='/healthz'
kubectl run test --image=nginx
kubectl get pod test -o wide
kubectl delete pod test
```

---

## Exercise 8 — Generate a fresh join token (simulate expiry)

```bash
kubeadm token list
kubeadm token create --print-join-command
```

---

## Challenge Questions

1. Why must swap be disabled before installing kubelet?
2. What happens if the cgroup driver differs between containerd and kubelet?
3. Why does the control plane node show NotReady before installing a CNI?
4. What is the default TTL of a kubeadm bootstrap token?
5. What flag would you add at init time to support adding more control-plane nodes later?

---

# Solutions

**Exercise 4 output:**
```
Your Kubernetes control-plane has initialized successfully!
...
kubeadm join 192.168.1.10:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:1234...
```

**Exercise 6 output:**
```
NAME           STATUS   ROLES           AGE   VERSION
controlplane   Ready    control-plane   10m   v1.30.0
worker01       Ready    <none>          1m    v1.30.0
```

---

## Challenge Answers

**1. Why disable swap?**
kubelet's resource management (especially memory-based eviction and QoS guarantees) assumes no swap. With swap enabled, a pod could exceed its memory limit and swap rather than being OOM-killed predictably, breaking Kubernetes' scheduling and eviction guarantees. kubeadm refuses to proceed if swap is on (unless explicitly told to tolerate it).

**2. Cgroup driver mismatch?**
kubelet and the container runtime must agree on whether to use `systemd` or `cgroupfs` for cgroup management. A mismatch causes kubelet to fail starting containers or report inconsistent resource stats — typically surfacing as pods stuck in `ContainerCreating` with cgroup-related errors in kubelet logs.

**3. Control plane NotReady before CNI?**
The node's `Ready` condition requires the kubelet to confirm pod networking is functional. Without a CNI plugin installed, kubelet can't set up the pod network, so `NetworkReady` (and thus overall `Ready`) stays false. Core DNS and other pods also stay `Pending` until a CNI provides IPs.

**4. Default bootstrap token TTL?**
24 hours. After expiry, `kubeadm join` with that token fails with an authentication error. Generate a fresh one anytime with `kubeadm token create --print-join-command`.

**5. Flag for future HA expansion?**
`--control-plane-endpoint=<load-balancer-dns-or-ip>:6443` at init time — this sets a stable endpoint name that all components and future control-plane joins use, decoupling them from any single node's IP. Without it, retrofitting HA later requires significant reconfiguration.
