# Containers & Pods — Practical Exercises

Hands-on exercises. You will see Linux primitives, build images, create every kind of pod, break things on purpose, and watch the kubelet react.

## Prerequisites

- A running Kubernetes cluster (minikube, kind, k3d, EKS, etc.)
- `docker` or `podman` on your machine
- `kubectl` configured to talk to your cluster

Quick check:
```bash
kubectl get nodes
docker version
```

---

## Section A — See the Linux Primitives Behind a Container

You will run a real container and inspect the namespaces and cgroups the kernel set up for it.

### A1. Start a container
```bash
docker run -d --name probe nginx
docker ps
```

### A2. Find its host PID
```bash
docker inspect probe --format '{{.State.Pid}}'
# remember this number, e.g., 12345
```

### A3. List the namespaces of that PID
```bash
sudo ls -l /proc/<PID>/ns/
```
You should see entries like `pid`, `net`, `mnt`, `uts`, `ipc`, `user`, `cgroup`. Each points to a unique inode — that is the namespace identity.

### A4. Compare with the host
```bash
sudo ls -l /proc/1/ns/
```
**Question:** which namespaces are different from PID 1's, and which are the same? Why?

### A5. Look at its cgroup limits
```bash
cat /proc/<PID>/cgroup
# follow that path under /sys/fs/cgroup/...
cat /sys/fs/cgroup/<path>/memory.max
cat /sys/fs/cgroup/<path>/cpu.max
```

### A6. Set a limit and watch it appear
```bash
docker rm -f probe
docker run -d --name probe --memory 256m --cpus 0.5 nginx
PID=$(docker inspect probe --format '{{.State.Pid}}')
cat /sys/fs/cgroup/$(awk -F: '{print $3}' /proc/$PID/cgroup)/memory.max
cat /sys/fs/cgroup/$(awk -F: '{print $3}' /proc/$PID/cgroup)/cpu.max
```

### A7. Enter a container's namespaces manually (no docker exec!)
```bash
sudo nsenter -t $PID -p -n -m -u hostname
sudo nsenter -t $PID -p -n -m -u ip addr
sudo nsenter -t $PID -p -n -m -u ps -ef
```
**This is exactly what `docker exec` and `kubectl exec` do under the hood.**

### A8. Cleanup
```bash
docker rm -f probe
```

---

## Section B — Build Your Own Container Image

### B1. Create a tiny app
```bash
mkdir myapp && cd myapp
cat > server.js <<'EOF'
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Hello from ' + (process.env.HOSTNAME || 'pod') + '\n');
}).listen(3000);
console.log('listening on 3000');
EOF

cat > Dockerfile <<'EOF'
FROM node:20-alpine
WORKDIR /app
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
EOF
```

### B2. Build the image and observe layers
```bash
docker build -t myapp:v1 .
docker history myapp:v1
docker image inspect myapp:v1 | jq '.[0].RootFS.Layers'
```

### B3. Build again and observe the cache
```bash
docker build -t myapp:v1 .   # second time should be all cached
echo " " >> server.js
docker build -t myapp:v2 .   # only the COPY layer rebuilds
```

### B4. Look at the final size
```bash
docker images | grep myapp
```

### B5. Push to the cluster's local registry
For minikube:
```bash
eval $(minikube docker-env)
docker build -t myapp:v1 .
```
For kind:
```bash
kind load docker-image myapp:v1 --name <cluster-name>
```

---

## Section C — Pod Basics

### C1. Run a single-container pod
```bash
kubectl run web --image=nginx --port=80
kubectl get pod web -o wide
```

### C2. Look at the FULL spec the kubelet actually runs
```bash
kubectl get pod web -o yaml > pod-actual.yaml
cat pod-actual.yaml | head -60
```
Compare what you typed (`--image=nginx`) versus what is in the YAML — Kubernetes adds defaults (resources, terminationGracePeriod, securityContext, etc.).

### C3. Find the pause container on the node
For minikube:
```bash
minikube ssh
sudo crictl pods
sudo crictl ps -a    # all containers including pause
exit
```
**Spot:** the container named `POD` (the pause sandbox) and the named container (`web`).

### C4. The pod's IP comes from the CNI
```bash
kubectl get pod web -o jsonpath='{.status.podIP}'
kubectl exec -it web -- ip addr
```
**Question:** is the pod IP routable from another pod on the same node? Test it:
```bash
kubectl run client --image=busybox --rm -it -- wget -qO- <web-pod-ip>:80
```

### C5. Cleanup
```bash
kubectl delete pod web
```

---

## Section D — Pod Lifecycle in Action

### D1. Watch a pod go through its phases
```bash
kubectl run lifecycle-demo --image=nginx --restart=Never
kubectl get pod lifecycle-demo -w
# Press Ctrl+C when Running
```

### D2. Force `ImagePullBackOff`
```bash
kubectl run badimage --image=does/not/exist:nope --restart=Never
kubectl get pod badimage
kubectl describe pod badimage | tail -15
kubectl delete pod badimage
```

### D3. Force `CrashLoopBackOff`
```bash
kubectl run crasher --image=busybox --restart=Always -- sh -c "echo starting; sleep 2; exit 1"
kubectl get pod crasher -w
# Watch it restart, with increasing back-off
kubectl describe pod crasher | grep -A5 "Last State"
kubectl delete pod crasher
```

### D4. Force `OOMKilled`
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hungry
spec:
  restartPolicy: Never
  containers:
  - name: c
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
EOF

kubectl get pod hungry -w
# Watch it transition: Running -> OOMKilled
kubectl describe pod hungry | grep -A5 "Last State"
kubectl delete pod hungry
```

### D5. See the difference between Pod phase and Container status
```bash
kubectl run multi --image=nginx --restart=Never
kubectl get pod multi -o jsonpath='{.status.phase}'      # Running
kubectl get pod multi -o jsonpath='{.status.containerStatuses[0].state}'   # detailed map
kubectl delete pod multi
```

---

## Section E — Multi-Container Pods (Sidecar Pattern)

### E1. Build a sidecar pod (web + log shipper)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-demo
spec:
  containers:
  - name: web
    image: nginx
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  - name: log-shipper
    image: busybox
    command: ["sh", "-c", "tail -F /var/log/nginx/access.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log/nginx
  volumes:
  - name: shared-logs
    emptyDir: {}
EOF

kubectl get pod sidecar-demo
kubectl get pod sidecar-demo -o jsonpath='{.spec.containers[*].name}'
```

### E2. Verify they share the network namespace
```bash
kubectl exec sidecar-demo -c log-shipper -- wget -qO- localhost:80
```
The sidecar accessed nginx via `localhost`. They share an IP.

### E3. Verify they share the volume
```bash
# Hit nginx to generate a log
kubectl exec sidecar-demo -c web -- curl -s localhost:80 >/dev/null
kubectl logs sidecar-demo -c log-shipper --tail=2
# You should see the access log line
```

### E4. Cleanup
```bash
kubectl delete pod sidecar-demo
```

---

## Section F — Init Containers

### F1. Wait for a service before starting
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  - name: wait-for-api
    image: busybox
    command: ["sh", "-c", "until nslookup kubernetes.default; do echo waiting...; sleep 2; done; echo ready!"]
  containers:
  - name: app
    image: nginx
EOF

kubectl get pod init-demo -w
# Phase: Pending -> PodInitializing -> Running
kubectl logs init-demo -c wait-for-api
```

### F2. What if the init fails?
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: init-failing
spec:
  restartPolicy: Never
  initContainers:
  - name: bad-init
    image: busybox
    command: ["sh", "-c", "echo failing!; exit 1"]
  containers:
  - name: app
    image: nginx
EOF

kubectl get pod init-failing
kubectl describe pod init-failing | tail -15
```

### F3. Cleanup
```bash
kubectl delete pod init-demo init-failing
```

---

## Section G — Probes

### G1. A liveness probe that succeeds, then fails
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: app
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command: ["cat", "/tmp/healthy"]
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 1
EOF

kubectl get pod liveness-demo -w
```
After ~30 seconds, the file is removed, liveness fails, kubelet restarts the container. Watch the RESTARTS column tick up.

```bash
kubectl describe pod liveness-demo | grep -A3 "Liveness"
```

### G2. A readiness probe — pod stays running but is not Ready
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  labels: { app: ready }
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","sleep 30; touch /tmp/ready; sleep 600"]
    readinessProbe:
      exec: { command: ["cat", "/tmp/ready"] }
      periodSeconds: 3
EOF

kubectl get pod readiness-demo -w
# READY column starts as 0/1, then 1/1 after ~30s
```

### G3. Build a Service and watch endpoints react to readiness
```bash
kubectl expose pod readiness-demo --port=80 --name=ready-svc
kubectl get endpoints ready-svc
# When pod becomes Ready -> its IP appears in endpoints
# Force not-ready:
kubectl exec readiness-demo -- rm /tmp/ready
sleep 10
kubectl get endpoints ready-svc
# IP is gone from endpoints!
```

### G4. Cleanup
```bash
kubectl delete pod liveness-demo readiness-demo
kubectl delete svc ready-svc
```

---

## Section H — Resources & QoS

### H1. Create one pod of each QoS class
```bash
# Guaranteed: requests == limits
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: qos-guaranteed }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "200m", memory: "200Mi" }
      limits:   { cpu: "200m", memory: "200Mi" }
EOF

# Burstable: requests < limits
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: qos-burstable }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "100m", memory: "100Mi" }
      limits:   { cpu: "200m", memory: "300Mi" }
EOF

# BestEffort: nothing
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: qos-besteffort }
spec:
  containers:
  - name: c
    image: nginx
EOF
```

### H2. Confirm the QoS class
```bash
for p in qos-guaranteed qos-burstable qos-besteffort; do
  echo -n "$p: "
  kubectl get pod $p -o jsonpath='{.status.qosClass}'
  echo
done
```

### H3. Cleanup
```bash
kubectl delete pod qos-guaranteed qos-burstable qos-besteffort
```

---

## Section I — Security Context

### I1. Refuse to run as root
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: root-blocker }
spec:
  securityContext:
    runAsNonRoot: true
  containers:
  - name: c
    image: nginx       # nginx:latest runs as root by default
EOF

kubectl get pod root-blocker
kubectl describe pod root-blocker | tail -10
```
**Question:** what error appears, and why?

### I2. Make the root filesystem read-only
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: readonly-fs }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","sleep 600"]
    securityContext:
      readOnlyRootFilesystem: true
EOF

kubectl exec readonly-fs -- touch /tmp/x
# error or success?
kubectl exec readonly-fs -- touch /etc/x
# error?
```

### I3. Cleanup
```bash
kubectl delete pod root-blocker readonly-fs
```

---

## Section J — Graceful Termination

### J1. A pod with a SIGTERM handler
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: graceful }
spec:
  terminationGracePeriodSeconds: 30
  containers:
  - name: c
    image: busybox
    command:
    - sh
    - -c
    - 'trap "echo SIGTERM; sleep 5; echo done; exit 0" TERM; while true; do sleep 1; done'
EOF
```

### J2. Delete and watch the trap fire
Open two terminals.
- Terminal 1: `kubectl logs -f graceful`
- Terminal 2: `kubectl delete pod graceful`

You will see "SIGTERM ... done" in the logs before the pod disappears.

### J3. Compare with a pod that ignores SIGTERM
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: ungraceful }
spec:
  terminationGracePeriodSeconds: 5
  containers:
  - name: c
    image: busybox
    command: ["sh", "-c", "trap '' TERM; while true; do echo alive; sleep 1; done"]
EOF

time kubectl delete pod ungraceful
```
**Question:** how long did the deletion take? Why?

### J4. Cleanup
```bash
kubectl delete pod graceful ungraceful --ignore-not-found
```

---

## Section K — Inspect a Live Pod End to End

A real-world combined exercise: deploy a non-trivial pod and inspect every layer.

### K1. Deploy
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: full
  labels: { app: full }
spec:
  initContainers:
  - name: init-msg
    image: busybox
    command: ["sh","-c","echo init done > /shared/init.log"]
    volumeMounts:
    - { name: shared, mountPath: /shared }
  containers:
  - name: web
    image: nginx
    ports: [{ containerPort: 80 }]
    resources:
      requests: { cpu: "100m", memory: "100Mi" }
      limits:   { cpu: "200m", memory: "200Mi" }
    livenessProbe:
      httpGet: { path: /, port: 80 }
      periodSeconds: 10
    readinessProbe:
      tcpSocket: { port: 80 }
      periodSeconds: 5
    volumeMounts:
    - { name: shared, mountPath: /usr/share/nginx/html/init }
  - name: tail
    image: busybox
    command: ["sh","-c","tail -F /shared/init.log"]
    volumeMounts:
    - { name: shared, mountPath: /shared }
  volumes:
  - name: shared
    emptyDir: {}
EOF
```

### K2. Inspect each layer
```bash
# What phase, IP, node?
kubectl get pod full -o wide

# Containers and statuses
kubectl get pod full -o jsonpath='{range .status.containerStatuses[*]}{.name}{": "}{.state}{"\n"}{end}'

# QoS, restartPolicy, gracePeriod
kubectl get pod full -o jsonpath='{.status.qosClass}{"\n"}{.spec.restartPolicy}{"\n"}{.spec.terminationGracePeriodSeconds}{"\n"}'

# Probe configs
kubectl get pod full -o jsonpath='{.spec.containers[0].livenessProbe}'
kubectl get pod full -o jsonpath='{.spec.containers[0].readinessProbe}'

# Logs from each container
kubectl logs full -c web --tail=5
kubectl logs full -c tail --tail=5
kubectl logs full -c init-msg

# Hit nginx via the pod IP from another pod
PODIP=$(kubectl get pod full -o jsonpath='{.status.podIP}')
kubectl run --rm -it tester --image=busybox --restart=Never -- wget -qO- $PODIP:80/init/init.log
```

### K3. Cleanup
```bash
kubectl delete pod full
```

---

## Section L — Stretch / Advanced

### L1. Use `kubectl debug` to attach a debug container
```bash
kubectl run target --image=nginx
kubectl debug -it target --image=busybox --target=target -- sh
# inside the debug container you can see target's process tree
```

### L2. Pod with both `requests` only — what is its QoS?
Try it and predict before you check.

### L3. Make a pod stuck in `Pending` for two reasons
1. Request more memory than any node has
2. Add a `nodeSelector` that no node satisfies
Read the events with `kubectl describe pod <name>` to see how the scheduler explains each failure.

### L4. Run a pod and `kill -9` its main process from inside the container. What happens?
```bash
kubectl run killtest --image=nginx
kubectl exec killtest -- kill -9 1
kubectl get pod killtest -w
```

### L5. Use ephemeral containers for live debugging
```bash
kubectl debug -it killtest --image=busybox --share-processes
ps -ef        # see nginx PIDs from the debug container
```

---

When you are done, open `03-Solution.md` for expected outputs, explanations, and the answers to each question.
