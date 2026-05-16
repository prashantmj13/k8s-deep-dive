# 63 — Pod Networking and CNI | Exercises

> **Cluster:** Any kubectl-accessible cluster (node access needed for some steps)
> **Estimated time:** 25 minutes

---

## Exercise 1 — Identify your CNI plugin

```bash
# Method 1: Check kube-system pods
kubectl get pods -n kube-system | grep -E "calico|flannel|cilium|weave|canal|antrea"

# Method 2: Check CNI config on a node (requires node access)
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf* 2>/dev/null || cat /etc/cni/net.d/*.conflist 2>/dev/null

# Method 3: Check CNI binaries
ls /opt/cni/bin/
```

---

## Exercise 2 — Inspect pod CIDRs

```bash
# Node pod CIDRs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# Cluster-wide pod CIDR
kubectl cluster-info dump | grep -m1 cluster-cidr

# Service CIDR
kubectl cluster-info dump | grep -m1 service-cluster-ip-range
```

---

## Exercise 3 — Pod IP assignment and networking

```bash
# Create two pods on different nodes (if possible)
kubectl create namespace cni-lab
kubectl config set-context --current --namespace=cni-lab

kubectl run pod-a --image=nginx
kubectl run pod-b --image=nginx

kubectl get pods -o wide    # note IPs and nodes

# Get individual IPs
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_B_IP=$(kubectl get pod pod-b -o jsonpath='{.status.podIP}')
echo "Pod A: $POD_A_IP"
echo "Pod B: $POD_B_IP"
```

---

## Exercise 4 — Pod-to-pod connectivity

```bash
# Pod A can reach Pod B directly by IP
kubectl exec pod-a -- curl -s --max-time 3 http://$POD_B_IP
kubectl exec pod-b -- curl -s --max-time 3 http://$POD_A_IP

# Check network interfaces inside a pod
kubectl exec pod-a -- ip addr
kubectl exec pod-a -- ip route

# Check DNS config
kubectl exec pod-a -- cat /etc/resolv.conf
```

---

## Exercise 5 — Inspect the node's network (if you have node access)

```bash
# SSH to a node or exec into a node debug pod
kubectl debug node/<node-name> -it --image=busybox

# Inside the debug pod:
ip link show type veth      # see veth pairs for pods
ip route                     # routing table
```

---

## Exercise 6 — Verify pod network is flat

```bash
# Pod A should reach pod B even across namespaces
kubectl run cross-ns-pod --image=busybox -n default --restart=Never -- sleep 3600

CROSS_IP=$(kubectl get pod cross-ns-pod -n default -o jsonpath='{.status.podIP}')

# From cni-lab namespace pod
kubectl exec pod-a -- curl -s --max-time 3 http://$CROSS_IP   # direct IP still works
```

---

## Exercise 7 — Check kube-proxy mode

```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode
# mode: "" means iptables (default)
# mode: "ipvs" means IPVS mode

# View iptables rules for a service
kubectl get svc -n cni-lab
# Then on a node:
# iptables -L -n -t nat | grep <service-ClusterIP>
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace cni-lab
kubectl delete pod cross-ns-pod -n default 2>/dev/null; true
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is a veth pair and how does it connect a pod to the node?
2. What is the difference between an overlay network and BGP routing for pod communication?
3. Why does Flannel NOT support NetworkPolicy?
4. What is the pod CIDR and how is it divided across nodes?
5. What is the difference between pod network CIDR and service network CIDR?
