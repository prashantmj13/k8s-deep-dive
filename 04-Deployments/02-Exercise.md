# Deployments — Practical Exercises

Hands-on. You will create Deployments, perform rolling updates, watch the controller juggle two ReplicaSets, intentionally break a rollout, roll back, pause/resume, and build a canary by hand.

## Setup

```bash
kubectl create namespace deplab
kubectl config set-context --current --namespace=deplab
```

When done:
```bash
kubectl delete namespace deplab
```

---

## Section A — Create Your First Deployment

### A1. Imperative, then YAML
```bash
kubectl create deployment web --image=nginx:1.25 --replicas=3
kubectl get deploy,rs,pods
```
Notice the chain: 1 Deployment -> 1 ReplicaSet -> 3 Pods. The RS name has a hash suffix (the pod-template-hash).

### A2. Convert to YAML for editing
```bash
kubectl get deployment web -o yaml > web.yaml
# inspect: spec.strategy is missing? Defaults are filled in by the API.
kubectl get deployment web -o yaml | grep -A4 strategy
```

### A3. Inspect status conditions
```bash
kubectl get deployment web -o jsonpath='{.status.conditions}' | python3 -m json.tool
```
You should see `Available=True, Progressing=True, Reason=NewReplicaSetAvailable`.

### A4. Look at the labels on pods
```bash
kubectl get pods -l app=web --show-labels
```
Notice the **pod-template-hash** label. That is what makes the Deployment-controller hierarchy unambiguous.

---

## Section B — Trigger a Rolling Update

### B1. Update the image
Open two terminals.

Terminal 1 (watcher):
```bash
kubectl get pods -l app=web -w
```

Terminal 2:
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout status deployment/web
```

In terminal 1 you'll see the pod churn: new pods Pending -> Running, old pods Terminating, in interleaved order. That's the rolling update.

### B2. Confirm the new RS
```bash
kubectl get rs
# You should now see TWO RSs: one with replicas=3 (new), one with replicas=0 (old)
```

### B3. Confirm both pods carry the new image
```bash
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}{": "}{.spec.containers[0].image}{"\n"}{end}'
```

---

## Section C — Rollout History & Rollback

### C1. See history
```bash
kubectl rollout history deployment/web
```
By default `CHANGE-CAUSE` is `<none>`. Annotate to fix that:
```bash
kubectl annotate deployment web kubernetes.io/change-cause="upgrade nginx 1.25 -> 1.27" --overwrite
kubectl rollout history deployment/web
```

### C2. Inspect a specific revision
```bash
kubectl rollout history deployment/web --revision=1
kubectl rollout history deployment/web --revision=2
```
Each revision shows the pod template at that point.

### C3. Roll back to the previous revision
```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# back to 1.25
```

### C4. Roll forward again, then to a specific revision
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl rollout undo deployment/web --to-revision=1
```

---

## Section D — Watch the Two ReplicaSets Dance

### D1. Set tighter knobs to make the dance visible
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"rollingUpdate":{"maxSurge":1,"maxUnavailable":0}}}}'
kubectl scale deployment web --replicas=4
```

### D2. Trigger a rollout and watch RS replica counts
Terminal 1:
```bash
kubectl get rs -w
```

Terminal 2:
```bash
kubectl set image deployment/web nginx=nginx:1.26
```

You'll see the new RS go 0 -> 1 -> 2 -> 3 -> 4 and the old RS go 4 -> 3 -> 2 -> 1 -> 0. The two together never exceed `replicas + maxSurge = 5`, never drop below `replicas - maxUnavailable = 4`.

---

## Section E — Break a Rollout on Purpose

### E1. Use a bad image
```bash
kubectl set image deployment/web nginx=nginx:does-not-exist
kubectl rollout status deployment/web
# Will hang. Press Ctrl+C.
```

### E2. Inspect
```bash
kubectl get pods -l app=web
# old pods still running; new pods stuck in ImagePullBackOff
kubectl describe deployment web | tail -15
```
Notice the conditions: `Progressing=True (still trying)`, `Available=True (old pods still serving)`.

### E3. Tighten the deadline so the rollout fails properly
```bash
kubectl patch deployment web -p '{"spec":{"progressDeadlineSeconds":60}}'
# wait ~60 seconds
kubectl describe deployment web | grep -A5 Conditions
# Now: Progressing=False, Reason=ProgressDeadlineExceeded
```

### E4. Recover
```bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```
**Lesson:** Kubernetes does not auto-rollback. CI/CD systems use the `progressDeadlineSeconds` + `kubectl rollout undo` pattern.

---

## Section F — Pause and Batch Multiple Changes

### F1. Pause
```bash
kubectl rollout pause deployment/web
```

### F2. Make several changes — none trigger a rollout
```bash
kubectl set image deployment/web nginx=nginx:1.27
kubectl set resources deployment/web -c=nginx --requests=cpu=100m,memory=128Mi --limits=cpu=200m,memory=256Mi
kubectl set env deployment/web LOG_LEVEL=info
kubectl get rs    # still showing the old RS active
```

### F3. Resume — all changes ship as one rollout
```bash
kubectl rollout resume deployment/web
kubectl rollout status deployment/web
kubectl get rs
```
**Lesson:** pause is invaluable when scripting multiple changes from CI to avoid N partial rollouts.

---

## Section G — Recreate Strategy

### G1. Switch to Recreate
```bash
kubectl patch deployment web -p '{"spec":{"strategy":{"type":"Recreate","rollingUpdate":null}}}'
```

### G2. Trigger a rollout
```bash
kubectl get pods -l app=web -w &
kubectl set image deployment/web nginx=nginx:1.25
sleep 30
kill %1
```
**Observe:** ALL old pods reach Terminating before any new pods are created. There is a real downtime gap.

### G3. Switch back
```bash
kubectl patch deployment web --type=merge -p '{"spec":{"strategy":{"type":"RollingUpdate","rollingUpdate":{"maxSurge":"25%","maxUnavailable":"25%"}}}}'
```

---

## Section H — Build a Canary Manually

### H1. Two Deployments, same selector
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-stable }
spec:
  replicas: 9
  selector: { matchLabels: { app: webcanary } }
  template:
    metadata: { labels: { app: webcanary, track: stable } }
    spec: { containers: [{ name: c, image: nginx:1.25 }] }
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web-canary }
spec:
  replicas: 1
  selector: { matchLabels: { app: webcanary } }
  template:
    metadata: { labels: { app: webcanary, track: canary } }
    spec: { containers: [{ name: c, image: nginx:1.27 }] }
---
apiVersion: v1
kind: Service
metadata: { name: webcanary-svc }
spec:
  selector: { app: webcanary }
  ports: [{ port: 80, targetPort: 80 }]
EOF
```

### H2. Hit the Service repeatedly and observe ratio
```bash
for i in $(seq 1 20); do
  kubectl run --rm -it tester$i --image=busybox --restart=Never -- wget -qO- webcanary-svc:80 -S 2>&1 | grep nginx
done
```
About 1 in 10 requests should land on the canary.

### H3. Promote the canary
```bash
kubectl scale deployment web-canary --replicas=10
kubectl scale deployment web-stable --replicas=0
```

### H4. Cleanup
```bash
kubectl delete deployment web-stable web-canary
kubectl delete svc webcanary-svc
```

---

## Section I — Restart Without Changing Image

### I1. Force all pods to restart
```bash
kubectl rollout restart deployment/web
kubectl rollout status deployment/web
```

### I2. Inspect what changed
```bash
kubectl get deployment web -o jsonpath='{.spec.template.metadata.annotations}'
# An annotation kubectl.kubernetes.io/restartedAt=<timestamp> was added
```
**Lesson:** any change to `spec.template` triggers a rollout. The `restart` command exploits this by adding an annotation.

---

## Section J — RevisionHistoryLimit

### J1. Set a low limit
```bash
kubectl patch deployment web -p '{"spec":{"revisionHistoryLimit":2}}'
```

### J2. Push 5 rollouts in a row
```bash
for tag in 1.25 1.26 1.27 1.25-perl 1.27-alpine; do
  kubectl set image deployment/web nginx=nginx:$tag
  kubectl rollout status deployment/web
done
```

### J3. See how many old RSs are kept
```bash
kubectl get rs
```
You should see only 2 inactive (replicas=0) RSs, plus the active one. Older ones were garbage-collected.

---

## Section K — Inspect Deployment Status Conditions

```bash
kubectl get deployment web -o jsonpath='{range .status.conditions[*]}{.type}{"="}{.status}{" reason="}{.reason}{"\n"}{end}'
```

After a successful rollout: `Available=True`, `Progressing=True reason=NewReplicaSetAvailable`.
After a failed rollout: `Progressing=False reason=ProgressDeadlineExceeded`.

---

## Section L — Cleanup

```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace deplab
```

Open `03-Solution.md` for expected outputs and a few "why?" answers.
