# 59 — Services | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 30 minutes
> **Namespace:** `svc-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace svc-lab
kubectl config set-context --current --namespace=svc-lab

# Deploy a backend app
kubectl create deployment backend --image=nginx --replicas=3
kubectl label deployment backend app=backend   # ensure label exists

# Verify pods have the label
kubectl get pods --show-labels
```

---

## Exercise 2 — ClusterIP service

```yaml
# clusterip-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-clusterip
  namespace: svc-lab
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f clusterip-svc.yaml
kubectl get svc backend-clusterip
kubectl describe svc backend-clusterip
kubectl get endpoints backend-clusterip
```

> **Observe:** CLUSTER-IP is assigned. Endpoints lists all 3 pod IPs.

---

## Exercise 3 — Test ClusterIP from inside the cluster

```bash
# Spin up a temporary pod and curl the service
kubectl run curl-test --image=curlimages/curl \
  --restart=Never --rm -it \
  -- curl -s http://backend-clusterip | grep title
```

---

## Exercise 4 — NodePort service

```yaml
# nodeport-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  namespace: svc-lab
spec:
  type: NodePort
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f nodeport-svc.yaml
kubectl get svc backend-nodeport

# Get node IP
kubectl get nodes -o wide

# Test from outside (replace with your node IP)
curl http://<NODE-IP>:30080
```

---

## Exercise 5 — LoadBalancer service

```yaml
# lb-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
  namespace: svc-lab
spec:
  type: LoadBalancer
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f lb-svc.yaml
kubectl get svc backend-lb -w
# Wait for EXTERNAL-IP to be assigned (on cloud clusters)
# On minikube: minikube tunnel (in a separate terminal)
```

---

## Exercise 6 — Headless service

```yaml
# headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
  namespace: svc-lab
spec:
  clusterIP: None
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f headless-svc.yaml
kubectl get svc backend-headless

# DNS returns individual pod IPs instead of a VIP
kubectl run dns-test --image=busybox --restart=Never --rm -it \
  -- nslookup backend-headless.svc-lab.svc.cluster.local
```

---

## Exercise 7 — Imperative service creation

```bash
# Create ClusterIP directly
kubectl expose deployment backend \
  --name=backend-quick \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP

kubectl get svc backend-quick
```

---

## Exercise 8 — Endpoints inspection

```bash
# Scale down to see endpoints change
kubectl scale deployment backend --replicas=1
kubectl get endpoints backend-clusterip

# Scale back up
kubectl scale deployment backend --replicas=3
kubectl get endpoints backend-clusterip
```

---

## Exercise 9 — Cleanup

```bash
kubectl delete namespace svc-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between `port`, `targetPort`, and `nodePort`?
2. How does a Service know which pods to route to?
3. What is a headless service and when do you use it?
4. What is the valid range for NodePort values?
5. What component on each node actually implements Service routing?
