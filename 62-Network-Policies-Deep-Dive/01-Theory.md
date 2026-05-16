# 62 — Network Policies (Deep Dive)

## Recap and Why This Matters

By default every pod can reach every other pod in the cluster — even across namespaces. NetworkPolicies are Kubernetes-native firewall rules that restrict this. This topic goes deeper than the introduction in topic 49.

---

## Default Behaviour Without Policies

```
Pod A (ns: frontend) ──→ Pod B (ns: backend)    ✅ allowed
Pod A (ns: frontend) ──→ Pod C (ns: database)   ✅ allowed
Pod X (ns: hacker)   ──→ Pod C (ns: database)   ✅ allowed  ← dangerous!
```

With NetworkPolicies you can enforce proper isolation.

---

## NetworkPolicy Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend              # policy lives in backend namespace
spec:
  podSelector:                    # which pods this policy applies to
    matchLabels:
      app: backend
  policyTypes:
  - Ingress                       # control inbound traffic
  - Egress                        # control outbound traffic
  ingress:
  - from:                         # allow from these sources
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: frontend
      podSelector:                # AND condition (same list item)
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  - ports:                        # always allow DNS
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

---

## AND vs OR in Selectors (Critical!)

```yaml
# OR: two separate list items — matches either condition
ingress:
- from:
  - namespaceSelector:            # allows ANY pod from ns with label env=prod
      matchLabels:
        env: prod
  - podSelector:                  # OR any pod with role=admin in SAME namespace
      matchLabels:
        role: admin

# AND: one list item with both — must match BOTH
ingress:
- from:
  - namespaceSelector:            # allows pods with role=admin
      matchLabels:                # from namespace with env=prod only
        env: prod
    podSelector:
      matchLabels:
        role: admin
```

---

## Default Deny All Patterns

```yaml
# Deny all ingress to all pods in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}         # {} selects ALL pods
  policyTypes:
  - Ingress               # no ingress rules = deny all ingress
---
# Deny all egress from all pods in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress                # no egress rules = deny all egress
---
# Deny BOTH ingress and egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

---

## Allow All Patterns

```yaml
# Allow all ingress (override deny-all for specific pods)
spec:
  podSelector:
    matchLabels:
      app: public-api
  policyTypes:
  - Ingress
  ingress:
  - {}                    # {} means allow from anywhere
```

---

## Real-World Pattern: Three-Tier App

```
Internet → Ingress → Frontend → Backend → Database
```

```yaml
# 1. Frontend: allow ingress from ingress controller only
kind: NetworkPolicy
metadata: { name: frontend-policy, namespace: app }
spec:
  podSelector: { matchLabels: { tier: frontend } }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
  egress:
  - to:
    - podSelector: { matchLabels: { tier: backend } }
    ports: [{ port: 8080 }]
  - ports: [{ port: 53, protocol: UDP }]   # DNS

# 2. Backend: allow ingress from frontend only
kind: NetworkPolicy
metadata: { name: backend-policy, namespace: app }
spec:
  podSelector: { matchLabels: { tier: backend } }
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - podSelector: { matchLabels: { tier: frontend } }
    ports: [{ port: 8080 }]
  egress:
  - to:
    - podSelector: { matchLabels: { tier: database } }
    ports: [{ port: 5432 }]
  - ports: [{ port: 53, protocol: UDP }]

# 3. Database: allow ingress from backend only
kind: NetworkPolicy
metadata: { name: database-policy, namespace: app }
spec:
  podSelector: { matchLabels: { tier: database } }
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: { matchLabels: { tier: backend } }
    ports: [{ port: 5432 }]
```

---

## IPBlock — Restrict by CIDR

```yaml
ingress:
- from:
  - ipBlock:
      cidr: 10.0.0.0/8      # allow from internal network
      except:
      - 10.1.0.0/16         # but not from this subnet
```

---

## Checking if NetworkPolicy is Working

```bash
# Test connectivity between pods
kubectl exec frontend-pod -- curl -m2 http://backend-svc:8080    # should work
kubectl exec testpod    -- curl -m2 http://backend-svc:8080    # should fail

# List all policies
kubectl get networkpolicies -A
kubectl describe networkpolicy allow-frontend-to-backend -n backend

# Note: NetworkPolicy requires a CNI that supports it (Calico, Cilium, Weave)
# Verify your CNI:
kubectl get pods -n kube-system | grep -E "calico|cilium|weave|flannel"
```
