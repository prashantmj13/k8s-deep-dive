# 62 — Network Policies | Exercises

> **Cluster:** Cluster with Calico, Cilium, or Weave CNI (not Flannel)
> **Estimated time:** 30 minutes
> **Namespace:** `netpol-lab`

---

## Exercise 1 — Verify CNI supports NetworkPolicy

```bash
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|antrea"
```

If none are shown, NetworkPolicy won't be enforced. On GKE, Calico is used by default.

---

## Exercise 2 — Setup: deploy a 3-tier app

```bash
kubectl create namespace netpol-lab
kubectl config set-context --current --namespace=netpol-lab

# Frontend
kubectl run frontend --image=nginx --labels="app=frontend,tier=frontend"
kubectl expose pod frontend --port=80 --name=frontend-svc

# Backend
kubectl run backend --image=nginx --labels="app=backend,tier=backend"
kubectl expose pod backend --port=80 --name=backend-svc

# Database
kubectl run database --image=nginx --labels="app=database,tier=database"
kubectl expose pod database --port=80 --name=database-svc

# Test pod (simulates external/untrusted access)
kubectl run testpod --image=busybox --restart=Never -- sleep 3600

kubectl get pods
```

---

## Exercise 3 — Verify open access (before any policy)

```bash
# All of these should succeed
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
kubectl exec testpod -- wget -qO- --timeout=3 http://database-svc
```

---

## Exercise 4 — Apply deny-all to the namespace

```yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
kubectl apply -f deny-all.yaml

# All of these should now FAIL (timeout)
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
```

---

## Exercise 5 — Allow ingress to frontend from anywhere

```yaml
# allow-frontend-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
  namespace: netpol-lab
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - {}    # allow all ingress
```

```bash
kubectl apply -f allow-frontend-ingress.yaml

# frontend now reachable
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc

# backend still blocked
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
```

---

## Exercise 6 — Allow frontend → backend only

```yaml
# allow-frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: netpol-lab
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
kubectl apply -f allow-frontend-to-backend.yaml

# From frontend pod — should work
kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc

# From testpod — still blocked
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
```

---

## Exercise 7 — Allow DNS egress for all pods

```yaml
# allow-dns-egress.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: netpol-lab
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
kubectl apply -f allow-dns-egress.yaml

# DNS resolution now works inside pods
kubectl exec frontend -- nslookup backend-svc
```

---

## Exercise 8 — Protect database (only backend can reach it)

```yaml
# protect-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: protect-database
  namespace: netpol-lab
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
kubectl apply -f protect-database.yaml

# backend can reach database
kubectl exec backend -- wget -qO- --timeout=3 http://database-svc

# frontend cannot reach database directly
kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc
```

---

## Exercise 9 — List and inspect all policies

```bash
kubectl get networkpolicies
kubectl describe netpol deny-all
kubectl describe netpol allow-frontend-to-backend
```

---

## Exercise 10 — Cleanup

```bash
kubectl delete namespace netpol-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What happens to a pod that matches NO NetworkPolicy selector?
2. Why must you always include a DNS egress rule when blocking all egress?
3. What is the difference between a `podSelector` and a `namespaceSelector` in a `from` block?
4. Can a NetworkPolicy block traffic between containers in the same pod?
5. How do you allow traffic from a specific pod in a specific namespace (AND logic)?
