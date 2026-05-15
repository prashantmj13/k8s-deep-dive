# 56 — Storage Classes | Exercises

> **Cluster:** minikube or GKE/AKS/EKS (cloud StorageClass exercises use GKE examples)
> **Estimated time:** 25 minutes

---

## Exercise 1 — Inspect existing StorageClasses

```bash
kubectl get storageclass
kubectl get sc

# Which one is the default?
kubectl get sc | grep default

# Describe the default SC
kubectl describe sc $(kubectl get sc | grep default | awk '{print $1}')
```

---

## Exercise 2 — Create a custom StorageClass (local path / minikube)

```yaml
# local-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-manual
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
```

```bash
kubectl apply -f local-sc.yaml
kubectl get sc
```

---

## Exercise 3 — Create a PV for the local SC

```yaml
# local-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv-01
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-manual
  hostPath:
    path: /tmp/local-pv-01
```

```bash
kubectl apply -f local-pv.yaml
kubectl get pv local-pv-01
```

> **Observe:** STATUS is `Available` — but will stay that way until a Pod is scheduled (WaitForFirstConsumer).

---

## Exercise 4 — Dynamic PVC using default StorageClass

On minikube the default SC is `standard` (hostpath provisioner):

```yaml
# dynamic-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  # no storageClassName — uses default
```

```bash
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc dynamic-pvc
kubectl get pv   # a new PV should appear automatically!
```

---

## Exercise 5 — Use the dynamic PVC in a pod

```yaml
# pod-dynamic.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-dynamic
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'dynamic storage works!' > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
```

```bash
kubectl apply -f pod-dynamic.yaml
kubectl exec pod-dynamic -- cat /data/test.txt
```

---

## Exercise 6 — Enable volume expansion

```yaml
# expandable-sc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
kubectl apply -f expandable-sc.yaml
kubectl describe sc expandable | grep -E "AllowVolume|ReclaimPolicy|Binding"
```

---

## Exercise 7 — Set a StorageClass as default

```bash
# Remove default from current default
CURRENT_DEFAULT=$(kubectl get sc | grep default | awk '{print $1}')
kubectl patch sc $CURRENT_DEFAULT \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Set local-manual as default
kubectl patch sc local-manual \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl get sc   # local-manual should now show (default)

# Revert
kubectl patch sc local-manual \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch sc $CURRENT_DEFAULT \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete pod pod-dynamic
kubectl delete pvc dynamic-pvc
kubectl delete pv local-pv-01
kubectl delete sc local-manual expandable
```

---

## Challenge Questions

1. What is the difference between `Immediate` and `WaitForFirstConsumer` binding modes?
2. What happens when a PVC doesn't specify a `storageClassName` at all?
3. What does `allowVolumeExpansion: true` enable?
4. Can you change a StorageClass after it's been created?
5. What is the `kubernetes.io/no-provisioner` provisioner used for?
