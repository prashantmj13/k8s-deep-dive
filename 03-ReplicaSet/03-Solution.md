# ReplicaSet — Solutions

Expected outputs, explanations, and answers for every exercise in `02-Exercise.md`.

---

## Section A — First ReplicaSet

### A2. Sample output
```
$ kubectl get rs
NAME   DESIRED   CURRENT   READY   AGE
web    3         3         3       10s

$ kubectl get pods -l app=web -o wide
NAME         READY   STATUS    RESTARTS   AGE   IP            NODE
web-7m4xz    1/1     Running   0          10s   10.244.0.5    minikube
web-h8r2t    1/1     Running   0          10s   10.244.0.6    minikube
web-n9k4q    1/1     Running   0          10s   10.244.0.7    minikube
```
**Lesson:** pod names use the format `<rs-name>-<5-char-random>`. The hash is generated from the controller's UID so two RSs in the same namespace can't collide.

### A3. Status fields explained
- **replicas** — desired count (from spec)
- **fullyLabeledReplicas** — pods matching ALL selector labels
- **readyReplicas** — pods that pass readiness probe (or have no probe)
- **availableReplicas** — pods that have been Ready for at least `minReadySeconds`

### A4. ownerReferences sample
```json
[
  {
    "apiVersion": "apps/v1",
    "kind": "ReplicaSet",
    "name": "web",
    "uid": "abc-123-def-456",
    "controller": true,
    "blockOwnerDeletion": true
  }
]
```
**Lesson:** `controller: true` means this is the *one* owning controller. `blockOwnerDeletion: true` makes the API hold deletion of the RS until pods are gone (cascade).

---

## Section B — Self-Healing

### B1. Sample watch output
```
NAME         READY   STATUS        RESTARTS   AGE
web-7m4xz    1/1     Terminating   0          5m
web-rx9pl    0/1     Pending       0          0s
web-rx9pl    0/1     ContainerCreating 0      0s
web-rx9pl    1/1     Running       0          3s
```
The replacement appeared in **under 1 second** because the controller's watch loop reacts to delete events instantly.

### B2. Events
```
LAST SEEN   TYPE     REASON              OBJECT             MESSAGE
2s          Normal   SuccessfulDelete    replicaset/web     Deleted pod: web-7m4xz
2s          Normal   SuccessfulCreate    replicaset/web     Created pod: web-rx9pl
```
**Lesson:** every action by every controller is auditable through events.

### B3. Bulk delete
All three pods are recreated **in parallel**, not one at a time. The controller batches creates up to a default cap (~500) per loop iteration.

---

## Section C — Scaling

### C1-C2. Sample output
```
$ kubectl scale rs web --replicas=5
replicaset.apps/web scaled

$ kubectl get rs web
NAME   DESIRED   CURRENT   READY   AGE
web    5         5         5       7m
```
**Lesson:** `kubectl scale` issues a single PATCH on the RS object. The controller does the rest.

### C3. Question — which pods got picked for deletion?
**Answer:** the controller deletes in this priority order:
1. Pods in `Pending` state (haven't started)
2. Pods that are not Ready (failing probes)
3. Younger pods first (newest creationTimestamp)
4. Pods on more crowded nodes

In a healthy cluster where everyone is Running and Ready, you will see the **newest** pods deleted. Verify:
```
$ kubectl get pods -l app=web --sort-by=.metadata.creationTimestamp
NAME         READY   STATUS    AGE
web-aaa11    1/1     Running   8m   <- oldest, kept
web-bbb22    1/1     Running   8m   <- oldest, kept
web-ccc33    1/1     Running   2m   <- newer, deleted
```
**Why this order?** Older pods have likely warmed up caches, opened connections, etc. Killing newer ones is less disruptive.

---

## Section D — Selectors

### D2. Adding extra labels — no churn
```
$ kubectl get pod web-7m4xz --show-labels
NAME         READY   STATUS    LABELS
web-7m4xz    1/1     Running   app=web,env=prod
```
The selector `app=web` still matches. Extra labels never confuse the controller.

### D3. Set-based selectors
`matchExpressions` allows `In`, `NotIn`, `Exists`, `DoesNotExist`. Useful for things like:
- "all pods with `tier=backend` except those with `experimental=true`"
- "all pods with environment label set, regardless of value"

The two forms can be combined; both must match.

---

## Section E — Orphaning

### E1. Output
```
$ kubectl get rs web
NAME   DESIRED   CURRENT   READY   AGE
web    3         3         3       30m

$ kubectl label pod web-7m4xz app-
pod/web-7m4xz unlabeled

$ kubectl get pods --show-labels
NAME         LABELS
web-7m4xz    <none>           <- the orphan, no longer matches
web-h8r2t    app=web
web-n9k4q    app=web
web-x8z2t    app=web          <- new pod created by RS

$ kubectl get rs web
NAME   DESIRED   CURRENT   READY   AGE
web    3         3         3       31m
```
**Lesson:** the RS reports CURRENT=3 because it counts only pods matching the selector. The orphan exists in the namespace but is invisible to the RS.

### E3. Practical use
This is the recommended way to "isolate" a misbehaving pod for debugging without taking down the service:
1. Remove the matching label -> the Service stops routing traffic to it (assuming the Service uses the same selector).
2. The RS spawns a replacement to maintain capacity.
3. You can `kubectl exec`, capture heap dumps, etc.
4. When done: `kubectl delete pod` the orphan.

---

## Section F — Adoption

### F2-F3. Output
```
$ kubectl label pod stranger app=web
pod/stranger labeled

$ kubectl get pods -l app=web
NAME       READY   STATUS    AGE
stranger   1/1     Running   2m
web-h8r2t  1/1     Running   30m
web-n9k4q  1/1     Running   30m
web-x8z2t  1/1     Running   5m
```
4 pods. The RS sees one too many. It deletes one (usually `stranger` because it has the youngest readiness, but selection depends on creation timestamp). After:
```
$ kubectl get pods -l app=web
NAME       READY   STATUS    AGE
web-h8r2t  1/1     Running   30m
web-n9k4q  1/1     Running   30m
web-x8z2t  1/1     Running   5m
```
**Lesson:** adoption sets the `ownerReferences` on the orphan pod first; only then does the controller decide which pods to keep. If the orphan is deemed worthier (older, healthier), it might survive while a younger RS-created pod gets deleted.

---

## Section G — The Big Gotcha

### G2. Question — what image are existing pods running?
**Answer:** still `nginx:1.25`. The RS template now says `nginx:1.27`, but **existing pods are not updated**. The selector still matches them, and the count is right, so the controller does nothing.

```
web-h8r2t: nginx:1.25
web-n9k4q: nginx:1.25
web-x8z2t: nginx:1.25
```

### G3. After deleting the pods
```
web-aaa11: nginx:1.27
web-bbb22: nginx:1.27
web-ccc33: nginx:1.27
```
The RS recreated them — using the **current** template, which has the new image.

**Why this matters:** if you change a Deployment's image, Kubernetes does the right thing — it gracefully drains old pods while bringing up new ones. If you change a bare RS's image, nothing happens until you also delete pods. This is the single biggest reason to use Deployments instead of RSs.

---

## Section H — Cascade vs Orphan

### H1-H2. Behaviors
| Command | RS deleted? | Pods deleted? |
|---|---|---|
| `kubectl delete rs web` (default) | yes | yes (cascade) |
| `kubectl delete rs web --cascade=orphan` | yes | NO, pods keep running |
| `kubectl delete rs web --cascade=foreground` | yes (waits for pods first) | yes |

### H4. Adoption after orphan
```
$ kubectl get pod web-aaa11 -o jsonpath='{.metadata.ownerReferences[*].name}'
web        # the new RS adopted the existing pod
```
**Lesson:** `--cascade=orphan` followed by re-applying the RS is essentially a "preserve pods through manifest changes" trick — but Deployments do this much more elegantly.

---

## Section I — Conflicting Selectors

### I2. Chaos pattern
You will see something like:
```
NAME            DESIRED   CURRENT   READY
rs-a            2         2         1
rs-b            2         2         1
```
And pods constantly being created/deleted. Why?

- `rs-a` reads pods with `app=conflict` and counts 4 (its 2 + rs-b's 2). It thinks it has too many. Deletes 2.
- `rs-b` does the same. Deletes 2.
- Both controllers respond by creating 2 more (`SuccessfulCreate`).
- Repeat forever.

**Lesson:** controllers must not have overlapping selectors. The Kubernetes API server has no validation for this — it relies on the user to design unique selectors. Deployments add a `pod-template-hash` label to the RS selector to prevent this even between Deployments.

---

## Section J — Bulk Scale

### J1. Time
```
$ time kubectl scale rs web --replicas=30
real    0m0.143s
```
The PATCH is fast; the actual pods take seconds to start because of image pulls and the scheduler's work. Watching `kubectl get pods -l app=web -w` you will see them ramp up in parallel.

### J2. Scale to 0
This is the cleanest way to "pause" a workload. The RS object remains; pods are gone. No manifest change required. Useful for:
- Cost savings during idle hours
- Investigating an issue without losing the deployment shape
- Maintenance windows

---

## Section K — End-to-End

Sample event stream (reformatted):
```
0s    Normal  ScalingReplicaSet  rs/web    Scaled up to 3
0s    Normal  SuccessfulCreate   rs/web    Created pod: web-7m4xz
0s    Normal  SuccessfulCreate   rs/web    Created pod: web-h8r2t
0s    Normal  SuccessfulCreate   rs/web    Created pod: web-n9k4q
30s   Normal  ScalingReplicaSet  rs/web    Scaled up to 4
30s   Normal  SuccessfulCreate   rs/web    Created pod: web-rx9pl
1m    Normal  Killing            pod/web-7m4xz  Stopping container web
1m    Normal  SuccessfulCreate   rs/web    Created pod: web-aaa11
2m    Normal  ScalingReplicaSet  rs/web    Scaled down to 2
2m    Normal  SuccessfulDelete   rs/web    Deleted pod: web-rx9pl
2m    Normal  SuccessfulDelete   rs/web    Deleted pod: web-aaa11
3m    Normal  SuccessfulDelete   rs/web    Deleted pod: web-h8r2t
3m    Normal  SuccessfulDelete   rs/web    Deleted pod: web-n9k4q
```
Every line tells you who did what.

---

## Section L — Stretch

### L1. API calls
You will see PATCH and PUT requests against `/apis/apps/v1/namespaces/rslab/replicasets/web/scale` — the special `/scale` subresource. This is why scaling can be authorized separately from editing the full RS spec.

### L2. Raw API
Every Kubernetes object is JSON over HTTPS. `kubectl get --raw` returns it unfiltered.

### L3. Drain
When you drain a node, the eviction API marks the pods on that node for deletion. The RS controller sees pods missing and creates replacements on other nodes. If those nodes have capacity, your service stays up the whole time.

### L4. Immutable selector
```
The ReplicaSet "web" is invalid: spec.selector: Invalid value: ...
field is immutable
```
**Lesson:** selectors cannot change because that would break the ownership relationship with all existing pods. To change the selector, you must delete and recreate.

### L5. Controller logs
```
I0425 ... replicaset_controller.go:548] "Too few replicas" replicaSet="rslab/web" need=3 creating=1
I0425 ... replicaset_controller.go:583] "Too many replicas" replicaSet="rslab/web" need=2 deleting=1
```
The controller logs every reconciliation decision.

---

## Cheat Sheet — What You Saw

| Concept | Command that proved it |
|---|---|
| RS creates pods to match desired count | `kubectl get rs && kubectl get pods -l app=web` |
| Self-healing | `kubectl delete pod` then watch a new one appear |
| Pod ownership | `kubectl get pod -o jsonpath='{.metadata.ownerReferences}'` |
| Scaling priority order | `kubectl scale --replicas=2` and check creation timestamps |
| Selector matches by label | `kubectl label pod app-` to orphan |
| Adoption | label a pod -> RS adopts it |
| Image change does NOT roll | `kubectl get pods -o jsonpath='...image...'` after edit |
| Cascade delete | `kubectl delete rs` removes pods |
| Orphan delete | `--cascade=orphan` keeps pods alive |
| Selector is immutable | `kubectl edit rs` rejects selector changes |

---

## What's Next

You now understand the unit that powers Deployments. Recommended next folders:

- **Deployments** — the rolling-update layer on top of RS
- **DaemonSets** — "one pod per node"
- **StatefulSets** — pods with stable identity for databases
- **Jobs / CronJobs** — run-to-completion workloads
- **Services** — how clients reach the pods an RS creates

Happy hacking!
