# 52 — Custom Controllers | Exercises

> **Prerequisites:** The CRD from topic 51 (appconfigs.stable.example.com)
> **Estimated time:** 20 minutes (conceptual + observing built-in controllers)

---

## Exercise 1 — Observe the built-in Deployment controller

```bash
# Create a deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Watch the controller create pods
kubectl get pods -w &
sleep 2

# Delete a pod — watch controller replace it
kubectl delete pod $(kubectl get pods -l app=nginx -o name | head -1)

# The controller immediately creates a replacement
kill %1
```

---

## Exercise 2 — Trace reconciliation events

```bash
kubectl create deployment trace-test --image=nginx --replicas=2

# Watch events — see controller taking action
kubectl get events --sort-by='.lastTimestamp' | grep -E "ScalingReplicaSet|Created|Pulled"

# Now scale and watch events
kubectl scale deployment trace-test --replicas=5
kubectl get events --sort-by='.lastTimestamp' | tail -10
```

---

## Exercise 3 — Observe a CRD-based controller (ArgoCD example)

If ArgoCD is installed:
```bash
kubectl get crds | grep argoproj
kubectl get applications -n argocd
kubectl describe application myapp -n argocd | grep -E "Status:|Health:|Sync:"
```

If not installed, observe CRD controller pattern with a simpler example:
```bash
# List all controllers running in kube-system
kubectl get pods -n kube-system | grep -E "controller|manager"
```

---

## Exercise 4 — Write a simple controller concept (Python + kopf)

This is a conceptual exercise — understand the pattern:

```python
import kopf
import kubernetes

@kopf.on.create('stable.example.com', 'v1', 'appconfigs')
def create_fn(spec, name, namespace, **kwargs):
    replicas = spec.get('replicas', 1)
    image = spec.get('image', 'nginx')

    # Create a Deployment to match the AppConfig
    api = kubernetes.client.AppsV1Api()
    deployment = {
        "apiVersion": "apps/v1",
        "kind": "Deployment",
        "metadata": {"name": name, "namespace": namespace},
        "spec": {
            "replicas": replicas,
            "selector": {"matchLabels": {"app": name}},
            "template": {
                "metadata": {"labels": {"app": name}},
                "spec": {"containers": [{"name": name, "image": image}]}
            }
        }
    }
    api.create_namespaced_deployment(namespace, deployment)
    return {'deployment': name}

@kopf.on.delete('stable.example.com', 'v1', 'appconfigs')
def delete_fn(name, namespace, **kwargs):
    api = kubernetes.client.AppsV1Api()
    api.delete_namespaced_deployment(name, namespace)
```

---

## Exercise 5 — Cleanup

```bash
kubectl delete deployment nginx trace-test 2>/dev/null; true
```

---

## Challenge Questions

1. What is the difference between a controller and an operator?
2. What is an Informer and why is it used instead of polling?
3. What does the Work Queue do in the controller pattern?
4. Why must a controller's service account have RBAC permissions?
5. What happens if a controller crashes mid-reconciliation?
