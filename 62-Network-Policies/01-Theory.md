# 62 — Network Policies

## The Default: All Traffic Allowed

By default Kubernetes is a **flat network** — every pod can reach every other pod across all namespaces. This is fine for development but dangerous in production where you want microsegmentation.

A **NetworkPolicy** is a pod-level firewall. It controls:
- What traffic is allowed **into** a pod (ingress)
- What traffic is allowed **out of** a pod (egress)

---

## Prerequisites

NetworkPolicy requires a **CNI plugin that enforces it**:

| CNI Plugin | NetworkPolicy Support |
|-----------|----------------------|
| Calico | ✅ Full support |
| Cilium | ✅ Full support + extended policies |
| Weave | ✅ Full support |
| Antrea | ✅ Full support |
| Flannel | ❌ No enforcement (policies are stored but ignored) |

---

## Core Concept: Default Deny + Allow List

The pattern used in production:

```
1. Deny all ingress and egress for the namespace
2. Explicitly allow only what is needed
```

---

## Policy Selectors — Who the Policy Applies To

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend    # applies to pods labelled app=backend
```

An empty `podSelector: {}` selects **all pods** in the namespace.

---

## Ingress Rules — Who Can Send Traffic In

```yaml
ingress:
- from:
  - podSelector:            # from pods with matching labels
      matchLabels:
        app: frontend
  - namespaceSelector:      # from pods in matching namespaces
      matchLabels:
        env: production
  - ipBlock:                # from specific IP ranges
      cidr: 10.0.0.0/8
      except:
      - 10.1.0.0/16
  ports:
  - protocol: TCP
    port: 8080
```

---

## Egress Rules — Where Pods Can Send Traffic To

```yaml
egress:
- to:
  - podSelector:
      matchLabels:
        app: database
  ports:
  - protocol: TCP
    port: 5432
- to:                      # always allow DNS
  - namespaceSelector: {}  # any namespace
  ports:
  - protocol: UDP
    port: 53
```

> ⚠️ **Always allow UDP 53 (DNS) in egress rules** — without it pods cannot resolve service names.

---

## AND vs OR Logic in Selectors

This is a common source of bugs:

```yaml
# OR — two separate list items
ingress:
- from:
  - namespaceSelector:          # traffic from ANY pod in prod namespace
      matchLabels:
        env: production
  - podSelector:                # OR from any pod labelled role=admin (in same namespace)
      matchLabels:
        role: admin

# AND — one list item with both selectors
ingress:
- from:
  - namespaceSelector:          # traffic from pods that are BOTH in prod namespace
      matchLabels:              # AND have label role=admin
        env: production
    podSelector:
      matchLabels:
        role: admin
```

---

## Complete Example: Three-Tier App

```yaml
# frontend: allow ingress from internet (all), egress only to backend
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes: [Ingress, Egress]
  ingress:
  - {}                          # allow all ingress (internet-facing)
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - port: 8080
  - ports:                      # DNS
    - port: 53
      protocol: UDP

# backend: only from frontend, egress only to database
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - port: 5432
  - ports:
    - port: 53
      protocol: UDP
```

---

## Deny All Policy (Starting Point)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}      # all pods
  policyTypes:
  - Ingress
  - Egress
  # no ingress or egress rules = deny everything
```

---

## Useful Commands

```bash
# List policies
kubectl get networkpolicies -A
kubectl get netpol              # shorthand

# Describe a policy
kubectl describe netpol deny-all -n production

# Test connectivity between pods
kubectl exec podA -- curl --max-time 3 http://podB-svc
kubectl exec podA -- nc -zv podB-ip 8080

# Check if CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|antrea"
```

---

## NetworkPolicy Cheat Sheet

| Scenario | Rule |
|----------|------|
| Allow all ingress | `ingress: [{}]` |
| Allow all egress | `egress: [{}]` |
| Deny all ingress | `policyTypes: [Ingress]` with no ingress rules |
| Deny all egress | `policyTypes: [Egress]` with no egress rules |
| Allow from specific namespace | `namespaceSelector: {matchLabels: {ns: prod}}` |
| Allow from specific pod | `podSelector: {matchLabels: {app: frontend}}` |
| Allow DNS always | `egress: [{ports: [{port: 53, protocol: UDP}]}]` |
