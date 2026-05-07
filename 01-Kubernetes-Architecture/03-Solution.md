# Kubernetes Architecture — Solutions (Expected Outputs)

For every exercise in `02-Exercise.md`, this file shows you the **expected output**, **what to look for**, and **what it teaches you about the architecture**. Outputs may vary slightly by cluster type and version.

---

## Section A — Cluster Inspection

### A1. `kubectl cluster-info`
```
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```
**What it tells you:** the API Server URL (here `192.168.49.2:8443`). All your `kubectl` traffic goes there. CoreDNS is the in-cluster DNS — also exposed via the API Server proxy.

### A2. `kubectl version -o yaml`
```yaml
clientVersion:
  gitVersion: v1.30.0
serverVersion:
  gitVersion: v1.30.0
```
**Lesson:** the cluster has two halves — the client (`kubectl` binary) and the server (`kube-apiserver`). They can be different versions but should stay within ±1 minor.

### A3. `kubectl get nodes -o wide`
```
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
minikube   Ready    control-plane   1d    v1.30.0   192.168.49.2   Ubuntu 22.04.3 LTS   5.15.0-..        containerd://1.7.x
```
**Lesson:** confirms the node is `Ready`, its role (`control-plane` for single-node setups, `<none>` or `worker` for workers), and the **container runtime** — important: it is NOT Docker; it is `containerd`.

### A4. `kubectl config view` and contexts
```
contexts:
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
```
The kubeconfig file is at `~/.kube/config` by default. It holds clusters, users, certs, and tokens — **the key to the API Server's front door**.

### A5. `kubectl api-resources`
- **Namespaced** examples: `pods`, `deployments`, `services`, `configmaps`.
- **Cluster-scoped** examples: `nodes`, `namespaces`, `persistentvolumes`, `clusterroles`.
**Lesson:** namespaced resources are scoped to a namespace boundary; cluster-scoped resources are visible everywhere.

---

## Section B — Control Plane

### B1. `kubectl get pods -n kube-system`
```
NAME                               READY   STATUS    RESTARTS   AGE
coredns-7db6d8ff4d-xyz             1/1     Running   0          1d
etcd-minikube                      1/1     Running   0          1d
kube-apiserver-minikube            1/1     Running   0          1d
kube-controller-manager-minikube   1/1     Running   0          1d
kube-proxy-abcde                   1/1     Running   0          1d
kube-scheduler-minikube            1/1     Running   0          1d
storage-provisioner                1/1     Running   0          1d
```
**Lesson:** the Control Plane is itself made of pods — ordinary pods, but special ones. Their suffix is the node name (e.g., `-minikube`) because they are **static pods** managed directly by the kubelet, not by a controller.

### B2. `kubectl describe pod kube-apiserver-...`
Look for in the description:
- `Image: registry.k8s.io/kube-apiserver:v1.30.0`
- `Command:` lists flags such as:
  - `--advertise-address=192.168.49.2`
  - `--secure-port=6443`
  - `--etcd-servers=https://127.0.0.1:2379`
  - `--service-cluster-ip-range=10.96.0.0/12`
- `Node: minikube/192.168.49.2`

**Lesson:** the API Server is a binary started with flags. The flags configure auth, etcd connection, network ranges, and ports.

### B3. The full YAML of kube-apiserver shows `kind: Pod` (not Deployment, not DaemonSet) — confirming static pod.

### B4. etcd
```
Command:
  etcd
  --advertise-client-urls=https://192.168.49.2:2379
  --listen-client-urls=https://127.0.0.1:2379,https://192.168.49.2:2379
  --data-dir=/var/lib/etcd
```
**Lesson:** etcd listens on port `2379` for client traffic and stores data at `/var/lib/etcd` on the host. Backing up this directory backs up the entire cluster state.

### B5. Scheduler logs (sample)
```
I0425 ... successfully bound pod default/myapp to node minikube
I0425 ... pod default/hungry: no nodes available to schedule pods
```
**Lesson:** the Scheduler logs every binding decision and every failure. This is exactly its job: assign pods to nodes.

### B6. Controller Manager logs (sample)
```
I0425 ... Started "deployment"
I0425 ... Started "replicaset"
I0425 ... Started "endpointslice"
I0425 ... node not ready, marking unschedulable
```
**Lesson:** the Controller Manager runs many controllers (deployment, replicaset, node, endpoint, service, etc.) inside one binary. Each one runs an independent reconciliation loop.

### B7. Static pod manifests
```
$ sudo ls -la /etc/kubernetes/manifests/
-rw------- root root  etcd.yaml
-rw------- root root  kube-apiserver.yaml
-rw------- root root  kube-controller-manager.yaml
-rw------- root root  kube-scheduler.yaml
```
**Question answer (from B7):** if you delete `kube-scheduler.yaml`, the kubelet sees the file is gone and stops the scheduler pod. The cluster keeps running, but **no new pods can be scheduled** until you put the file back. Existing pods continue normally because they have already been bound to nodes.

---

## Section C — Worker Nodes

### C1. `kubectl describe node` highlights

**Conditions** — current health signals:
```
Type             Status
MemoryPressure   False
DiskPressure     False
PIDPressure      False
NetworkAvailable True
Ready            True
```
The kubelet sets these.

**Capacity vs Allocatable** — Capacity is the raw hardware (8 CPU, 16 GB). Allocatable is what's left after reserving for system + kubelet (e.g., 7.8 CPU, 15.5 GB).

**System Info** — kernel, OS, container runtime, kubelet version, kube-proxy version. This is where you confirm the runtime is e.g. `containerd://1.7.6`.

**Non-terminated Pods** — every pod currently scheduled on this node, with its CPU/memory requests.

**Allocated resources** — what percentage of the node is committed.

### C2. `jsonpath` extracts
`kubectl get node <name> -o jsonpath='{.status.nodeInfo}'` returns:
```
{"machineID":"...","systemUUID":"...","kernelVersion":"5.15...","osImage":"Ubuntu 22.04...",
"containerRuntimeVersion":"containerd://1.7.6","kubeletVersion":"v1.30.0","kubeProxyVersion":"v1.30.0"}
```

### C3. Pods on a specific node — useful when troubleshooting node problems.

### C4. Kubelet on the host
```
$ sudo systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/lib/systemd/system/kubelet.service; enabled)
   Active: active (running) since ...
```
**Question answer:** in `/var/lib/kubelet/config.yaml` you will find:
```yaml
staticPodPath: /etc/kubernetes/manifests
```
This is **why** the Control Plane components magically appear as pods — the kubelet watches this directory and starts whatever YAML is in it.

### C5. crictl
```
$ sudo crictl ps
CONTAINER     IMAGE          STATE     NAME                       POD ID
abc123...     nginx          Running   myapp                      ...
def456...     pause:3.9      Running   POD                        ...   <- the pod sandbox
```
**Lesson:** every pod has a hidden "pause" container that holds the network namespace. Your real container shares it.

### C6. kube-proxy mode
```
$ kubectl logs -n kube-system kube-proxy-xxx | grep -i "proxy mode"
Using iptables Proxier
```
Most clusters use **iptables**. Larger clusters often switch to **IPVS** for performance.

---

## Section D — Pods

### D1. `kubectl run myapp --image=nginx`
```
NAME    READY   STATUS    NODE
myapp   1/1     Running   minikube
```
**Lesson:** the Scheduler picked `minikube` (only node). `kubectl run` creates a single pod (no Deployment).

### D2. Full YAML — key fields:
```yaml
spec:
  nodeName: minikube           # set by Scheduler
  containers:
  - image: nginx
    name: myapp
status:
  phase: Running
  podIP: 10.244.0.6
  containerStatuses:
  - containerID: containerd://abc...
    ready: true
```
**Lesson:** every detail of the pod's lifecycle is visible — image, IP, container ID, ready state.

### D3. `kubectl describe pod myapp` events:
```
Events:
  Normal  Scheduled         <pod assigned to minikube>
  Normal  Pulling           <Pulling image "nginx">
  Normal  Pulled            <Successfully pulled image>
  Normal  Created           <Created container myapp>
  Normal  Started           <Started container myapp>
```
**Lesson:** a complete audit trail of how the pod came to be running. You can read each step in chronological order.

### D4. crictl inspect — shows the OCI container spec, mounts, network namespace info, and PID on the host.

### D5. Inside the pod
```
# hostname
myapp
# ip addr
1: lo: <LOOPBACK,UP> ...
2: eth0: <BROADCAST,MULTICAST,UP> mtu 1500 ...
    inet 10.244.0.6/24 scope global eth0
```
**Lesson:** the pod has its own network namespace and IP — separate from the node IP.

### D6. `kubectl logs myapp` shows nginx startup logs — the container's stdout/stderr.

### D7. **Question answer:** a "naked" pod (no owner) is gone forever once deleted. This is why production workloads always use Deployments / StatefulSets / DaemonSets — they wrap pods with controllers that recreate them.

---

## Section E — API Server in Action

### E1. `kubectl get events --watch` continually streams every cluster event in real time. Every action in Kubernetes is observable here.

### E2. `kubectl --v=8`
```
GET https://192.168.49.2:8443/api/v1/namespaces/default/pods?limit=500
Request Headers:
  Authorization: Bearer <token>
  Accept: application/json
Response Status: 200 OK
```
**Lesson:** `kubectl` is just an HTTP client. Every command translates to REST calls.

### E3. Direct curl
```
$ curl -k -H "Authorization: Bearer $TOKEN" $APISERVER/healthz
ok
```
**Question answer:** a healthy API Server returns the literal string `ok` from `/healthz`. The endpoint `/readyz?verbose` shows individual readiness checks.

---

## Section F — Self-Healing

### F1. Deployment chain
```
deployment.apps/web      3/3   3 available
replicaset.apps/web-7c   3     3 ready
pod/web-7c-aaaa          Running
pod/web-7c-bbbb          Running
pod/web-7c-cccc          Running
```
**Lesson:** Deployment owns ReplicaSet, which owns Pods. This three-level hierarchy is what enables rolling updates and history.

### F2. Watching after delete
```
web-7c-aaaa   Terminating
web-7c-zzzz   ContainerCreating   <- new pod appears within seconds
web-7c-zzzz   Running
```
**Lesson:** the ReplicaSet controller noticed `actual=2 < desired=3` and asked the API Server to create a new pod. Self-healing is just a control loop.

### F3. Scaling up/down — the same controller watches `replicas` and reconciles.

### F4. Drain / cordon
```
node/minikube cordoned
evicting pod default/web-7c-bbbb
node/minikube drained
```
**Lesson:** cordon = "no new pods here"; drain = "evict existing pods politely". Both are how you safely take a node out of service for maintenance.

---

## Section G — Scheduler Decision

### G1. `kubectl describe pod hungry`
```
Events:
  Warning  FailedScheduling   default-scheduler   0/1 nodes are available:
                              1 Insufficient memory.
```
**Lesson:** the Scheduler will refuse to bind a pod when no node fits. The pod stays in `Pending` until conditions change.

### G3. nodeSelector
```
$ kubectl get pod pinned -o wide
pinned   1/1   Running   ...   minikube   <- forced by nodeSelector
```
**Lesson:** you can constrain scheduling decisions with labels + selectors, taints/tolerations, and affinity rules.

---

## Section H — Cluster Resource Use

### H1. `kubectl top nodes`
```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   312m         3%     1894Mi          11%
```
**Lesson:** metrics-server scrapes the kubelet's `/stats/summary` and exposes a Metrics API used by HPA and `kubectl top`.

### H2. `kubectl get --raw='/readyz?verbose'`
```
[+]ping ok
[+]log ok
[+]etcd ok
[+]informer-sync ok
[+]poststarthook/start-kube-apiserver-admission-initializer ok
... 
readyz check passed
```
**Lesson:** this is the readiness probe used by load balancers and `kubeadm upgrade`.

---

## Section I — Architecture Map (Notes Key)

| Command | What it teaches you |
|---|---|
| `kubectl cluster-info` | API Server is the entry point; it exposes everything. |
| `kubectl get nodes -o wide` | Each node has IP, kubelet version, container runtime, kernel — the worker fleet. |
| `kubectl get pods -n kube-system` | Control Plane is itself a set of pods (static pods). |
| `ls /etc/kubernetes/manifests/` | The kubelet bootstraps the Control Plane from these YAML files. |
| `systemctl status kubelet` | The kubelet is a real Linux daemon, not a pod. It is the bridge between Linux and Kubernetes. |
| `crictl ps` | The container runtime (containerd) actually runs the containers. Every pod has a `pause` sandbox container. |
| `kubectl describe pod` | Events show the lifecycle: Scheduled -> Pulled -> Created -> Started. |
| `kubectl exec ... ip addr` | Each pod has its own IP and network namespace, independent of the node. |

---

## Section J — Stretch Solutions

### J1. `kubectl proxy` — exposes the API Server on `localhost` without auth headers (it injects them for you). Useful for dashboards and scripts.

### J2. `/stats/summary` — kubelet's per-node metrics including pod-level CPU, memory, network, and filesystem stats. This is the data source for `kubectl top`.

### J3. etcdctl get
```
/registry/pods/default/myapp
/registry/pods/kube-system/coredns-...
/registry/services/specs/default/kubernetes
/registry/configmaps/kube-system/kube-proxy
```
**Lesson:** Kubernetes objects are stored as keys in etcd organized by `/registry/<resource>/<namespace>/<name>`. The API Server is the only thing that should write to these keys.

### J4. Bringing the API Server down
- `kubectl` **stops working** (no API Server to talk to).
- Existing pods **keep running** because the kubelet has its cache of pod specs.
- New scheduling and self-healing **stop**.
- `docker ps` / `crictl ps` on each node still shows running containers.
**Lesson:** the data plane (workloads) is decoupled from the control plane. The cluster degrades gracefully, not catastrophically.

---

## What to Do Next

You have now seen every architectural component in action. Recommended next folders to create with the same Theory / Exercise / Solution pattern:

- Pods (deeper dive: lifecycle, probes, init containers, resources)
- ReplicaSets & Deployments
- Services (ClusterIP, NodePort, LoadBalancer, Headless)
- ConfigMaps & Secrets
- Volumes & Persistent Volumes
- Namespaces & Resource Quotas
- Ingress & Ingress Controllers
- StatefulSets
- DaemonSets
- Jobs & CronJobs
- RBAC & Service Accounts
- Networking (CNI, NetworkPolicy)
- Helm

Happy learning!
