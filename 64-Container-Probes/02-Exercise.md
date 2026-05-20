# 64 — Container Probes | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes
> **Namespace:** `probes-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace probes-lab
kubectl config set-context --current --namespace=probes-lab
```

---

## Exercise 2 — Liveness probe: HTTP GET

Deploy an nginx pod with a liveness probe. We'll simulate a healthy app first.

```yaml
# pod-liveness-http.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-http
  namespace: probes-lab
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 3
      timeoutSeconds: 2
```

```bash
kubectl apply -f pod-liveness-http.yaml
kubectl get pod pod-liveness-http -w   # watch for 30s — should stay Running
kubectl describe pod pod-liveness-http | grep -A10 "Liveness"
```

---

## Exercise 3 — Simulate liveness failure

```yaml
# pod-liveness-fail.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-fail
  namespace: probes-lab
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "App starting..."
      sleep 20
      echo "App crashing now — removing health file"
      exit 0     # container exits, triggers restart
    livenessProbe:
      exec:
        command: ["sh", "-c", "exit 1"]   # always fails immediately
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

```bash
kubectl apply -f pod-liveness-fail.yaml
kubectl get pod pod-liveness-fail -w
```

> **Observe:** After ~20s the RESTARTS column increments — liveness probe failed 3 times → container restarted.

```bash
kubectl describe pod pod-liveness-fail | grep -E "Liveness|Unhealthy|Killing|Restart"
```

---

## Exercise 4 — Readiness probe: gates traffic

```yaml
# pod-readiness.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-readiness
  namespace: probes-lab
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Starting up — not ready yet"
      sleep 15
      echo "ready" > /tmp/ready
      echo "App is now ready"
      sleep 3600
    readinessProbe:
      exec:
        command: ["test", "-f", "/tmp/ready"]
      initialDelaySeconds: 2
      periodSeconds: 3
      failureThreshold: 5
```

```bash
kubectl apply -f pod-readiness.yaml

# Create a service to watch endpoints
kubectl expose pod pod-readiness --port=80 --target-port=80 --name=readiness-svc

# Watch pod status and endpoints simultaneously
kubectl get pod pod-readiness -w &
kubectl get endpoints readiness-svc -w
```

> **Observe:** For the first ~15 seconds the pod shows `0/1 READY` and the endpoints list is **empty**. After the file is created the pod becomes `1/1 READY` and the endpoint IP appears.

```bash
# Stop background watches
kill %1 %2 2>/dev/null; true
```

---

## Exercise 5 — Startup probe for slow-starting app

```yaml
# pod-startup.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-startup
  namespace: probes-lab
spec:
  containers:
  - name: app
    image: busybox
    command:
    - sh
    - -c
    - |
      echo "Simulating slow startup — takes 25 seconds"
      sleep 25
      echo "done" > /tmp/started
      echo "Startup complete!"
      sleep 3600
    startupProbe:
      exec:
        command: ["test", "-f", "/tmp/started"]
      failureThreshold: 10    # 10 × 5s = 50 seconds max startup window
      periodSeconds: 5
    livenessProbe:
      exec:
        command: ["test", "-f", "/tmp/started"]
      periodSeconds: 10
      failureThreshold: 3
```

```bash
kubectl apply -f pod-startup.yaml
kubectl get pod pod-startup -w
```

> **Observe:** Pod shows `0/1` (startup probe running). After 25s the startup probe passes, pod becomes `1/1`, and the liveness probe takes over.

```bash
kubectl describe pod pod-startup | grep -A5 "Startup\|Liveness"
```

---

## Exercise 6 — All three probes together

```yaml
# pod-all-probes.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-all-probes
  namespace: probes-lab
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    startupProbe:
      httpGet:
        path: /
        port: 80
      failureThreshold: 6
      periodSeconds: 5
    livenessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /
        port: 80
      periodSeconds: 5
      failureThreshold: 3
      successThreshold: 1
```

```bash
kubectl apply -f pod-all-probes.yaml
kubectl get pod pod-all-probes -w
kubectl describe pod pod-all-probes | grep -E "Startup|Liveness|Readiness" | head -20
```

---

## Exercise 7 — TCP socket probe

```yaml
# pod-tcp-probe.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-tcp-probe
  namespace: probes-lab
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
```

```bash
kubectl apply -f pod-tcp-probe.yaml
kubectl get pod pod-tcp-probe -w
kubectl describe pod pod-tcp-probe | grep -A5 "Liveness\|Readiness"
```

---

## Exercise 8 — Observe probe timing in events

```bash
# Check events for any probe-related actions
kubectl get events --sort-by='.lastTimestamp' | grep -E "Probe|Unhealthy|Killing|BackOff"

# Describe all pods to see probe configs at a glance
kubectl describe pods | grep -E "Liveness:|Readiness:|Startup:"
```

---

## Exercise 9 — Cleanup

```bash
kubectl delete namespace probes-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between a liveness probe failure and a readiness probe failure?
2. Why should liveness probes NOT check database connectivity?
3. What problem does the startup probe solve that `initialDelaySeconds` cannot?
4. A pod is `1/1 Running` but not receiving traffic — what is likely wrong?
5. What does `successThreshold: 2` on a readiness probe mean?
