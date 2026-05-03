# Multiple Schedulers — Solutions

## Section B — Default Bind
A pod without `schedulerName` ends up with `schedulerName: default-scheduler`. Events from `source.component=default-scheduler` show the binding.

## Section C — Orphan Pod
Pod stays Pending indefinitely with no FailedScheduling event. The default scheduler ignores pods whose `schedulerName` doesn't match its own. **Lesson:** never reference a scheduler that isn't actually running.

## Section D — Second Scheduler
After deploying:
```
$ kubectl get events -n mschedlab --field-selector source.component=my-scheduler
LAST SEEN   TYPE     REASON      OBJECT         MESSAGE
2s          Normal   Scheduled   pod/special    Successfully assigned mschedlab/special to ...
```
Confirming the custom scheduler bound the pod, not the default one.

## Cheat Sheet

| Step | What |
|---|---|
| 1 | Create ServiceAccount in `kube-system` |
| 2 | Bind `system:kube-scheduler` ClusterRole to it |
| 3 | Create KubeSchedulerConfiguration (ConfigMap) with your `schedulerName` |
| 4 | Deploy `kube-scheduler` with `--config=...` |
| 5 | In pod spec: `schedulerName: my-scheduler` |

| Easier alternative | Use **scheduler profiles** in the default kube-scheduler binary (next folder). |
