# Resource Limits — Solutions

## Section A — OOMKilled
```
Last State:  Terminated
  Reason:    OOMKilled
  Exit Code: 137
  Started:   ...
  Finished:  ...
```
Exit code 137 = 128 + signal 9 (SIGKILL sent by the kernel OOM killer).

## Section B — CrashLoopBackOff
The kubelet restarts after each kill (because `restartPolicy` defaults to `Always`). With each successive restart, the backoff delay doubles: 10s, 20s, 40s, 80s, 160s, 300s (max). The pod's status oscillates between `Running`, `OOMKilled`, and `CrashLoopBackOff`.

## Section C — Cgroup Files
- `memory.max` = the limit in bytes, or `max` if unlimited.
- `cpu.max` = `<quota> <period>` in microseconds. `100000 100000` = 1 full CPU per period. `50000 100000` = half a CPU. `max 100000` = no limit.

## Section D — Throttling
```
nr_periods    1234
nr_throttled  1230            # nearly every period was throttled
throttled_usec 5000000000     # ~5 seconds throttled in this window
```
**Lesson:** the process is alive but spent most of its time waiting. This is what causes mysterious latency in production. CPU limits are silent killers of p99 latency.

## Section E — QoS via Limits
- Equal req/limit on **all** containers AND both CPU+memory → Guaranteed.
- Anything else with at least one resource set → Burstable.
- Nothing set anywhere → BestEffort.

## Section F — LimitRange Enforcement
A LimitRange's `max` is checked at admission. The error message includes which dimension was violated (`maximum cpu usage per Container is 1`), making it easy to diagnose.

## Cheat Sheet
| Limit type | Effect when exceeded |
|---|---|
| `limits.memory` | OOMKilled (exit 137) |
| `limits.cpu` | Throttled, no kill |
| `limits.ephemeral-storage` | Pod evicted |

| Field | Required? | Recommendation |
|---|---|---|
| `requests.cpu` | yes | set to typical usage |
| `requests.memory` | yes | set to typical usage |
| `limits.memory` | yes | 1.5–2× requests |
| `limits.cpu` | optional | only for batch/multi-tenant |
