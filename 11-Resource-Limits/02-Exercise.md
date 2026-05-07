# Resource Limits — Practical Exercises

## Setup
```bash
kubectl create namespace limlab
kubectl config set-context --current --namespace=limlab
```

---

## Section A — Force OOMKilled

### A1. Pod with low memory limit and a memory hog
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: oom-target }
spec:
  restartPolicy: Never
  containers:
  - name: c
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm","1","--vm-bytes","200M","--vm-hang","1"]
EOF

kubectl get pod oom-target -w
# wait for OOMKilled
```

### A2. Inspect
```bash
kubectl describe pod oom-target | grep -A5 "Last State"
# Reason: OOMKilled  Exit Code: 137
```

### A3. Cleanup
```bash
kubectl delete pod oom-target
```

---

## Section B — CrashLoopBackOff From Repeated OOMKill

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: oom-loop }
spec:
  containers:
  - name: c
    image: polinux/stress
    resources:
      limits: { memory: "50Mi" }
    command: ["stress"]
    args: ["--vm","1","--vm-bytes","100M","--vm-hang","1"]
EOF

kubectl get pod oom-loop -w
# Status oscillates: Running -> OOMKilled -> CrashLoopBackOff
kubectl get pod oom-loop -o jsonpath='{.status.containerStatuses[0].restartCount}'
```

Cleanup:
```bash
kubectl delete pod oom-loop
```

---

## Section C — Read Cgroup Files

```bash
kubectl run reader --image=alpine -- sh -c "sleep 3600"
kubectl wait --for=condition=Ready pod/reader --timeout=60s
kubectl exec reader -- cat /sys/fs/cgroup/memory.max
kubectl exec reader -- cat /sys/fs/cgroup/cpu.max
kubectl delete pod reader
```

A pod with no limits set shows `max` in both files.

---

## Section D — Observe CPU Throttling

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: throttler }
spec:
  containers:
  - name: c
    image: polinux/stress
    resources:
      limits: { cpu: "100m" }
    command: ["stress"]
    args: ["--cpu","2"]                # try to use 2 CPUs while limited to 0.1
EOF

kubectl wait --for=condition=Ready pod/throttler --timeout=60s

# Wait a bit, then check throttling
sleep 30
kubectl exec throttler -- cat /sys/fs/cgroup/cpu.stat
```
Expected: `nr_throttled` and `throttled_usec` (or `throttled_time` on cgroup v1) are large numbers — the container is spending most of its time waiting for CPU.

Cleanup:
```bash
kubectl delete pod throttler
```

---

## Section E — Verify QoS Class via Limits

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: g }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "100m", memory: "100Mi" }
      limits:   { cpu: "100m", memory: "100Mi" }
EOF

kubectl get pod g -o jsonpath='{.status.qosClass}'    # Guaranteed

# Slightly different limits -> Burstable
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: b }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      requests: { cpu: "100m", memory: "100Mi" }
      limits:   { cpu: "200m", memory: "100Mi" }
EOF

kubectl get pod b -o jsonpath='{.status.qosClass}'    # Burstable

kubectl delete pod g b
```

---

## Section F — LimitRange to Enforce Defaults

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata: { name: enforce }
spec:
  limits:
  - type: Container
    max:        { cpu: "1",   memory: "1Gi" }       # max each container
    min:        { cpu: "10m", memory: "10Mi" }      # minimum
    defaultRequest: { cpu: "50m", memory: "64Mi" }  # default if unset
    default:        { cpu: "500m", memory: "256Mi" }
EOF
```

Try to over-limit:
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: too-big }
spec:
  containers:
  - name: c
    image: nginx
    resources:
      limits: { cpu: "4", memory: "8Gi" }       # exceeds max
EOF
```
Expected: rejected with `maximum cpu usage per Container is 1`.

Cleanup:
```bash
kubectl delete pod too-big --ignore-not-found
kubectl delete limitrange enforce
```

---

## Section G — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace limlab
```
