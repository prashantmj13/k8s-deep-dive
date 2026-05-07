# Commands and Arguments — Solutions

## Section A — Defaults
```
$ kubectl exec defaults -- cat /proc/1/cmdline | tr '\0' ' '
nginx: master process nginx -g daemon off;
```
The image's `ENTRYPOINT` and `CMD` are used as-is.

## Section B — Args only
The container runs the image's ENTRYPOINT plus your args. Final command is `<image-entrypoint> sleep 3600`. For busybox the default entrypoint is essentially `sh`, so the result is `sh sleep 3600` — busybox interprets this and runs `sleep`.

## Section C — Full Override
PID 1 is `sleep 3600` directly. Cleanest approach.

## Section D — Substitution
`$(GREETING) $(NAME)` becomes `hello world` before the args reach the container. Note: this is K8s's substitution, not the shell's. `$VAR` (single dollar) would NOT be expanded by K8s — but the shell `sh -c` would expand it at runtime. Both work here because of the `sh -c`, but `$(VAR)` is the K8s-native way.

## Section E — exec Wrapper
```
$ ps -ef
PID  USER  COMMAND
1    root  sleep 3600         <- PID 1 is the real app
```
The `exec` causes `sh` to be replaced by `sleep`. Signals delivered by the kubelet go to `sleep` directly.

## Section F — Without exec
```
$ ps -ef
PID  USER  COMMAND
1    root  sh -c "echo bad; sleep 3600"
7    root  sleep 3600
```
PID 1 is `sh`. SIGTERM goes to `sh`, which doesn't always propagate. On `kubectl delete pod`, the kubelet will SIGTERM the shell; if shell doesn't forward, the kubelet waits the full grace period and SIGKILLs.

**Fix:** add `exec` before the long-running command.

## Section G — Imperative kubectl run
```yaml
spec:
  containers:
  - command: [sh, -c, "echo hi; sleep 3600"]
```
`--command` puts the args into `command`. Without it, `kubectl run --image=X -- arg1 arg2` puts them into `args`.

---

## Cheat Sheet

| Goal | Field |
|---|---|
| Override ENTRYPOINT | `command` |
| Override CMD only | `args` |
| Run a shell pipeline | `command: [sh, -c]; args: ["..."]` |
| Use env in args | `$(VAR_NAME)` |
| Make app PID 1 in shell wrapper | `exec /bin/app` |
| Imperative override | `kubectl run X --image=Y --command -- ...` |
