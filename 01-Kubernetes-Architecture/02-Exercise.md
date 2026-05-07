# Kubernetes Architecture — Practical Exercises

These exercises are 100% hands-on. You will use `kubectl` and a few node-level commands to actually **see** every component of the architecture you read about in `01-Theory.md`.

## Prerequisites

You need a running cluster. Pick the easiest one:

- **minikube** — `minikube start --driver=docker`
- **kind** — `kind create cluster --name learn`
- **k3d** — `k3d cluster create learn`
- A real cluster (EKS / GKE / AKS / on-prem) if you have access

Verify:
```bash
kubectl version --short
kubectl cluster-info
kubectl get nodes
```

If those three work, you are ready.

---

## Section A — Inspect the Cluster Itself

**Goal:** find the API Server URL, the cluster's Kubernetes version, and how many nodes there are.

### A1. Where is the API Server?
```bash
kubectl cluster-info
```
Identify which URL belongs to the API Server.

### A2. What version is running?
```bash
kubectl version -o yaml
```
Find both client and server versions.

### A3. List all nodes and their roles
```bash
kubectl get nodes -o wide
```
Note the columns: STATUS, ROLES, VERSION, INTERNAL-IP, OS-IMAGE, KERNEL-VERSION, CONTAINER-RUNTIME.

### A4. Look at your kubeconfig (how you authenticate)
```bash
kubectl config view
kubectl config current-context
kubectl config get-contexts
```
**Question to answer:** Where on your machine is the kubeconfig file? (Hint: `echo $KUBECONFIG` or default `~/.kube/config`.)

### A5. List every API resource type the cluster supports
```bash
kubectl api-resources
kubectl api-resources --namespaced=true | head
kubectl api-resources --namespaced=false
kubectl api-versions
```
**Question:** Which resources are namespaced and which are cluster-scoped? Find one example of each.

---

## Section B — Inspect the Control Plane

In most clusters (kubeadm-based, minikube, kind), the Control Plane components run as **static pods** in the `kube-system` namespace. Let's find them.

### B1. List all control plane pods
```bash
kubectl get pods -n kube-system
kubectl get pods -n kube-system -o wide
```
**Spot:** `kube-apiserver-*`, `etcd-*`, `kube-scheduler-*`, `kube-controller-manager-*`, `kube-proxy-*`, `coredns-*`.

### B2. Inspect the API Server pod
```bash
kubectl describe pod -n kube-system kube-apiserver-<your-node-name>
```
Find:
- The image being used
- The flags it was started with (`--advertise-address`, `--secure-port`, `--etcd-servers`)
- Which node it runs on

### B3. View the live API Server YAML
```bash
kubectl get pod -n kube-system kube-apiserver-<your-node-name> -o yaml | less
```

### B4. Inspect etcd
```bash
kubectl describe pod -n kube-system etcd-<your-node-name>
kubectl get pod -n kube-system etcd-<your-node-name> -o yaml | grep -A5 command
```
Find: which port etcd listens on, and where it stores data.

### B5. Inspect the Scheduler
```bash
kubectl describe pod -n kube-system kube-scheduler-<your-node-name>
kubectl logs -n kube-system kube-scheduler-<your-node-name> --tail=20
```

### B6. Inspect the Controller Manager
```bash
kubectl describe pod -n kube-system kube-controller-manager-<your-node-name>
kubectl logs -n kube-system kube-controller-manager-<your-node-name> --tail=20
```
**Tip:** every line in the log roughly corresponds to a controller doing reconciliation.

### B7. View the static pod manifest files (where the Control Plane comes from)
For minikube/kubeadm clusters, the static pod YAMLs live on the control-plane node at `/etc/kubernetes/manifests/`. Let's look:

```bash
# For minikube
minikube ssh
sudo ls -la /etc/kubernetes/manifests/
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml
exit
```

```bash
# For kind
docker exec -it learn-control-plane bash
ls -la /etc/kubernetes/manifests/
cat /etc/kubernetes/manifests/etcd.yaml
exit
```

**Question:** What happens if you delete the file `/etc/kubernetes/manifests/kube-scheduler.yaml`? (Don't actually do it on a shared cluster — just predict, then read the answer in the solution file.)

---

## Section C — Inspect Worker Nodes

### C1. Get full details about a node
```bash
kubectl get nodes
kubectl describe node <node-name>
```
In the output, find these sections and explain what each shows:
- **Conditions** (Ready, MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable)
- **Addresses** (InternalIP, Hostname)
- **Capacity** vs **Allocatable**
- **System Info** (OS, kernel, container runtime, kubelet version)
- **Non-terminated Pods**
- **Allocated resources**

### C2. Get node info as JSON / YAML
```bash
kubectl get node <node-name> -o yaml | less
kubectl get node <node-name> -o jsonpath='{.status.nodeInfo}'
kubectl get node <node-name> -o jsonpath='{.status.capacity}'
```

### C3. See all running pods on a specific node
```bash
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node-name>
```

### C4. Check the kubelet on the node itself
For minikube:
```bash
minikube ssh
sudo systemctl status kubelet
sudo journalctl -u kubelet --no-pager | tail -50
sudo cat /var/lib/kubelet/config.yaml | head -40
exit
```

For kind:
```bash
docker exec -it learn-control-plane systemctl status kubelet
docker exec -it learn-control-plane cat /var/lib/kubelet/config.yaml
```

**Question:** Find the line `staticPodPath` in the kubelet config. What value does it have? Why is it important?

### C5. Check the container runtime on the node
```bash
# Inside the node (minikube ssh / kind exec)
sudo crictl version
sudo crictl info | head -30
sudo crictl ps
sudo crictl pods
```

### C6. Check kube-proxy mode
```bash
kubectl logs -n kube-system <kube-proxy-pod-name> | grep -i "proxy mode"
kubectl get cm -n kube-system kube-proxy -o yaml | grep mode:
```
**Question:** Is your cluster using `iptables` mode or `IPVS` mode?

---

## Section D — Inspect Pods (the smallest unit)

### D1. Create your first pod and watch it land on a node
```bash
kubectl run myapp --image=nginx
kubectl get pods -o wide
```
**Observe:** which node did the Scheduler put it on?

### D2. Look at the full pod object
```bash
kubectl get pod myapp -o yaml | less
```
Find:
- `spec.nodeName` — set by the Scheduler
- `spec.containers[0].image`
- `status.phase`
- `status.podIP`
- `status.conditions`
- `status.containerStatuses[0].containerID`

### D3. Describe the pod (event log included)
```bash
kubectl describe pod myapp
```
Read the **Events** at the bottom — you can literally see Scheduler -> Pulled image -> Created container -> Started.

### D4. See the container running inside via crictl (on the node)
```bash
# minikube ssh / kind exec into the node where myapp runs
sudo crictl ps | grep myapp
sudo crictl inspect <container-id> | less
```

### D5. Open a shell inside the pod
```bash
kubectl exec -it myapp -- sh
# inside the pod:
hostname
ip addr        # see the pod's IP
ps -ef         # see processes
exit
```

### D6. Look at logs
```bash
kubectl logs myapp
kubectl logs myapp -f
```

### D7. Where is the kubelet getting the pod definition from?
```bash
kubectl get pod myapp -o yaml | grep -A2 "ownerReferences"
```
**Question:** A pod created with `kubectl run` has no owner. What happens if you delete this pod? (Try it!)
```bash
kubectl delete pod myapp
kubectl get pods
```
The pod is gone forever — no controller to bring it back.

---

## Section E — See the API Server's Job in Action

Watch the API Server stream changes in real time.

### E1. Watch all events as they happen
```bash
kubectl get events --all-namespaces --watch
```
Open a second terminal and create / delete a pod:
```bash
kubectl run hello --image=nginx
kubectl delete pod hello
```
Watch the events appear in the first terminal.

### E2. Use `kubectl --v=8` to see the raw API calls
```bash
kubectl --v=8 get pods 2>&1 | head -40
```
You will see lines like `GET https://<apiserver>:6443/api/v1/...` — these are the raw HTTP calls `kubectl` makes.

### E3. Hit the API Server directly with curl
```bash
# Get the API server endpoint
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl create token default 2>/dev/null || kubectl get secret -o jsonpath='{.items[0].data.token}' | base64 -d)

curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/api/v1/namespaces/default/pods
curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/version
curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/healthz
```
**Question:** what does `/healthz` return for a healthy API Server?

---

## Section F — See the Controller Manager / Self-Healing in Action

### F1. Create a Deployment (which uses a controller)
```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl get deploy,rs,pods -l app=web -o wide
```
Notice the chain: **Deployment -> ReplicaSet -> Pods** — each managed by a controller.

### F2. Kill a pod and watch the controller bring it back
```bash
kubectl get pods -l app=web
kubectl delete pod -l app=web --field-selector status.phase=Running --wait=false
kubectl get pods -l app=web -w
```
**Observation:** within seconds, the ReplicaSet controller creates new pods to replace the deleted ones. Press Ctrl+C to stop watching.

### F3. Try to "trick" the cluster — scale down by force
```bash
kubectl scale deploy web --replicas=5
kubectl get pods -l app=web -o wide
kubectl scale deploy web --replicas=1
kubectl get pods -l app=web -o wide
```

### F4. Drain a node and watch pods reschedule
```bash
kubectl get nodes
kubectl cordon <node-name>      # mark unschedulable
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide        # see pods moved
kubectl uncordon <node-name>
```
This shows the Scheduler picking new homes for evicted pods.

---

## Section G — See the Scheduler's Decision

### G1. Force a pod that cannot be scheduled
Create a pod that asks for an absurd amount of memory:
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hungry
spec:
  containers:
  - name: hungry
    image: nginx
    resources:
      requests:
        memory: "500Gi"
EOF

kubectl get pod hungry
kubectl describe pod hungry | tail -20
```
Read the events at the bottom — the Scheduler will say something like `0/1 nodes are available: 1 Insufficient memory.`

### G2. Cleanup
```bash
kubectl delete pod hungry
```

### G3. Use nodeSelector to hand-pick a node
```bash
kubectl label nodes <node-name> tier=frontend

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pinned
spec:
  nodeSelector:
    tier: frontend
  containers:
  - name: c
    image: nginx
EOF

kubectl get pod pinned -o wide
```

---

## Section H — See the Whole Cluster's Resource Use

### H1. Live resource usage (requires metrics-server)
```bash
kubectl top nodes
kubectl top pods -A
```
If you get an error, install metrics-server first:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
# For minikube/kind, you may need to add --kubelet-insecure-tls to the deployment.
```

### H2. Check overall cluster health
```bash
kubectl get componentstatuses    # deprecated but still works on many clusters
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

---

## Section I — The Final Architecture Map

Now that you have inspected every piece, run these commands one after another and explain in your own words **what each output reveals about the architecture**:

```bash
# 1. The cluster
kubectl cluster-info
kubectl get nodes -o wide

# 2. The control plane
kubectl get pods -n kube-system

# 3. The static manifests on the master
# (run via minikube ssh / kind exec)
sudo ls /etc/kubernetes/manifests/

# 4. A worker's perspective
sudo systemctl status kubelet
sudo crictl ps

# 5. A pod's perspective
kubectl run demo --image=nginx
kubectl describe pod demo
kubectl exec -it demo -- ip addr

# 6. Cleanup
kubectl delete pod demo
```

Write a short note (2-3 sentences) for each command describing **what it tells you about the Kubernetes architecture**. Compare your notes with the answer key in `03-Solution.md`.

---

## Section J — Stretch / Optional

1. Use `kubectl proxy` to expose the API Server locally and browse it with curl:
   ```bash
   kubectl proxy --port=8080 &
   curl http://localhost:8080/api/v1/namespaces/default/pods
   curl http://localhost:8080/healthz
   curl http://localhost:8080/api/v1/nodes
   ```

2. Use `kubectl get --raw` to fetch metrics from the kubelet directly:
   ```bash
   kubectl get --raw "/api/v1/nodes/<node-name>/proxy/stats/summary" | head
   ```

3. Use `etcdctl` (advanced) to read keys directly from etcd. Inside the etcd pod:
   ```bash
   kubectl exec -it -n kube-system etcd-<node-name> -- sh
   # inside the pod
   ETCDCTL_API=3 etcdctl \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key \
     get / --prefix --keys-only | head
   ```
   You will see how Kubernetes stores objects as keys like `/registry/pods/default/myapp`.

4. Bring down the API Server temporarily (only on a throwaway minikube/kind cluster!) and observe what still works and what breaks:
   ```bash
   # On the control plane node:
   sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
   # wait 30 seconds
   kubectl get pods    # this will fail
   docker ps           # the running pods are still there!
   sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
   ```

---

When you finish, open `03-Solution.md` for expected outputs and explanations.
