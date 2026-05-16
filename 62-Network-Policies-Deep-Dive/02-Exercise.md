# 62 — Network Policies Deep Dive | Exercises

> **Cluster:** Cluster with Calico, Cilium, or Weave CNI
> **Estimated time:** 30 minutes

---

## Exercise 1 — Setup three-tier app

```bash
kubectl create namespace three-tier
kubectl config set-context --current --namespace=three-tier

# Frontend
kubectl run frontend --image=nginx --labels=tier=frontend
kubectl expose pod frontend --port=80 --name=frontend-svc

# Backend
kubectl run backend --image=nginx --labels=tier=backend
kubectl expose pod backend --port=80 --name=backend-svc

# Database
kubectl run database --image=nginx --labels=tier=database
kubectl expose pod database --port=80 --name=database-svc

# Test pod (no tier label)
kubectl run testpod --image=busybox --restart=Never -- sleep 3600
```

---

## Exercise 2 — Verify open access (before policies)

```bash
# All these should succeed
kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc
kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc
kubectl exec testpod  -- wget -qO- --timeout=3 http://backend-svc
kubectl exec testpod  -- wget -qO- --timeout=3 http://database-svc
```

---

## Exercise 3 — Apply default deny to database

```yaml
# db-deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-deny-all
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
```

```bash
kubectl apply -f db-deny-all.yaml

# Now database should be unreachable
kubectl exec testpod  -- wget -qO- --timeout=3 http://database-svc   # blocked
kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc   # blocked
kubectl exec backend  -- wget -qO- --timeout=3 http://database-svc   # blocked
```

---

## Exercise 4 — Allow only backend to reach database

```yaml
# db-allow-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-allow-backend
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f db-allow-backend.yaml

kubectl exec backend  -- wget -qO- --timeout=3 http://database-svc  # allowed
kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc  # still blocked
kubectl exec testpod  -- wget -qO- --timeout=3 http://database-svc  # still blocked
```

---

## Exercise 5 — Apply backend policy (ingress from frontend only)

```yaml
# backend-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
```

```bash
kubectl apply -f backend-policy.yaml

kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc   # allowed
kubectl exec testpod  -- wget -qO- --timeout=3 http://backend-svc   # blocked
```

---

## Exercise 6 — Test AND vs OR selector logic

```yaml
# or-policy.yaml - allows from EITHER condition
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: or-test
  namespace: three-tier
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend    # OR
    - podSelector:
        matchLabels:
          tier: frontend   # any of these can reach db
```

```bash
kubectl apply -f or-policy.yaml
kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc  # now allowed
kubectl exec backend  -- wget -qO- --timeout=3 http://database-svc  # still allowed
kubectl exec testpod  -- wget -qO- --timeout=3 http://database-svc  # still blocked
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace three-tier
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. If a pod is selected by two NetworkPolicies, are the rules ANDed or ORed?
2. What does an empty `podSelector: {}` match?
3. Why must you always include a DNS egress rule when restricting egress?
4. Can NetworkPolicy restrict traffic between pods on the same node?
5. How do you allow all traffic in all directions (override a deny-all)?
