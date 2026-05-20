# 64 — Container Probes

## What are Probes?

Kubernetes has no way to know if your application inside a container is actually working — a container can be running but the app inside could be deadlocked, out of memory, or still initialising. **Probes** let you tell Kubernetes how to check the health of your application so it can make intelligent decisions.

---

## Three Types of Probes

| Probe | Question it answers | Action on failure |
|-------|---------------------|-------------------|
| `livenessProbe` | Is the app alive? Is it stuck? | Restart the container |
| `readinessProbe` | Is the app ready to receive traffic? | Remove pod from Service endpoints |
| `startupProbe` | Has the app finished starting up? | Delay liveness/readiness checks until startup completes |

---

## Probe Mechanisms — How to Check

All three probe types support the same four check mechanisms:

### 1. HTTP GET
Most common. Kubernetes sends an HTTP GET request — success if response is 2xx or 3xx:

```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: X-Health-Check
    value: "true"
  scheme: HTTP    # or HTTPS
```

### 2. TCP Socket
Kubernetes tries to open a TCP connection — success if connection is established:

```yaml
tcpSocket:
  port: 5432    # good for databases, message queues
```

### 3. Exec (Command)
Kubernetes runs a command inside the container — success if exit code is 0:

```yaml
exec:
  command:
  - sh
  - -c
  - "redis-cli ping | grep PONG"
```

### 4. gRPC (Kubernetes 1.24+)
Kubernetes calls a gRPC health check endpoint:

```yaml
grpc:
  port: 50051
  service: "my.grpc.Service"    # optional
```

---

## Probe Configuration Parameters

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10   # wait before first probe (app startup time)
  periodSeconds: 10         # how often to probe
  timeoutSeconds: 5         # timeout per probe attempt
  failureThreshold: 3       # consecutive failures before action
  successThreshold: 1       # consecutive successes to mark healthy (liveness must be 1)
```

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `initialDelaySeconds` | 0 | Seconds after container starts before probing begins |
| `periodSeconds` | 10 | How often (in seconds) to perform the probe |
| `timeoutSeconds` | 1 | Seconds after which the probe times out |
| `failureThreshold` | 3 | Consecutive failures before marking unhealthy |
| `successThreshold` | 1 | Consecutive successes to mark healthy again |

---

## 1. Liveness Probe

**Purpose:** Detect when an app is alive but broken (deadlock, memory leak, hung thread). Kubernetes restarts the container.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 20
  failureThreshold: 3
```

**What a good `/healthz` endpoint does:**
- Returns 200 if the app is functioning correctly
- Returns 500 if the app is deadlocked or critically broken
- Should be **fast** (< 1 second) — no heavy logic, no DB calls
- Does NOT check dependencies (that's readiness)

**When NOT to use liveness probes:**
- Avoid checking external dependencies (DB, external API) — if the DB goes down, you don't want all pods restarting in a cascade
- Avoid slow operations that could cause false positives

---

## 2. Readiness Probe

**Purpose:** Signal when the app is ready to accept traffic. Until the probe passes, the pod is removed from all Service endpoints — no traffic is sent to it.

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
  successThreshold: 2    # need 2 consecutive successes to go back to ready
```

**Key difference from liveness:** Readiness failure does NOT restart the container. The pod stays running but gets no traffic. Once it passes again, it returns to the endpoint pool.

**A good `/ready` endpoint checks:**
- Database connection pool has available connections
- Cache is populated and ready
- Downstream service dependencies are reachable
- Internal queues are not overloaded

**Critical use case — rolling deployments:**
Without readiness probes, Kubernetes will send traffic to new pods immediately on deployment, potentially before the app is warm. Readiness probes ensure new pods only get traffic once they're fully ready.

---

## 3. Startup Probe

**Purpose:** Give slow-starting apps extra time before liveness/readiness probes kick in. While the startup probe is running, liveness and readiness are suspended.

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30    # 30 × 10s = 5 minutes max startup time
  periodSeconds: 10
```

**Why not just use `initialDelaySeconds`?**
- `initialDelaySeconds` is fixed — if the app sometimes starts in 10s and sometimes in 90s, you'd have to set it to 90s always (slow failure detection)
- `startupProbe` adapts — it polls until success or `failureThreshold` is hit, then hands off to liveness/readiness

---

## All Three Together

```yaml
containers:
- name: app
  image: myapp:v1.2.3
  ports:
  - containerPort: 8080

  # Wait up to 5 minutes for the app to start
  startupProbe:
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30
    periodSeconds: 10

  # Restart container if app is deadlocked
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 0    # startup probe handles the delay
    periodSeconds: 10
    failureThreshold: 3

  # Remove from service endpoints if not ready
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 0
    periodSeconds: 5
    failureThreshold: 3
    successThreshold: 2
```

---

## Probe Interaction with Pod Lifecycle

```
Container starts
      ↓
startupProbe runs (liveness + readiness suspended)
      ↓ (startupProbe passes)
livenessProbe + readinessProbe run in parallel
      ↓
readinessProbe fails → removed from endpoints (no traffic)
readinessProbe passes → added back to endpoints
livenessProbe fails (3×) → container restarted
```

---

## Exec Probe Examples

```yaml
# Check Redis is responding
livenessProbe:
  exec:
    command: ["redis-cli", "ping"]
  periodSeconds: 10

# Check a file exists (custom health indicator)
readinessProbe:
  exec:
    command: ["test", "-f", "/tmp/app-ready"]
  periodSeconds: 5

# Check PostgreSQL is accepting connections
livenessProbe:
  exec:
    command:
    - sh
    - -c
    - "pg_isready -U postgres"
  periodSeconds: 30
```

---

## TCP Probe Examples

```yaml
# Check MySQL port
livenessProbe:
  tcpSocket:
    port: 3306
  initialDelaySeconds: 30
  periodSeconds: 10

# Check Kafka broker
readinessProbe:
  tcpSocket:
    port: 9092
  periodSeconds: 10
```

---

## Checking Probe Status

```bash
# See probe configuration
kubectl describe pod mypod | grep -A20 "Liveness\|Readiness\|Startup"

# Watch probe failures in events
kubectl describe pod mypod | grep -A5 Events

# Watch pod status during startup
kubectl get pod mypod -w

# Check if pod is in endpoints (readiness)
kubectl get endpoints my-service
```

---

## Common Mistakes

| Mistake | Effect | Fix |
|---------|--------|-----|
| No `initialDelaySeconds` on slow app | Liveness kills app before it starts | Use `startupProbe` or set `initialDelaySeconds` |
| Liveness checks external DB | DB outage causes cascade pod restarts | Liveness should only check app internals |
| `failureThreshold: 1` | One slow response restarts container | Use `failureThreshold: 3` minimum |
| Same path for liveness and readiness | Can't distinguish stuck vs not-ready | Use separate `/healthz` and `/ready` endpoints |
| No readiness probe on rolling deploy | Traffic hits unready pods | Always use readiness in Deployments |
