# Multiple Schedulers — Practical Exercises

## Setup
```bash
kubectl create namespace mschedlab
kubectl config set-context --current --namespace=mschedlab
```

---

## Section A — Inspect the Default Scheduler

```bash
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl get pods -n kube-system -l component=kube-scheduler -o yaml | grep -A5 command
```

---

## Section B — Watch Which Scheduler Bound a Pod

```bash
kubectl run regular --image=nginx
kubectl get pod regular -o jsonpath='{.spec.schedulerName}{"\n"}'
# default-scheduler
kubectl get events --field-selector source.component=default-scheduler --sort-by='.lastTimestamp' | tail -5
kubectl delete pod regular
```

---

## Section C — Submit a Pod to a Non-existent Scheduler

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: orphan }
spec:
  schedulerName: my-scheduler         # not running
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod orphan
# Pending
kubectl describe pod orphan | tail -8
# No FailedScheduling event from default-scheduler — because default-scheduler isn't watching this pod
```
**Lesson:** if no scheduler with that name exists, the pod sits forever with no error.

Cleanup:
```bash
kubectl delete pod orphan
```

---

## Section D — Deploy a Second Scheduler (same binary, different name)

This is more involved; it requires RBAC, a config, and a Deployment.

### D1. ServiceAccount + ClusterRoleBinding
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata: { name: my-scheduler, namespace: kube-system }
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: my-scheduler-binding }
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata: { name: my-scheduler-volume-binding }
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:volume-scheduler
subjects:
- kind: ServiceAccount
  name: my-scheduler
  namespace: kube-system
EOF
```

### D2. Config (KubeSchedulerConfiguration)
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata: { name: my-scheduler-config, namespace: kube-system }
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
    - schedulerName: my-scheduler
    leaderElection:
      leaderElect: false
EOF
```

### D3. Deployment (uses the same kube-scheduler image)
```bash
KUBE_VER=$(kubectl version -o json | python3 -c "import json,sys;print(json.load(sys.stdin)['serverVersion']['gitVersion'])")
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector: { matchLabels: { component: my-scheduler } }
  template:
    metadata: { labels: { component: my-scheduler } }
    spec:
      serviceAccountName: my-scheduler
      containers:
      - name: kube-scheduler
        image: registry.k8s.io/kube-scheduler:$KUBE_VER
        command:
        - kube-scheduler
        - --config=/etc/kubernetes/config.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/kubernetes
      volumes:
      - name: config
        configMap: { name: my-scheduler-config }
EOF

kubectl rollout status deploy/my-scheduler -n kube-system
```

### D4. Submit a pod to it
```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata: { name: special }
spec:
  schedulerName: my-scheduler
  containers: [{ name: c, image: nginx }]
EOF

kubectl get pod special -o wide
kubectl get pod special -o jsonpath='{.spec.schedulerName}{"\n"}'
kubectl get events --field-selector source.component=my-scheduler --sort-by='.lastTimestamp' | tail -5
```

### D5. Cleanup
```bash
kubectl delete pod special
kubectl delete deploy -n kube-system my-scheduler
kubectl delete cm -n kube-system my-scheduler-config
kubectl delete sa -n kube-system my-scheduler
kubectl delete clusterrolebinding my-scheduler-binding my-scheduler-volume-binding
```

---

## Section E — Cleanup
```bash
kubectl config set-context --current --namespace=default
kubectl delete namespace mschedlab
```
