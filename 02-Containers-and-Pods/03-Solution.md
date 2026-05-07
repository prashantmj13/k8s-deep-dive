# Containers & Pods — Solutions

Expected outputs, explanations, and answers for every exercise in `02-Exercise.md`.

---

## Section A — Linux Primitives

### A3. `ls -l /proc/<PID>/ns/` — sample output
```
lrwxrwxrwx ... cgroup -> 'cgroup:[4026532301]'
lrwxrwxrwx ... ipc    -> 'ipc:[4026532299]'
lrwxrwxrwx ... mnt    -> 'mnt:[4026532297]'
lrwxrwxrwx ... net    -> 'net:[4026532302]'
lrwxrwxrwx ... pid    -> 'pid:[4026532300]'
lrwxrwxrwx ... user   -> 'user:[4026531837]'
lrwxrwxrwx ... uts    -> 'uts:[4026532298]'
```
The number after the colon is the **namespace ID** (an inode). Two processes with the same number share that namespace.

### A4. Question — which namespaces differ from PID 1's?
- `pid`, `net`, `mnt`, `uts`, `ipc`, `cgroup` will be **different** — those are the isolated ones.
- `user` is usually the **same** as the host's because user namespacing is off by default. (When you turn on user namespaces, root inside is mapped to a non-root UID outside — much safer.)

### A5. cgroup v2 example
```
$ cat /proc/12345/cgroup
0::/system.slice/docker-abc123...scope
$ cat /sys/fs/cgroup/system.slice/docker-abc123.../memory.max
max
$ cat /sys/fs/cgroup/.../cpu.max
max 100000
```
With no limits set, both are `max`.

### A6. Limits applied
```
$ cat .../memory.max
268435456              # 256 * 1024 * 1024
$ cat .../cpu.max
50000 100000           # 50% of one CPU period
```
**Lesson:** Docker / Kubernetes limits are translated by the runtime into cgroup writes the kernel enforces.

### A7. `nsenter`
- `-t <pid>` join the namespaces of this PID
- `-p -n -m -u` join pid, net, mnt, uts namespaces
- The command then runs inside those namespaces

This is exactly what `kubectl exec` does behind the scenes: ask the runtime to enter the target's namespaces.

---

## Section B — Build Your Own Image

### B2. `docker history myapp:v1`
```
IMAGE      CREATED         CREATED BY                 SIZE
abc123     2s ago          CMD ["node" "server.js"]   0B
def456     2s ago          EXPOSE 3000                0B
ghi789     2s ago          COPY server.js .           257B
jkl012     2s ago          WORKDIR /app               0B
<base>     ...             FROM node:20-alpine        ~180MB
```
**Lesson:** zero-byte layers are metadata only (`CMD`, `EXPOSE`, `WORKDIR`). Only filesystem-changing instructions add real bytes.

### B3. Cache behavior
- The first re-run is fully cached: every step says `Using cache`.
- After modifying `server.js`, the `COPY server.js .` step **and every step after it** rebuilds. Earlier steps stay cached.
- This is why you place rarely-changing instructions early (e.g., `COPY package.json . && npm install` before `COPY . .`).

### B4. Final size
A typical Node 20 alpine image is ~180MB. Compare to a `node:20` (Debian based) image at ~1.1GB. Choosing a slim base saves enormous storage and pull time.

---

## Section C — Pod Basics

### C2. The materialized YAML
Kubernetes adds a lot of defaults you didn't type:
```yaml
spec:
  restartPolicy: Always
  terminationGracePeriodSeconds: 30
  dnsPolicy: ClusterFirst
  serviceAccountName: default
  schedulerName: default-scheduler
  enableServiceLinks: true
  containers:
  - name: web
    image: nginx
    imagePullPolicy: Always           # because tag wasn't pinned
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    resources: {}
status:
  phase: Running
  podIP: 10.244.0.7
  qosClass: BestEffort               # because no requests/limits
```

### C3. crictl pods / ps
```
$ sudo crictl pods
POD ID       CREATED        STATE       NAME       NAMESPACE
abc123       30s ago        Ready       web        default
$ sudo crictl ps -a
CONTAINER    IMAGE          STATE       NAME
def456       nginx          Running     web
abc123       pause:3.9      Running     POD             <- the pause sandbox
```
**Lesson:** every pod has a pause container. That is what holds the network namespace alive while the real containers come and go (e.g., across restarts).

### C4. Question — pod IP routability
**Answer:** yes. Within the cluster, every pod can reach every other pod by its IP without NAT. That is one of the four fundamental Kubernetes networking rules. The CNI plugin (Calico, Flannel, Cilium...) is responsible for making this true.

The `wget` from a busybox pod to the web pod's IP succeeds.

---

## Section D — Lifecycle

### D2. ImagePullBackOff
```
Events:
  Normal   Pulling   Pulling image "does/not/exist:nope"
  Warning  Failed    Failed to pull image: rpc error: code = NotFound
  Warning  Failed    Error: ErrImagePull
  Normal   BackOff   Back-off pulling image
  Warning  Failed    Error: ImagePullBackOff
```
**Lesson:** the kubelet retries with exponential backoff (10s, 20s, 40s, ...). The pod sits in `ContainerCreating`/`Waiting` indefinitely until the image is reachable.

### D3. CrashLoopBackOff
```
NAME      READY   STATUS             RESTARTS   AGE
crasher   0/1     CrashLoopBackOff   4          70s
```
Inside `describe`:
```
Last State:    Terminated
  Reason:      Error
  Exit Code:   1
  Started:     ...
  Finished:    ...
```
**Lesson:** crash + restart = backoff cycle. Backoff caps at 5 minutes between restarts. The pod stays `Running` (phase) but not `Ready`.

### D4. OOMKilled
```
Last State:    Terminated
  Reason:      OOMKilled
  Exit Code:   137
```
Exit code 137 = 128 + signal 9 (SIGKILL sent by the kernel OOM killer). With `restartPolicy: Never`, the pod ends in phase `Failed`.

### D5. Phase vs container status
```
$ kubectl get pod multi -o jsonpath='{.status.phase}'
Running
$ kubectl get pod multi -o jsonpath='{.status.containerStatuses[0].state}'
{"running":{"startedAt":"2026-04-25T..."}}
```
**Lesson:** the pod's `phase` summarizes; the container's `state` is detailed. Always look at both when debugging.

---

## Section E — Sidecar Pattern

### E2. Localhost between containers
The log shipper successfully fetches `localhost:80` because **all containers in a pod share a network namespace**. They see the same loopback interface. They cannot have port conflicts (only one container can bind 80).

### E3. Shared volume
The `emptyDir` volume is mounted into both containers at the same path. Anything one writes the other can read. `emptyDir` lives for the lifetime of the pod — gone when the pod is deleted.

---

## Section F — Init Containers

### F1. Phase progression
```
NAME        READY   STATUS            RESTARTS   AGE
init-demo   0/1     Init:0/1          0          2s
init-demo   0/1     PodInitializing   0          5s
init-demo   1/1     Running           0          7s
```
**Lesson:** the main container does NOT start until `Init:N/N` (all init containers complete).

### F2. Failed init
With `restartPolicy: Never`, the pod ends in phase `Failed`. With `Always` or `OnFailure`, the kubelet keeps restarting the init container (in a backoff loop) until it succeeds — the main containers never start until then.

---

## Section G — Probes

### G1. Liveness restart
```
NAME             READY   STATUS    RESTARTS
liveness-demo    1/1     Running   0
liveness-demo    1/1     Running   1   <- after first failure
liveness-demo    1/1     Running   2
```
After ~30s the file is removed; one liveness check fails (`failureThreshold: 1`); kubelet sends SIGTERM, container restarts. The cycle repeats forever.

`kubectl describe pod`:
```
Liveness:   exec [cat /tmp/healthy] delay=5s timeout=1s period=5s #success=1 #failure=1
Events:
  Warning  Unhealthy  Liveness probe failed: cat: can't open '/tmp/healthy'
  Normal   Killing    Container app failed liveness probe, will be restarted
```

### G2. Readiness in action
- For the first ~30 seconds, `READY` shows `0/1` — pod is `Running` but not Ready.
- Once `/tmp/ready` is touched, it flips to `1/1`.

### G3. Endpoints reflect readiness
```
$ kubectl get endpoints ready-svc
NAME        ENDPOINTS         AGE
ready-svc   10.244.0.9:80     30s   <- only when pod is Ready
```
After `rm /tmp/ready`:
```
$ kubectl get endpoints ready-svc
NAME        ENDPOINTS         AGE
ready-svc   <none>            60s   <- pod is still running but not Ready
```
**Lesson:** readiness is the contract between a pod and a Service. It is the safest knob for graceful drain or warm-up.

---

## Section H — QoS

### H2. Output
```
qos-guaranteed: Guaranteed
qos-burstable: Burstable
qos-besteffort: BestEffort
```
**Lesson:** the QoS class is **derived** from your spec; you cannot set it directly. The kubelet uses it during eviction decisions when the node is under pressure: BestEffort dies first.

---

## Section I — Security Context

### I1. Question — what error?
```
Status:  Failed
Reason:  CreateContainerConfigError
Message: container has runAsNonRoot and image will run as root
```
**Why:** the nginx official image runs as root by default. With `runAsNonRoot: true`, the kubelet refuses to start the container. Fix: use `nginxinc/nginx-unprivileged` or set `runAsUser: 101` (the nginx UID).

### I2. Read-only root FS
- `touch /tmp/x` — `/tmp` is on a separate (writable) tmpfs in many setups, so this can succeed in some images. Often **fails** with strict read-only.
- `touch /etc/x` — fails: `Read-only file system`.

To allow writes only where needed, mount an `emptyDir` at the path:
```yaml
volumeMounts:
- name: tmp
  mountPath: /tmp
volumes:
- name: tmp
  emptyDir: {}
```

---

## Section J — Termination

### J2. Graceful logs
```
$ kubectl logs -f graceful
SIGTERM
done
```
Total deletion time: ~5 seconds (the trap's sleep). Kubelet sent SIGTERM, your trap caught it, slept 5s, exited 0.

### J3. Question — how long for the ungraceful pod?
**Answer:** ~5 seconds. Why? Because it `trap '' TERM` (ignored SIGTERM). The kubelet waited the `terminationGracePeriodSeconds: 5` you set, then sent SIGKILL. With the default 30s, this same pod would take 30 seconds to delete.

**Lesson:** apps that ignore SIGTERM appear to "hang" during rollouts. Always handle SIGTERM and exit promptly.

---

## Section K — Full Pod Inspection

### K2. Sample outputs
```
$ kubectl get pod full -o wide
NAME   READY   STATUS    NODE       IP
full   2/2     Running   minikube   10.244.0.10

$ kubectl get pod full -o jsonpath='{range .status.containerStatuses[*]}{.name}{": "}{.state}{"\n"}{end}'
web: {"running":{"startedAt":"..."}}
tail: {"running":{"startedAt":"..."}}

$ kubectl get pod full -o jsonpath='{.status.qosClass}'
Burstable

$ kubectl exec full -c tail -- cat /shared/init.log
init done

$ wget -qO- $PODIP:80/init/init.log
init done
```
**Lesson:** the init container wrote `/shared/init.log` once at startup; both runtime containers can read it via the shared volume; nginx serves it at `/init/init.log` because the same volume is mounted into nginx's html dir.

---

## Section L — Stretch

### L1. `kubectl debug --target` attaches an ephemeral container that **shares the target's PID namespace**, so you can `ps -ef` and see the target's processes. This is invaluable when the target image is `distroless` or `scratch` (no shell).

### L2. Pod with `requests` only — QoS?
**Burstable.** The class is "Guaranteed" only when **every** container has CPU+memory requests **equal to** limits. Anything less = Burstable (unless none are set, then BestEffort).

### L3. Pending pod events
```
Events:
  Warning  FailedScheduling  0/1 nodes are available: 1 Insufficient memory.
```
or
```
Warning  FailedScheduling  0/1 nodes are available: 1 node(s) didn't match Pod's node affinity/selector.
```
**Lesson:** the scheduler always tells you why it failed. Read the events.

### L4. Killing PID 1
```
$ kubectl exec killtest -- kill -9 1
$ kubectl get pod killtest -w
NAME       READY   STATUS    RESTARTS   AGE
killtest   1/1     Running   1          ...
```
**Lesson:** killing PID 1 inside a container kills the container. If `restartPolicy: Always` (the default for `kubectl run`-created pods nowadays — actually `kubectl run` defaults to `Never`, but a Deployment-owned pod is `Always`), the container is restarted. The pod is preserved — only its container is replaced.

### L5. Ephemeral debug container
```
$ kubectl debug -it killtest --image=busybox --share-processes
# ps -ef
PID   USER     TIME  COMMAND
    1 root      0:00 nginx: master process nginx -g daemon off;
   30 nginx     0:00 nginx: worker process
   ...
```
**Lesson:** ephemeral containers are the modern way to debug a running pod without changing it. They cannot be added at creation time — only via `kubectl debug` afterward.

---

## Cheat Sheet — Things You Saw in Action

| Concept | Command that proved it |
|---|---|
| Containers are processes | `nsenter -t <pid> ... ps -ef` |
| Cgroups enforce limits | `cat /sys/fs/cgroup/.../memory.max` |
| Image layers are cached | `docker history`, second `docker build` |
| Pause container holds network ns | `crictl ps -a \| grep POD` |
| Pod IP is routable | `kubectl run client ... wget pod-ip` |
| Init containers gate main | `kubectl get pod -w` (Init:0/1 -> Running) |
| Liveness restarts | `kubectl describe pod` (Killing event) |
| Readiness controls endpoints | `kubectl get endpoints` before/after probe fail |
| QoS is derived | `kubectl get pod -o jsonpath='{.status.qosClass}'` |
| SIGTERM grace period | trap + `time kubectl delete pod` |

---

## What's Next

You now know how containers and pods work down to Linux primitives. Recommended folders to create next with the same Theory / Exercise / Solution structure:

- ReplicaSets & Deployments (rolling updates, history, rollbacks)
- Services (ClusterIP, NodePort, LoadBalancer, Headless, EndpointSlices)
- ConfigMaps & Secrets (mount, env, projection, immutable)
- Volumes (emptyDir, hostPath, PVC/PV, StorageClass)
- Namespaces & ResourceQuotas
- Ingress & Ingress Controllers
- StatefulSets, DaemonSets, Jobs, CronJobs
- RBAC, Service Accounts, NetworkPolicies
- Helm, Kustomize

Happy hacking!
