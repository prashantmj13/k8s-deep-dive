# Logging — Practical Exercises

## Setup
```bash
kubectl create namespace loglab
kubectl config set-context --current --namespace=loglab
```

---

## Section A — Basic kubectl logs

### A1. Create a chatty pod
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: counter }
spec:
  restartPolicy: Always
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","i=0; while true; do echo \"$(date) line $i\"; i=$((i+1)); sleep 1; done"]
EOF

kubectl wait --for=condition=Ready pod/counter --timeout=60s
```

### A2. Read logs in different ways
```bash
kubectl logs counter --tail=5
kubectl logs counter -f &
sleep 5
kill %1
```

### A3. Time-bounded
```bash
kubectl logs counter --since=10s
kubectl logs counter --since=1m | wc -l
```

---

## Section B — `--previous` After a Crash

### B1. Force a crash
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: crasher }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sh","-c","echo about to crash; sleep 5; exit 1"]
EOF

# Wait for at least one restart
sleep 30
kubectl get pod crasher
```

### B2. Read current vs previous
```bash
kubectl logs crasher --tail=5
kubectl logs crasher --previous --tail=5
```
**Lesson:** logs from the most recent crashed instance are still on disk until the next rotation.

### B3. Cleanup
```bash
kubectl delete pod crasher
```

---

## Section C — Multi-container Pod

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: multi }
spec:
  containers:
  - name: web
    image: nginx
  - name: helper
    image: busybox
    command: ["sh","-c","while true; do echo helper $(date); sleep 3; done"]
EOF

kubectl wait --for=condition=Ready pod/multi --timeout=60s
kubectl logs multi -c web --tail=3
kubectl logs multi -c helper --tail=3

# Default container
kubectl logs multi
```
**Question:** what error appears with the default container?

Cleanup:
```bash
kubectl delete pod multi
```

---

## Section D — Logs by Selector

```bash
kubectl create deployment web --image=nginx --replicas=3
kubectl wait --for=condition=Ready pods -l app=web --timeout=60s

kubectl logs -l app=web --max-log-requests=5 --prefix --tail=2
```
**Lesson:** `--prefix` adds the pod name to each line so you can tell who said what.

Cleanup:
```bash
kubectl delete deployment web
```

---

## Section E — Inspect On-Disk Files

For minikube:
```bash
minikube ssh
sudo ls -la /var/log/containers/ | head
sudo ls -la /var/log/pods/ | head
sudo cat /var/log/containers/$(ls /var/log/containers/ | head -1) | head -3
exit
```

You should see JSON-per-line entries like:
```json
{"log":"...\n","stream":"stdout","time":"2026-04-25T..."}
```

---

## Section F — Log Rotation

```bash
minikube ssh
sudo grep containerLogMax /var/lib/kubelet/config.yaml
exit
```

Defaults: `containerLogMaxSize: 10Mi`, `containerLogMaxFiles: 5`.

Generate enough output to force a rotation:
```bash
kubectl run blast --image=busybox -- sh -c "yes 'lots of text lots of text lots of text' | head -c 11M; sleep 3600"
sleep 10
minikube ssh
sudo ls -la /var/log/pods/loglab_blast_*/
exit
```
You should see multiple `0.log`, `0.log.<timestamp>` files.

Cleanup:
```bash
kubectl delete pod blast --ignore-not-found
```

---

## Section G — Sidecar Log Tailer Pattern

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: legacy }
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh","-c","mkdir -p /var/log/legacy; while true; do echo \"$(date) request\" >> /var/log/legacy/access.log; sleep 1; done"]
    volumeMounts:
    - { name: logs, mountPath: /var/log/legacy }
  - name: tailer
    image: busybox
    args: [sh, -c, "tail -F /shared/access.log"]
    volumeMounts:
    - { name: logs, mountPath: /shared }
  volumes:
  - { name: logs, emptyDir: {} }
EOF

kubectl wait --for=condition=Ready pod/legacy --timeout=60s
kubectl logs legacy -c tailer --tail=3
```
The tailer's stdout (which is what `kubectl logs` reads) carries the legacy app's file contents.

Cleanup:
```bash
kubectl delete pod legacy
```

---

## Section H — Cleanup
```bash
kubectl delete pod counter --ignore-not-found
kubectl config set-context --current --namespace=default
kubectl delete namespace loglab
```
