# 49 — Network Policy

## What is a Network Policy?

By default, **all pods can communicate with all other pods** in a Kubernetes cluster (and even across namespaces). A NetworkPolicy is a firewall rule for pods — it controls which pods can talk to which other pods on which ports.

---

## Prerequisites

NetworkPolicy requires a **CNI plugin that supports it**:
- ✅ Calico, Cilium, Weave, Antrea
- ❌ Flannel (no NetworkPolicy support by default)

---

## Default Behaviour (no NetworkPolicy)

```
Pod A ──────────────────────────────▶ Pod B  (allowed)
Pod A ──────────────────────────────▶ Pod C  (allowed)
Any pod ──────────────────────────── Any pod (all allowed)
```

---

## How NetworkPolicy Works

A NetworkPolicy selects pods using `podSelector` and defines:
- **ingress** rules — what traffic is allowed IN to the selected pods
- **egress** rules — what traffic is allowed OUT from the selected pods

**If a pod is selected by at least one NetworkPolicy, all traffic not explicitly allowed is DENIED.**

---

## Basic Deny-All Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}        # selects ALL pods in namespace
  policyTypes:
  - Ingress
  - Egress
```

This blocks all inbound and outbound traffic for every pod in `production`.

---

## Allow Specific Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend          # applies to backend pods
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend     # only allow from frontend pods
    ports:
    - protocol: TCP
      port: 8080
```

---

## Allow from a Namespace

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
```

---

## Combined podSelector + namespaceSelector

```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        env: production
    podSelector:            # AND condition (same list item)
      matchLabels:
        role: prometheus
```

⚠️ **AND vs OR logic:**
```yaml
# OR: two separate list items
- from:
  - namespaceSelector: {...}   # from ANY pod in matching namespace
  - podSelector: {...}         # OR from matching pods in same namespace

# AND: one list item with both
- from:
  - namespaceSelector: {...}   # from matching pods in matching namespace
    podSelector: {...}
```

---

## Allow Egress to Specific Service

```yaml
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - ports:                   # allow DNS
    - protocol: UDP
      port: 53
```

> Always allow DNS (UDP 53) in egress policies or pods can't resolve service names.

---

## Allow Ingress from Specific IP Block

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8
      except:
      - 10.1.0.0/16
```

---

## Useful Commands

```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy deny-all -n production
kubectl get networkpolicies -n production

# Test connectivity
kubectl exec podA -- curl http://podB-service:8080
kubectl exec podA -- nc -zv podB-ip 8080
```
