# 70 — Installing Kubernetes with kubeadm

## Overview

`kubeadm` is the official tool for bootstrapping a minimum-viable, secure Kubernetes cluster in a repeatable way. This walks through a full installation: prerequisites, control plane init, CNI, and joining workers.

---

## Step 1 — Prerequisites (all nodes)

```bash
# Disable swap (required by kubelet)
sudo swapoff -a
sudo sed -i '/swap/d' /etc/fstab

# Enable required kernel modules
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Required sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

---

## Step 2 — Install a Container Runtime (containerd)

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable SystemdCgroup (required for kubelet compatibility)
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## Step 3 — Install kubeadm, kubelet, kubectl (all nodes)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 4 — Initialize the Control Plane (control plane node only)

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --control-plane-endpoint="LOAD_BALANCER_DNS:6443" \
  --upload-certs
```

> Use `--control-plane-endpoint` even for single control-plane setups — it makes adding HA control planes later possible without rebuilding.

Output includes:
```
kubeadm join LOAD_BALANCER_DNS:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>
```
**Save this command** — you need it to join worker nodes.

---

## Step 5 — Configure kubectl Access

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
# Should show the control plane node as NotReady (no CNI yet)
```

---

## Step 6 — Install a CNI Plugin

```bash
# Calico example
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Wait for nodes to become Ready
kubectl get nodes -w
```

---

## Step 7 — Join Worker Nodes

On each worker node, run the join command saved from Step 4:

```bash
sudo kubeadm join LOAD_BALANCER_DNS:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

If the token expired (default TTL 24h), generate a new one from the control plane:

```bash
kubeadm token create --print-join-command
```

---

## Step 8 — Verify the Cluster

```bash
kubectl get nodes
kubectl get pods -n kube-system
kubectl cluster-info
```

All nodes should show `Ready` and all kube-system pods `Running`.

---

## Adding Additional Control Plane Nodes (HA)

```bash
# Generate a new certificate key if --upload-certs key expired (>2h)
kubeadm init phase upload-certs --upload-certs

# On the new control-plane node:
sudo kubeadm join LOAD_BALANCER_DNS:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key>
```

---

## Common kubeadm Commands

```bash
kubeadm version
kubeadm config print init-defaults     # see all default config
kubeadm token list                     # active join tokens
kubeadm token create                   # generate a new token
kubeadm reset                          # tear down a node's kubeadm state
kubeadm certs check-expiration         # cert expiry check
```

---

## Common Installation Failures

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| kubeadm init hangs at "waiting for kubelet" | swap enabled, or cgroup driver mismatch | `swapoff -a`; match cgroup driver in containerd and kubelet |
| Nodes stuck `NotReady` | No CNI installed | Apply a CNI manifest |
| `kubeadm join` fails: token expired | Default 24h TTL passed | `kubeadm token create --print-join-command` |
| Pods stuck `Pending` on all nodes | Control plane taint, no schedulable nodes | Check `kubectl describe node`, untaint or add workers |
| `connection refused` on 6443 | API server not running / firewall | Check `crictl ps`, open port 6443 |
