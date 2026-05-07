# Rolling Update and Rollback — Solutions

## Section A — Five Rollouts
```
$ kubectl rollout history deployment/web
REVISION  CHANGE-CAUSE
1         initial 1.25
2         upgrade 1.26
3         upgrade 1.27
4         upgrade 1.27-alpine
5         upgrade 1.27
6         upgrade 1.28
```
Note revision 5 has the same image as revision 3, but a NEW revision number — Kubernetes increments the counter every time the template changes, even if the result is the same.

## Section B — Rollback to Specific Revision
After `--to-revision=2`, all pods now run `nginx:1.26`. The history grows by one (a new revision N+1 with the same template as 2).

## Section C — Pause/Resume
While paused, `kubectl set` commands modify `spec.template` but no rollout starts. On resume, the controller sees the cumulative diff and ships ONE rollout. This avoids 3 rollout cycles for 3 changes.

## Section D — Stuck Rollout
With a bad image, new pods are stuck in `ImagePullBackOff` indefinitely. The Deployment status shows `Progressing=True` (still trying) but `Available=True` (old pods still serving). After `progressDeadlineSeconds`, status flips to `Progressing=False, Reason=ProgressDeadlineExceeded`.

**Auto-rollback?** No — Kubernetes does NOT auto-rollback. Your CI must check `kubectl rollout status` exit code and call `kubectl rollout undo` on failure.

## Section E — Restart
The annotation `kubectl.kubernetes.io/restartedAt: "<timestamp>"` is added to the pod template. Any change to template triggers a new RS (and rolling update). All pods are gracefully replaced.

## Section F — revisionHistoryLimit
With limit=2, only 2 inactive RSs are retained (oldest deleted automatically). You can roll back to anything in `kubectl rollout history` output, but older revisions are gone.

## Cheat Sheet

| Command | What it does |
|---|---|
| `kubectl set image deployment/X c=Y:tag` | Trigger rollout |
| `kubectl rollout status deployment/X` | Block until done; exit code |
| `kubectl rollout history deployment/X` | List revisions |
| `kubectl rollout history deployment/X --revision=N` | Show specific template |
| `kubectl rollout undo deployment/X` | Roll back one |
| `kubectl rollout undo deployment/X --to-revision=N` | Roll back to N |
| `kubectl rollout pause deployment/X` | Stop reconciling |
| `kubectl rollout resume deployment/X` | One batched rollout |
| `kubectl rollout restart deployment/X` | Force restart all pods |
| `kubectl annotate deployment/X kubernetes.io/change-cause="..."` | Document why |
