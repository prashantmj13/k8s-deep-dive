# 26 — Init Containers | Exercises

> **Cluster:** Any kubectl-accessible cluster (minikube / kind / GKE)
> **Estimated time:** 20–25 minutes
> **Namespace:** `init-lab`

---

## Exercise 1 — Create namespace

```bash
kubectl create namespace init-lab
kubectl config set-context --current --namespace=init-lab
```

---

## Exercise 2 — Basic Init Container

Create `pod-init-basic.yaml` — an init container that writes a message to a shared volume before the app reads it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-init-basic
  namespace: init-lab
spec:
  initContainers:
  - name: init-writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello from init container" > /shared/message.txt']
    volumeMounts:
    - name: shared
      mountPath: /shared
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /shared/message.txt && sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
```

```bash
kubectl apply -f pod-init-basic.yaml
kubectl logs pod-init-basic -c app
```

---

## Exercise 3 — Multiple Sequential Init Containers

Create `pod-init-multi.yaml` with 3 init containers running in sequence:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-init-multi
  namespace: init-lab
spec:
  initContainers:
  - name: init-step1
    image: busybox
    command: ['sh', '-c', 'echo "Step 1 done" && sleep 2']
  - name: init-step2
    image: busybox
    command: ['sh', '-c', 'echo "Step 2 done" && sleep 2']
  - name: init-step3
    image: busybox
    command: ['sh', '-c', 'echo "Step 3 done" && sleep 2']
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "App started!" && sleep 3600']
```

```bash
kubectl apply -f pod-init-multi.yaml

# Watch the pod progress through init stages
kubectl get pod pod-init-multi -w
```

> **Observe:** The STATUS column will show `Init:0/3`, `Init:1/3`, `Init:2/3`, then `Running`

---

## Exercise 4 — Check Init Container Logs

```bash
kubectl logs pod-init-multi -c init-step1
kubectl logs pod-init-multi -c init-step2
kubectl logs pod-init-multi -c init-step3
kubectl logs pod-init-multi -c app
```

---

## Exercise 5 — Simulate a Failing Init Container

Create `pod-init-fail.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-init-fail
  namespace: init-lab
spec:
  initContainers:
  - name: init-fail
    image: busybox
    command: ['sh', '-c', 'echo "Failing on purpose" && exit 1']
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "This should never run" && sleep 3600']
```

```bash
kubectl apply -f pod-init-fail.yaml
kubectl get pod pod-init-fail -w
kubectl describe pod pod-init-fail
```

> **Observe:** Pod stays in `Init:CrashLoopBackOff`. The app container never starts.

---

## Exercise 6 — Wait for a Service (Dependency Check)

First create a service that doesn't exist yet, then a pod that waits for it:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-wait-for-svc
  namespace: init-lab
spec:
  initContainers:
  - name: wait-for-myservice
    image: busybox
    command: ['sh', '-c', 'until nslookup myservice.init-lab.svc.cluster.local; do echo waiting...; sleep 2; done']
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'echo "Service is up, app starting!" && sleep 3600']
```

```bash
kubectl apply -f pod-wait-for-svc.yaml
kubectl get pod pod-wait-for-svc -w
```

Now create the service to unblock it:

```bash
kubectl create service clusterip myservice --tcp=80:80
kubectl get pod pod-wait-for-svc -w
```

> **Observe:** Once the service exists, the init container succeeds and the app starts.

---

## Exercise 7 — Describe and Inspect Init Container Status

```bash
kubectl describe pod pod-init-basic
```

Look for the `Init Containers:` section. Note the `State`, `Exit Code`, and `Started/Finished` timestamps.

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace init-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What happens if the second of three init containers fails — do the previous ones re-run?
2. Can init containers use the same volumes as app containers?
3. What pod status do you see when init containers are still running?
4. Can you set a `livenessProbe` on an init container?
5. How do init containers differ from sidecar containers?
