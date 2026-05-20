# 64 — Container Probes | Solutions & Expected Outputs

---

## Exercise 2 — Liveness HTTP probe running healthy

```bash
kubectl describe pod pod-liveness-http | grep -A10 "Liveness"
```
```
Liveness:  http-get http://:80/ delay=5s timeout=2s period=10s #success=1 #failure=3
```

```bash
kubectl get pod pod-liveness-http -w
```
```
NAME                  READY   STATUS    RESTARTS   AGE
pod-liveness-http     1/1     Running   0          30s
pod-liveness-http     1/1     Running   0          60s
```

RESTARTS stays at 0 — nginx is healthy and the probe succeeds every 10 seconds.

---

## Exercise 3 — Liveness failure and restart

```bash
kubectl get pod pod-liveness-fail -w
```
```
NAME                READY   STATUS    RESTARTS   AGE
pod-liveness-fail   1/1     Running   0          5s
pod-liveness-fail   1/1     Running   1          25s    ← restarted!
pod-liveness-fail   1/1     Running   2          50s    ← again
pod-liveness-fail   1/1     Running   3          75s
pod-liveness-fail   0/1     CrashLoopBackOff   3   90s  ← backoff kicks in
```

```bash
kubectl describe pod pod-liveness-fail | grep -E "Liveness|Unhealthy|Killing"
```
```
Liveness:   exec [sh -c exit 1] delay=5s timeout=1s period=5s #success=1 #failure=3
Warning  Unhealthy   Liveness probe failed: command "sh -c exit 1" exited with 1
Warning  Killing     Stopping container app
```

---

## Exercise 4 — Readiness gates traffic

```bash
# During first 15 seconds:
kubectl get pod pod-readiness
```
```
NAME            READY   STATUS    RESTARTS   AGE
pod-readiness   0/1     Running   0          8s    ← 0/1 = not ready
```

```bash
kubectl get endpoints readiness-svc
```
```
NAME             ENDPOINTS   AGE
readiness-svc    <none>      8s    ← no endpoints, no traffic
```

```bash
# After 15 seconds (file created):
kubectl get pod pod-readiness
```
```
NAME            READY   STATUS    RESTARTS   AGE
pod-readiness   1/1     Running   0          18s   ← 1/1 = ready
```

```bash
kubectl get endpoints readiness-svc
```
```
NAME             ENDPOINTS           AGE
readiness-svc    10.244.1.5:80      18s    ← endpoint added, traffic flows
```

---

## Exercise 5 — Startup probe output

```bash
kubectl get pod pod-startup -w
```
```
NAME          READY   STATUS    RESTARTS   AGE
pod-startup   0/1     Running   0          5s    ← startup probe running
pod-startup   0/1     Running   0          15s   ← still starting
pod-startup   0/1     Running   0          25s   ← still starting
pod-startup   1/1     Running   0          30s   ← startup probe passed, liveness takes over
```

```bash
kubectl describe pod pod-startup | grep -A5 "Startup"
```
```
Startup:  exec [test -f /tmp/started] delay=0s timeout=1s period=5s #success=1 #failure=10
```

---

## Exercise 7 — TCP probe output

```bash
kubectl describe pod pod-tcp-probe | grep -A4 "Liveness\|Readiness"
```
```
Liveness:   tcp-socket :80 delay=5s timeout=1s period=10s #success=1 #failure=3
Readiness:  tcp-socket :80 delay=3s timeout=1s period=5s #success=1 #failure=3
```

TCP probe simply tries to open a connection to port 80. nginx is listening → connection succeeds → probe passes.

---

## Challenge Answers

**1. Liveness failure vs readiness failure?**
- **Liveness failure**: Container is considered stuck/broken. Kubernetes **kills and restarts** the container according to the pod's `restartPolicy`.
- **Readiness failure**: Container is considered not ready for traffic. Kubernetes **removes the pod's IP from Service endpoints** — no traffic is routed to it. The container is NOT restarted. Once the readiness probe passes again, the pod is added back to endpoints.

**2. Why NOT check databases in liveness probes?**
If a liveness probe checks an external database and the DB becomes temporarily unavailable, **every pod** with that check will fail liveness and restart simultaneously — a cascade restart that makes the outage worse. Liveness probes should only check internal app health (is this process responsive?). Leave external dependency checks to readiness probes, which just stop traffic rather than restarting pods.

**3. Startup probe vs initialDelaySeconds?**
`initialDelaySeconds` is a **fixed** blind waiting period — you must set it to the maximum possible startup time. If your app sometimes starts in 10s and sometimes 60s, you must set `initialDelaySeconds: 60`, which means liveness failure detection is delayed 60 seconds for every restart.

A startup probe **polls actively** — the moment the app is ready the probe passes (even if it only took 10s), and then liveness kicks in immediately. The `failureThreshold × periodSeconds` gives a maximum startup window while remaining responsive. Best of both worlds.

**4. Pod Running but not receiving traffic?**
The most likely cause is a **failing readiness probe**. The pod is alive (liveness passes) but the readiness check fails — so it is removed from Service endpoints. Traffic goes to other pods only.

Diagnose with:
```bash
kubectl describe pod <pod> | grep -A5 "Readiness"
kubectl get endpoints <service>
kubectl describe pod <pod> | grep -A10 Events
```

**5. successThreshold: 2 on readiness probe?**
The pod must pass the readiness probe **2 consecutive times** before being added back to Service endpoints. This prevents flapping — a pod that alternates between passing and failing won't be added and removed rapidly from the endpoint pool, which would cause intermittent traffic failures. Note: `successThreshold` for **liveness** must always be 1 (you can't require multiple successes to "un-kill" a container).
