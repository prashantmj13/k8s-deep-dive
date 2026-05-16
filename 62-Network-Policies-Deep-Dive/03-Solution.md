# 62 — Network Policies Deep Dive | Solutions

---

## Exercise 2 — Before policies

```
frontend → backend-svc:   200 OK
frontend → database-svc:  200 OK
testpod  → backend-svc:   200 OK
testpod  → database-svc:  200 OK
```

All open — no restrictions yet.

---

## Exercise 3 — After db-deny-all

```
testpod  → database-svc:  wget: download timed out
frontend → database-svc:  wget: download timed out
backend  → database-svc:  wget: download timed out
```

Database is now completely isolated.

---

## Exercise 4 — After db-allow-backend

```
backend  → database-svc:  200 OK          ← allowed by new policy
frontend → database-svc:  timed out       ← still blocked
testpod  → database-svc:  timed out       ← still blocked
```

Two policies now apply to database pods: `db-deny-all` and `db-allow-backend`. Rules are **ORed** — the allow wins for backend traffic.

---

## Exercise 5 — After backend-policy

```
frontend → backend-svc:   200 OK          ← allowed
testpod  → backend-svc:   timed out       ← blocked
```

Chain is now: frontend → backend → database. testpod is isolated from both.

---

## Challenge Answers

**1. Two policies — AND or OR?**
**OR.** When multiple NetworkPolicies select the same pod, their rules are combined with OR logic. Traffic is allowed if it satisfies ANY of the matching policies' rules. There is no way to create a "stricter than default" combination using multiple policies — you can only add more allowed traffic, not restrict it further through additional policies.

**2. Empty podSelector: {} matches?**
ALL pods in the namespace. It's a wildcard selector. Used in default-deny policies to apply to every pod without specifying labels.

**3. Why always include DNS egress rule?**
When you add an Egress NetworkPolicy, all outbound traffic is blocked by default unless explicitly allowed. This includes DNS (UDP/TCP port 53 to CoreDNS). Without it, the pod can't resolve `backend-svc` to an IP, causing all connections to fail with DNS errors even if the target is otherwise allowed.

**4. Can NetworkPolicy restrict same-node traffic?**
Yes, with a CNI plugin that supports it (Calico, Cilium). The CNI implements the rules at the kernel level using eBPF or iptables, which intercepts traffic even between pods on the same node before it reaches the destination.

**5. Allow all traffic (override deny-all)?**
```yaml
spec:
  podSelector:
    matchLabels:
      app: my-public-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}        # {} = allow from anywhere
  egress:
  - {}        # {} = allow to anywhere
```
