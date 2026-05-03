# Scheduler Profiles — Solutions

## Section A — Default Behavior
kubeadm doesn't pass `--config` by default. The scheduler uses built-in defaults: spread (LeastAllocated), all standard plugins enabled.

## Section B — Two Profiles
After patching, scheduler logs show:
```
"Profile loaded" name="default-scheduler"
"Profile loaded" name="bin-packer"
```

## Section C — Per-Pod Profile
```
$ kubectl get pod pod-binpack -o jsonpath='{.spec.schedulerName}'
bin-packer
```
Events from `source.component=bin-packer` confirm a different scoring path was used. On a single-node cluster the visible difference is small, but on a multi-node cluster you'd see the bin-packer favor already-loaded nodes.

## Section D — No-Preempt Profile
Disabling `DefaultPreemption` means a pod whose Filter pass fails on every node simply stays Pending. No other pods are evicted to make room.

## Cheat Sheet

| Customization | Where |
|---|---|
| Different scoring strategy | `pluginConfig.NodeResourcesFit.args.scoringStrategy.type` |
| Disable preemption | `plugins.postFilter.disabled: [{name: DefaultPreemption}]` |
| Boost spread | `plugins.score.enabled: [{name: PodTopologySpread, weight: 5}]` |
| Custom plugin | Compile own kube-scheduler binary |

| Why use profiles vs separate scheduler | Lower overhead, no race conditions, shared cache. Use profiles unless you need true isolation. |
