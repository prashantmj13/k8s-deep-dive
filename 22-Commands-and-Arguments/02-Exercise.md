# Commands and Arguments — Exercises

## Setup
```bash
kubectl create namespace cmdlab
kubectl config set-context --current --namespace=cmdlab
```

---

## Section A — Defaults (No command, No args)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: defaults }
spec:
  containers:
  - { name: c, image: nginx }
EOF

kubectl wait --for=condition=Ready pod/defaults --timeout=60s
kubectl exec defaults -- cat /proc/1/cmdline | tr '\0' ' '
echo
```
The image runs its default `nginx -g daemon off;`.

---

## Section B — Override args Only

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: only-args }
spec:
  containers:
  - name: c
    image: busybox
    args: ["sleep", "3600"]
EOF

kubectl wait --for=condition=Ready pod/only-args --timeout=60s
kubectl exec only-args -- ps -ef
```
PID 1 is `sleep 3600` because busybox's default ENTRYPOINT is `sh`, and our args replaced the default CMD (which was just `sh` invoking `sh`). Actually busybox is special — see notes.

---

## Section C — Override command and args

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: full-override }
spec:
  containers:
  - name: c
    image: busybox
    command: ["sleep"]
    args: ["3600"]
EOF

kubectl wait --for=condition=Ready pod/full-override --timeout=60s
kubectl exec full-override -- ps -ef | head
```
PID 1 is now `sleep 3600`.

---

## Section D — Variable Substitution in args

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: subst }
spec:
  containers:
  - name: c
    image: busybox
    env:
    - name: GREETING
      value: hello
    - name: NAME
      value: world
    command: ["sh","-c"]
    args: ["echo $(GREETING) $(NAME); sleep 3600"]
EOF

kubectl logs subst | head
```
You should see `hello world`. K8s expanded `$(GREETING)` and `$(NAME)` before running.

---

## Section E — Shell Pipeline With exec

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: pipeline }
spec:
  containers:
  - name: c
    image: alpine
    command: ["sh","-c"]
    args: ["echo starting up; exec sleep 3600"]
EOF

kubectl wait --for=condition=Ready pod/pipeline --timeout=60s
kubectl exec pipeline -- ps -ef
```
PID 1 is `sleep 3600`, NOT `sh`, thanks to `exec`. Without `exec`, the shell would stay as PID 1.

---

## Section F — Without exec (the bug)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: bad-pid1 }
spec:
  containers:
  - name: c
    image: alpine
    command: ["sh","-c"]
    args: ["echo bad; sleep 3600"]   # no exec
EOF

kubectl wait --for=condition=Ready pod/bad-pid1 --timeout=60s
kubectl exec bad-pid1 -- ps -ef
```
PID 1 is `sh`, and `sleep` is PID 7 or so. SIGTERM goes to `sh`, which may not propagate to `sleep`. This is a common signal-handling bug in containers.

---

## Section G — Imperative kubectl run

```bash
kubectl run imp --image=busybox --command -- sh -c "echo hi; sleep 3600"
kubectl get pod imp -o yaml | grep -A4 'command\|args'
```
The `--command` flag tells `kubectl run` to put everything after `--` into `spec.containers[0].command`. Without `--command`, it goes into `args`.

---

## Section H — Cleanup
```bash
kubectl delete pod defaults only-args full-override subst pipeline bad-pid1 imp --ignore-not-found
kubectl config set-context --current --namespace=default
kubectl delete namespace cmdlab
```
