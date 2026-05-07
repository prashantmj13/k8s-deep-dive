# Logging — Solutions

## Section A — Basic
- `--tail=N` shows the last N lines.
- `-f` follows new lines (like `tail -f`).
- `--since=10s` / `--since=1m` filter by recency.

## Section B — Previous
After a crash, `kubectl logs --previous` reads the previous container instance's log file. Useful for "why did my pod crash?" debugging. Without it, you only see the new instance, which may have just started.

## Section C — Multi-container
```
$ kubectl logs multi
error: a container name must be specified for pod multi, choose one of: [web helper]
```
**Solution:** specify with `-c <name>` or use `--all-containers`.

You can also set the **default container** annotation:
```yaml
metadata:
  annotations:
    kubectl.kubernetes.io/default-container: web
```
Then `kubectl logs multi` works without `-c`.

## Section D — Selector
```
[pod/web-7c5d4f9bd-aaa11/nginx] 10.244.0.5 - GET / 200
[pod/web-7c5d4f9bd-bbb22/nginx] 10.244.0.6 - GET / 200
```
The `--prefix` adds `[pod/<name>/<container>]` so lines from different replicas don't get tangled.

## Section E — Node Files
```json
{"log":"2026/04/25 10:30:00 [notice] 1#1: nginx starting\n","stream":"stderr","time":"2026-04-25T10:30:00.123Z"}
```
Three fields: `log` (the line, with trailing `\n`), `stream` (stdout/stderr), `time`.

## Section F — Rotation
```
$ ls /var/log/pods/loglab_blast_xxx/
0.log
0.log.20260425-103045
0.log.20260425-103048
```
Old logs have timestamps appended. Once you exceed `containerLogMaxFiles`, the oldest is deleted.

## Section G — Sidecar
The `tailer` sidecar's stdout becomes the source for `kubectl logs`. The legacy app keeps writing to a file; the sidecar surfaces it. **The cluster-level shipper only needs to know about stdout** — this lifts file-based legacy logs into the standard pipeline.

---

## Cheat Sheet

| Command | Use |
|---|---|
| `kubectl logs <pod>` | last 10 lines (or all, depending on version) |
| `kubectl logs <pod> -f` | follow |
| `kubectl logs <pod> -c <name>` | specific container |
| `kubectl logs <pod> --previous` | previous instance after restart |
| `kubectl logs deploy/X` | one of the deployment's pods |
| `kubectl logs -l app=X --prefix` | all pods matching label |
| `kubectl logs <pod> --since=1h` | time-window |

| Best practice | Why |
|---|---|
| Log to stdout/stderr | only stream the kubelet captures |
| Use a structured format (JSON) | easier for collectors to parse |
| Install fluent-bit DaemonSet | so logs survive pod deletion |
| Set log levels via env var | tunable without rebuild |
| Avoid logging secrets | compliance |
