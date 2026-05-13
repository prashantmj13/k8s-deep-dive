# 26 — Init Containers | Solutions & Expected Outputs

---

## Exercise 2 — Basic Init Container

```bash
kubectl logs pod-init-basic -c app
```
**Expected output:**
```
Hello from init container
```

The init container wrote to `/shared/message.txt` on the shared `emptyDir` volume. The app container reads it after the init container exits 0.

---

## Exercise 3 — Multiple Sequential Init Containers

```bash
kubectl get pod pod-init-multi -w
```
**Expected output:**
```
NAME             READY   STATUS     RESTARTS   AGE
pod-init-multi   0/1     Init:0/3   0          2s
pod-init-multi   0/1     Init:1/3   0          4s
pod-init-multi   0/1     Init:2/3   0          6s
pod-init-multi   0/1     PodInitializing  0    8s
pod-init-multi   1/1     Running    0          9s
```

Each init container runs for ~2 seconds then exits. The next one starts only after the previous exits 0.

---

## Exercise 4 — Init Container Logs

```bash
kubectl logs pod-init-multi -c init-step1
# Output: Step 1 done

kubectl logs pod-init-multi -c init-step2
# Output: Step 2 done

kubectl logs pod-init-multi -c init-step3
# Output: Step 3 done

kubectl logs pod-init-multi -c app
# Output: App started!
```

---

## Exercise 5 — Failing Init Container

```bash
kubectl get pod pod-init-fail -w
```
**Expected output:**
```
NAME            READY   STATUS                  RESTARTS   AGE
pod-init-fail   0/1     Init:0/1                0          2s
pod-init-fail   0/1     Init:CrashLoopBackOff   1          10s
pod-init-fail   0/1     Init:CrashLoopBackOff   2          25s
```

```bash
kubectl describe pod pod-init-fail
```
**Key section in output:**
```
Init Containers:
  init-fail:
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    1
```

> The app container section shows `State: Waiting / Reason: PodInitializing` — it never starts.

---

## Exercise 6 — Wait for Service

Before service is created:
```bash
kubectl get pod pod-wait-for-svc
```
```
NAME                READY   STATUS     RESTARTS   AGE
pod-wait-for-svc    0/1     Init:0/1   0          30s
```

Init container logs show:
```bash
kubectl logs pod-wait-for-svc -c wait-for-myservice
# Output (repeating):
# waiting...
# nslookup: can't resolve 'myservice.init-lab.svc.cluster.local'
```

After `kubectl create service clusterip myservice --tcp=80:80`:
```
NAME                READY   STATUS    RESTARTS   AGE
pod-wait-for-svc    1/1     Running   0          45s
```

---

## Exercise 7 — Describe Output

```
Init Containers:
  init-writer:
    Container ID:  containerd://abc123...
    Image:         busybox
    Command:
      sh
      -c
      echo "Hello from init container" > /shared/message.txt
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 13 May 2026 10:00:01 +0800
      Finished:     Wed, 13 May 2026 10:00:02 +0800
    Ready:          True
```

---

## Challenge Answers

**1. If second of three fails, do previous ones re-run?**
Yes. When the pod restarts (due to init container failure), **all init containers run again from the beginning**. Init containers are not individually retried — the whole sequence restarts.

**2. Can init containers use same volumes as app containers?**
Yes. Volumes defined in `spec.volumes` are shared across both init and app containers. This is one of the primary use cases — init containers pre-populate data that app containers consume.

**3. Pod status during init containers?**
`Init:N/M` where N = completed init containers, M = total. Example: `Init:1/3` means 1 of 3 done.

**4. Can you set livenessProbe on init container?**
No. Init containers do not support `livenessProbe`, `readinessProbe`, or `startupProbe`. They are only expected to run to completion (exit 0).

**5. Init containers vs sidecar containers?**
- Init containers run **before** app containers and **terminate** when done.
- Sidecar containers run **alongside** app containers for the full pod lifetime (e.g. log shippers, proxies).
- As of Kubernetes 1.29, there is a native sidecar container feature (`restartPolicy: Always` in initContainers) that allows init containers to act as persistent sidecars.
