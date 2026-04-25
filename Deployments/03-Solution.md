# Deployments — Solutions

Expected outputs and explanations for `02-Exercise.md`.

---

## Section A — First Deployment

### A1. The hierarchy
```
$ kubectl get deploy,rs,pods
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment/web        3/3     3            3           5s

NAME                            DESIRED   CURRENT   READY   AGE
replicaset.apps/web-7c8d4f9bd   3         3         3       5s

NAME                          READY   STATUS    RESTARTS   AGE
pod/web-7c8d4f9bd-aaa11        1/1     Running   0          5s
pod/web-7c8d4f9bd-bbb22        1/1     Running   0          5s
pod/web-7c8d4f9bd-ccc33        1/1     Running   0          5s
```
The hash `7c8d4f9bd` is the pod-template-hash. Every pod, the RS, and the RS selector all carry it as a label.

### A4. Labels on pods
```
LABELS
app=web,pod-template-hash=7c8d4f9bd
```
The `pod-template-hash` is what makes the system robust: even if you create another Deployment with the same `app=web`, the RSs will have different hashes and won't fight over each other's pods.

---

## Section B — Rolling Update

### B2. Two RSs after rollout
```
$ kubectl get rs
NAME              DESIRED   CURRENT   READY   AGE
web-7c8d4f9bd     0         0         0       5m       <- old (kept for rollback)
web-9f3e2a1cd     3         3         3       1m       <- new (active)
```
The old RS is scaled to 0 but kept. That is what allows `rollout undo` to be instantaneous.

---

## Section C — History & Rollback

### C1. History
```
$ kubectl rollout history deployment/web
REVISION  CHANGE-CAUSE
1         <none>
2         upgrade nginx 1.25 -> 1.27
```

### C3. Rollback effect
`kubectl rollout undo` does NOT delete the new RS. It scales the new RS to 0 and the previous RS back up. Result:
```
NAME              DESIRED
web-7c8d4f9bd     3        <- back to active
web-9f3e2a1cd     0        <- now inactive
```

---

## Section D — Watching the Dance

Sample sequence with `maxSurge=1, maxUnavailable=0, replicas=4`:
```
old=4 new=0
old=4 new=1   <- surge new
old=3 new=1   <- new ready, drain old
old=3 new=2
old=2 new=2
old=2 new=3
old=1 new=3
old=1 new=4
old=0 new=4   <- done
```
Total never exceeds 5 (maxSurge=1). Total never drops below 4 (maxUnavailable=0).

---

## Section E — Broken Rollout

### E2. Conditions during failure
```
Conditions:
  Type           Status   Reason
  Available      True     MinimumReplicasAvailable     <- old pods still serve
  Progressing    True     ReplicaSetUpdated            <- still trying new RS
```
**Lesson:** the Deployment hasn't given up yet. The old pods continue handling traffic — your service is unaffected from the user's perspective.

### E3. After progressDeadlineSeconds
```
Conditions:
  Type           Status   Reason
  Available      True     MinimumReplicasAvailable
  Progressing    False    ProgressDeadlineExceeded     <- now CI can detect failure
```
**Lesson:** Kubernetes does NOT automatically roll back. Your CI pipeline should:
1. `kubectl set image ...`
2. `kubectl rollout status --timeout=10m` (returns non-zero on failure)
3. On failure: `kubectl rollout undo`

---

## Section F — Pause/Resume

When paused, all changes pile up on `spec.template` but nothing happens. Resume produces ONE rollout that reflects the cumulative diff. This is essential for scripts that make 5 changes — without pause you'd get 5 rollouts.

---

## Section G — Recreate

### G2. Pod sequence with Recreate
```
old: web-7c8d  Running
old: web-7c8d  Terminating
old: web-9f3e  Terminating
old: web-aaa   Terminating
(gap — 0 pods running)
new: web-xxx   Pending
new: web-xxx   ContainerCreating
new: web-xxx   Running
```
The gap is real downtime. Use Recreate only when version-mixing is unsafe.

---

## Section H — Canary

### H2. Traffic split
The Service selects all pods with `app=webcanary`. With 9 stable + 1 canary, each pod receives roughly equal traffic when iptables/IPVS round-robins among Service endpoints. Result: ~10% canary.

This is **probabilistic per-connection** routing, not per-request weighting. For finer control, use a service mesh (Istio VirtualService, Linkerd TrafficSplit).

---

## Section I — Restart Without Image Change

The annotation pattern:
```yaml
spec:
  template:
    metadata:
      annotations:
        kubectl.kubernetes.io/restartedAt: "2026-04-25T15:30:00Z"
```
Any change in the template triggers a new RS, even an annotation. This forces a rolling restart that picks up new ConfigMap/Secret values (since most apps read them only at boot).

---

## Section J — RevisionHistoryLimit

After 5 rollouts with limit 2:
```
$ kubectl get rs
NAME              DESIRED
web-aaa           0  <- revision N-2
web-bbb           0  <- revision N-1
web-ccc           3  <- active (revision N)
```
Older RSs are garbage-collected. Set this to 0 to keep no history (cannot roll back) — only do this for non-critical workloads.

---

## Cheat Sheet

| Command | What it does |
|---|---|
| `kubectl create deployment X --image=Y` | Imperative create |
| `kubectl set image deployment/X c=Y:tag` | Trigger rollout |
| `kubectl rollout status deployment/X` | Block until done; non-zero exit on failure |
| `kubectl rollout history deployment/X` | List revisions |
| `kubectl rollout undo deployment/X [--to-revision=N]` | Roll back |
| `kubectl rollout pause deployment/X` | Stop reconciling |
| `kubectl rollout resume deployment/X` | One batched rollout |
| `kubectl rollout restart deployment/X` | Force restart of all pods |
| `kubectl scale deployment/X --replicas=N` | Manual scale |
| `kubectl autoscale deployment/X --min --max --cpu-percent` | HPA |

Happy hacking!
