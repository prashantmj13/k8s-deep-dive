# Priority Classes — Solutions

## Section A — Built-ins
```
NAME                      VALUE        GLOBAL-DEFAULT
system-cluster-critical   2000000000   false
system-node-critical      2000001000   false
```
Both are above 1B; user pods cannot preempt them.

## Section C — Preemption Events
```
Events:
  Warning  FailedScheduling   default-scheduler   0/1 nodes are available: 1 Insufficient memory.
  Normal   Preempted          default-scheduler   Preempted by prilab/vip on node ... preemption: ...
  Normal   Scheduled          default-scheduler   Successfully assigned prilab/vip to ...
```
The scheduler chose the smallest set of low-priority pods to evict that would create enough room. Evictions are graceful.

## Section D — preemptionPolicy: Never
The pod stays Pending forever (or until something frees up). It doesn't try to evict anyone. Use this for batch/CI jobs that should never disrupt user-facing services.

## Section E — globalDefault
Setting `globalDefault: true` injects this priority into pods that don't specify a class. Only one PriorityClass in the cluster can have this flag. The API server will reject creation of a second.

## Cheat Sheet
| Field | Range | Effect |
|---|---|---|
| `value` | -2³¹ to 1,000,000,000 | scheduler order; preemption threshold |
| `preemptionPolicy: PreemptLowerPriority` | default | will evict to fit |
| `preemptionPolicy: Never` | opt-in | only waits |
| `globalDefault: true` | one allowed | applies to pods without explicit class |
| Built-in `system-*` | >2B | reserved for system components |

| Pod field | Use |
|---|---|
| `spec.priorityClassName` | name of the class |
| `spec.priority` | auto-filled from the class; do not set manually |
