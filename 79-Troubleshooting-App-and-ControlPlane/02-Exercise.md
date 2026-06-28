# 79 — Troubleshooting App and Control Plane | Exercises

> **Cluster:** kubeadm cluster (some exercises need control-plane node access)
> **Estimated time:** 35 minutes
> **Namespace:** `trouble-lab`

---

## Exercise 1 — Diagnose a CrashLoopBackOff

```bash
kubectl create namespace trouble-lab
kubectl config set-context --current --namespace=trouble-lab

kubectl run crash-pod --image=busybox -- sh -c "echo starting; exit 1"
kubectl get pod crash-pod -w
```

```bash
kubectl describe pod crash-pod | grep -A5 "Last State"
kubectl logs crash-pod --previous
```

---

## Exercise 2 — Diagnose ImagePullBackOff

```bash
kubectl run bad-image --image=nginx:this-tag-does-not-exist-12345
kubectl describe pod bad-image | grep -A5 "Events"
```

---

## Exercise 3 — Diagnose a Pending pod (resource request too high)

```bash
kubectl run huge-pod --image=nginx --requests='cpu=1000,memory=1000Gi' 2>/dev/null || \
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: huge-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "1000"
        memory: "1000Gi"
EOF

kubectl get pod huge-pod
kubectl describe pod huge-pod | grep -A5 Events
```

---

## Exercise 4 — Diagnose OOMKilled

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-pod
spec:
  containers:
  - name: app
    image: polinux/stress
    resources:
      limits:
        memory: "50Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod oom-pod -w
kubectl describe pod oom-pod | grep -A5 "Last State"
```

---

## Exercise 5 — Check control plane component health

```bash
kubectl get pods -n kube-system | grep -E "apiserver|etcd|scheduler|controller-manager"
kubectl get componentstatuses 2>/dev/null
kubectl get --raw='/healthz?verbose'
```

---

## Exercise 6 — Check certificate expiry (on control plane node)

```bash
sudo kubeadm certs check-expiration
```

---

## Exercise 7 — Simulate scheduler being down (advanced, careful)

```bash
# On the control plane node — temporarily move the scheduler manifest out
sudo mv /etc/kubernetes/manifests/kube-scheduler.yaml /tmp/

# Create a new pod — it should stay Pending forever (no scheduler to assign it)
kubectl run no-scheduler-test --image=nginx
sleep 10
kubectl get pod no-scheduler-test

# Restore the scheduler
sudo mv /tmp/kube-scheduler.yaml /etc/kubernetes/manifests/
sleep 15
kubectl get pod no-scheduler-test
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace trouble-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What's the first command you should always run when a pod isn't behaving as expected?
2. What does exit code 137 typically indicate?
3. Why would a pod stay Pending indefinitely with no events at all about scheduling?
4. If `kubectl get pods` works but new pods never get scheduled, which control plane component is the likely suspect?
5. What single command checks all kubeadm-managed certificate expiry dates at once?

---

# Solutions

**Exercise 1:**
```
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
```
```bash
kubectl logs crash-pod --previous
# starting
```

**Exercise 2:**
```
Warning  Failed  Failed to pull image "nginx:this-tag-does-not-exist-12345":
  rpc error: code = NotFound desc = failed to pull and unpack image:
  ... manifest unknown
```

**Exercise 3:**
```
Warning  FailedScheduling  0/2 nodes are available: 2 Insufficient cpu, 2 Insufficient memory.
```

**Exercise 4:**
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

**Exercise 7:** While the scheduler manifest is removed, `no-scheduler-test` stays `Pending` indefinitely with `spec.nodeName` empty — no events fire because nothing is trying (and failing) to schedule it; the scheduler itself simply isn't running to even attempt assignment. After restoring the manifest, the pod is scheduled and becomes `Running` within seconds.

## Challenge Answers

**1. First command for misbehaving pod?**
`kubectl describe pod <name>` — the Events section at the bottom almost always points directly at the root cause (scheduling failure, image pull error, probe failure, volume mount issue) before you even need to check logs.

**2. Exit code 137?**
128 + 9 = SIGKILL. In Kubernetes this almost always means the container was forcibly killed — most commonly by the OOM killer when it exceeded its memory limit (`OOMKilled` reason), but can also result from a manual `kubectl delete --force` or node-level resource pressure eviction.

**3. Pending with literally no events?**
This typically means the **scheduler itself isn't running** — if the scheduler is down, it never even attempts to place the pod, so there's no "FailedScheduling" event (that event only fires when the scheduler tries and fails). Contrast with a resource-constrained Pending pod, which DOES get repeated FailedScheduling events because the scheduler is actively retrying.

**4. Pods never scheduled, kubectl otherwise works?**
The **kube-scheduler**. Since the API server is clearly up (kubectl works) and existing pods run fine (kubelet is fine), but new pods never get a `nodeName` assigned, the scheduler component is the prime suspect — check `kubectl get pods -n kube-system | grep scheduler` and its logs.

**5. Check all cert expiry?**
`kubeadm certs check-expiration` — lists every kubeadm-managed certificate (apiserver, etcd, kubelet client/server certs, kubeconfig embedded certs) with its expiry date and residual time in one table.
