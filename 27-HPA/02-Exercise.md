# 27 — HPA | Exercises

> **Cluster:** minikube or any cluster with Metrics Server installed
> **Estimated time:** 25–30 minutes
> **Namespace:** `hpa-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace hpa-lab
kubectl config set-context --current --namespace=hpa-lab

# Verify metrics-server is running
kubectl get deployment metrics-server -n kube-system
# If not present on minikube:
# minikube addons enable metrics-server
```

---

## Exercise 2 — Deploy a CPU-intensive app

Create `deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: hpa-lab
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: hpa-lab
spec:
  ports:
  - port: 80
  selector:
    app: php-apache
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

---

## Exercise 3 — Create HPA imperatively

```bash
kubectl autoscale deployment php-apache \
  --cpu-percent=50 \
  --min=1 \
  --max=5

kubectl get hpa
```

> **Note:** Initially the TARGETS column may show `<unknown>/50%` — wait 1–2 minutes for Metrics Server to collect data.

---

## Exercise 4 — Create HPA declaratively (v2)

Delete the old HPA and recreate with YAML:

```bash
kubectl delete hpa php-apache
```

Create `hpa.yaml`:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
  namespace: hpa-lab
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

```bash
kubectl apply -f hpa.yaml
kubectl describe hpa php-apache-hpa
```

---

## Exercise 5 — Generate load and watch scaling

Open a second terminal and run a load generator:

```bash
kubectl run load-gen \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache.hpa-lab.svc.cluster.local; done"
```

In your first terminal, watch HPA and pods:

```bash
kubectl get hpa php-apache-hpa -w
kubectl get pods -w
```

> **Observe:** CPU usage climbs above 50%, HPA scales up replicas. May take 1–2 minutes.

---

## Exercise 6 — Stop load and watch scale down

```bash
kubectl delete pod load-gen
```

Watch the HPA scale back down (takes ~5 minutes due to stabilization window):

```bash
kubectl get hpa php-apache-hpa -w
```

---

## Exercise 7 — Add scale behaviour

Update `hpa.yaml` to add faster scale-up and slower scale-down:

```yaml
spec:
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
```

```bash
kubectl apply -f hpa.yaml
kubectl describe hpa php-apache-hpa
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete namespace hpa-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What happens if you manually `kubectl scale deployment php-apache --replicas=3` while HPA is active?
2. Why must pods have resource `requests` set for HPA to work?
3. What is the default stabilization window for scale-down and why does it exist?
4. What is the difference between `Utilization` and `AverageValue` target types?
5. Can HPA scale a StatefulSet?
