# 49 — Network Policy | Exercises

> **Cluster:** Cluster with Calico or Cilium CNI (not Flannel)
> **Estimated time:** 25 minutes
> **Namespace:** `netpol-lab`

---

## Exercise 1 — Setup

```bash
kubectl create namespace netpol-lab
kubectl config set-context --current --namespace=netpol-lab

# Deploy frontend
kubectl run frontend --image=nginx --labels=app=frontend
kubectl expose pod frontend --port=80 --name=frontend-svc

# Deploy backend
kubectl run backend --image=nginx --labels=app=backend
kubectl expose pod backend --port=80 --name=backend-svc

# Deploy a test pod
kubectl run testpod --image=busybox --restart=Never -- sleep 3600
```

---

## Exercise 2 — Verify open access (before any policy)

```bash
# testpod should reach both
kubectl exec testpod -- wget -qO- http://frontend-svc
kubectl exec testpod -- wget -qO- http://backend-svc
```

---

## Exercise 3 — Apply deny-all to backend

```yaml
# deny-all-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f deny-all-backend.yaml

# Now this should FAIL (timeout):
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc

# Frontend still accessible:
kubectl exec testpod -- wget -qO- http://frontend-svc
```

---

## Exercise 4 — Allow frontend to reach backend

```yaml
# allow-frontend-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-backend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f allow-frontend-backend.yaml

# From testpod — still blocked:
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc

# From frontend pod — should work:
kubectl exec frontend -- wget -qO- http://backend-svc
```

---

## Exercise 5 — Deny all egress from frontend (except DNS)

```yaml
# deny-egress-frontend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-egress-frontend
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53      # allow DNS only
```

```bash
kubectl apply -f deny-egress-frontend.yaml
kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc
# Should timeout — egress to backend now blocked
```

---

## Exercise 6 — List all network policies

```bash
kubectl get networkpolicies
kubectl describe networkpolicy deny-all-backend
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace netpol-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What happens to a pod that has no NetworkPolicy selecting it?
2. What is the difference between OR and AND in NetworkPolicy `from` rules?
3. Why must you always allow UDP port 53 in egress policies?
4. Can NetworkPolicy block traffic from the same pod?
5. Does NetworkPolicy work without a compatible CNI?
