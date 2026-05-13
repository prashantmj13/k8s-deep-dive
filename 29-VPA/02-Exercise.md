# 29 — VPA | Exercises

> **Cluster:** Any cluster (VPA must be installed separately)
> **Estimated time:** 25–30 minutes
> **Namespace:** `vpa-lab`

---

## Exercise 1 — Install VPA

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify VPA components
kubectl get pods -n kube-system | grep vpa
```

Expected pods:
- `vpa-recommender-xxx`
- `vpa-updater-xxx`
- `vpa-admission-controller-xxx`

---

## Exercise 2 — Deploy a test app with deliberately wrong resources

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hamster
  namespace: vpa-lab
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hamster
  template:
    metadata:
      labels:
        app: hamster
    spec:
      containers:
      - name: hamster
        image: registry.k8s.io/ubuntu-slim:0.1
        resources:
          requests:
            cpu: 100m       # intentionally too low
            memory: 50Mi    # intentionally too low
        command: ["/bin/sh"]
        args:
        - "-c"
        - "while true; do timeout 0.5 yes >/dev/null; sleep 0.5; done"
```

```bash
kubectl create namespace vpa-lab
kubectl config set-context --current --namespace=vpa-lab
kubectl apply -f deployment.yaml
kubectl get pods
```

---

## Exercise 3 — Create VPA in Off mode (recommendations only)

Create `vpa-off.yaml`:

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hamster-vpa
  namespace: vpa-lab
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hamster
  updatePolicy:
    updateMode: "Off"
  resourcePolicy:
    containerPolicies:
    - containerName: hamster
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
```

```bash
kubectl apply -f vpa-off.yaml
kubectl get vpa
```

---

## Exercise 4 — Wait for recommendations

Wait 5–10 minutes for the Recommender to gather data, then:

```bash
kubectl describe vpa hamster-vpa
```

Look for the `Recommendation` section with `Target`, `Lower Bound`, `Upper Bound`.

---

## Exercise 5 — Switch to Auto mode

Update the VPA to auto-apply recommendations:

```bash
kubectl patch vpa hamster-vpa --type='merge' \
  -p '{"spec":{"updatePolicy":{"updateMode":"Auto"}}}'
```

Watch pods being evicted and recreated with new resource values:

```bash
kubectl get pods -w
kubectl describe pod <new-pod-name> | grep -A6 Requests
```

> **Observe:** New pods have updated CPU/memory requests set by VPA.

---

## Exercise 6 — Check VPA status

```bash
kubectl get vpa hamster-vpa -o yaml | grep -A20 status
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace vpa-lab
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-down.sh
```

---

## Challenge Questions

1. What is the difference between VPA `Auto` mode and `Initial` mode?
2. Why should you not run VPA and HPA both on CPU for the same deployment?
3. What VPA component is responsible for evicting pods to apply recommendations?
4. How long should you wait before VPA recommendations are reliable?
5. What happens to pods with `updateMode: Off` — does VPA change anything?
