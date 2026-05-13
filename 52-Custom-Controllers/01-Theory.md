# 52 — Custom Controllers

## What is a Custom Controller?

A **Custom Controller** is a program that watches Kubernetes resources (built-in or custom) and takes actions to reconcile the **actual state** with the **desired state**. This is the same pattern used by built-in controllers (Deployment controller, ReplicaSet controller, etc.).

---

## The Control Loop (Reconciliation Loop)

```
     Watch for changes
           ↓
    Observe current state
           ↓
    Compare with desired state
           ↓
    Take action to close the gap
           ↓
    Repeat forever
```

This is called a **reconciliation loop** or **control loop**.

---

## Example: Built-in Controllers

| Controller | Watches | Takes Action |
|-----------|---------|-------------|
| ReplicaSet | ReplicaSet + Pods | Creates/deletes pods to match replicas |
| Deployment | Deployment + ReplicaSets | Creates new RS, scales down old RS |
| Node | Node objects | Evicts pods from unhealthy nodes |

Your custom controller follows the same pattern for your own CRDs.

---

## Controller vs Operator

| Term | Meaning |
|------|---------|
| Controller | Watches any resource, reconciles state |
| Operator | Controller + CRD — manages a specific application's lifecycle |

An Operator = CRD (describes desired state) + Controller (makes it happen).

---

## Controller Pattern

```
                    ┌─────────────────┐
                    │  kube-apiserver │
                    └────────┬────────┘
                             │ watch events
                             ▼
                    ┌─────────────────┐
                    │   Informer      │  (caches objects, delivers events)
                    └────────┬────────┘
                             │ add/update/delete events
                             ▼
                    ┌─────────────────┐
                    │   Work Queue    │  (deduplicates, rate-limits)
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   Reconciler    │  (your business logic)
                    │                 │  1. Get current state
                    │                 │  2. Get desired state
                    │                 │  3. Create/update/delete to match
                    └─────────────────┘
```

---

## Writing a Controller (Go, controller-runtime)

```go
// Reconciler interface
type Reconciler interface {
    Reconcile(ctx context.Context, req reconcile.Request) (reconcile.Result, error)
}

// Your implementation
func (r *AppConfigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. Fetch the custom resource
    appConfig := &stablev1.AppConfig{}
    if err := r.Get(ctx, req.NamespacedName, appConfig); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 2. Check actual state (e.g. Deployment)
    deployment := &appsv1.Deployment{}
    err := r.Get(ctx, req.NamespacedName, deployment)

    // 3. Create or update to match desired state
    if errors.IsNotFound(err) {
        // Create deployment
        newDeployment := r.buildDeployment(appConfig)
        return ctrl.Result{}, r.Create(ctx, newDeployment)
    }

    // 4. Update if spec changed
    if deployment.Spec.Replicas != appConfig.Spec.Replicas {
        deployment.Spec.Replicas = appConfig.Spec.Replicas
        return ctrl.Result{}, r.Update(ctx, deployment)
    }

    return ctrl.Result{}, nil
}
```

---

## Scaffolding with kubebuilder

```bash
# Install kubebuilder
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder && mv kubebuilder /usr/local/bin/

# Initialize project
kubebuilder init --domain stable.example.com --repo github.com/me/myoperator

# Create API (CRD + Controller)
kubebuilder create api --group stable --version v1 --kind AppConfig

# Generate manifests (CRD YAML, RBAC)
make manifests

# Run locally
make run

# Build and deploy
make docker-build docker-push IMG=registry.example.com/myoperator:v0.1
make deploy IMG=registry.example.com/myoperator:v0.1
```

---

## Controller Deployment

Controllers run as Deployments inside the cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: appconfig-controller
  namespace: myoperator-system
spec:
  replicas: 1
  template:
    spec:
      serviceAccountName: appconfig-controller-sa
      containers:
      - name: controller
        image: registry.example.com/myoperator:v0.1
```

The controller's service account needs RBAC permissions to watch/create/update the resources it manages.

---

## Key Libraries

| Language | Library |
|----------|---------|
| Go | `controller-runtime`, `client-go` |
| Python | `kopf`, `kubernetes` client |
| Java | `java-operator-sdk` |
| Shell | `shell-operator` |
