# 52 — Custom Controllers | Solutions

---

## Exercise 1 — Controller replaces deleted pod

```
NAME                    READY   STATUS    RESTARTS
nginx-xxx-aaa           1/1     Running   0
nginx-xxx-bbb           1/1     Running   0
nginx-xxx-ccc           1/1     Running   0

nginx-xxx-aaa           1/1     Terminating   0     ← you deleted it
nginx-xxx-ddd           0/1     Pending       0     ← controller created replacement
nginx-xxx-ddd           1/1     Running       0
```

The ReplicaSet controller detects desired=3, actual=2, creates one new pod within milliseconds.

---

## Exercise 2 — Scaling events

```
REASON              MESSAGE
ScalingReplicaSet   Scaled up replica set trace-test-xxx to 2
ScalingReplicaSet   Scaled up replica set trace-test-xxx to 5
```

---

## Challenge Answers

**1. Controller vs Operator?**
A controller is any program that runs a reconciliation loop over Kubernetes resources. An operator is a controller that also includes a CRD — it packages domain knowledge about a specific application (e.g. a PostgreSQL operator knows how to do backups, failover, scaling). All operators are controllers, but not all controllers are operators.

**2. What is an Informer?**
An Informer is a client-side caching layer that watches the API server for changes to a resource type. It maintains a local cache so controllers can read object state without hitting the API server on every reconcile. It delivers events (Added, Modified, Deleted) to the controller via callbacks. Using an Informer is much more efficient than polling the API server every second.

**3. Work Queue role?**
The Work Queue receives events from Informers and deduplicate them — if an object changes 10 times in one second, only one reconcile is triggered. It also rate-limits and retries failed reconciliations with exponential backoff. This prevents overwhelming the API server and ensures eventual consistency.

**4. Controller RBAC permissions?**
The controller runs as a pod with a service account. When it calls the Kubernetes API (to get/create/update resources), the API server checks RBAC. Without proper permissions the controller gets `Forbidden` errors and can't do its job. Kubebuilder auto-generates the RBAC rules needed based on `//+kubebuilder:rbac` markers in the code.

**5. Crash mid-reconciliation?**
Reconciliation is designed to be **idempotent** — running the same reconcile multiple times produces the same result. When the controller restarts, it re-processes all objects from the Informer cache. If an object was partially updated, the next reconcile will detect the drift and finish the job. This is why the control loop pattern is so resilient.
