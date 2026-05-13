# 49 — Network Policy | Solutions

---

## Exercise 2 — Open access

```bash
kubectl exec testpod -- wget -qO- http://frontend-svc
# returns nginx HTML

kubectl exec testpod -- wget -qO- http://backend-svc
# returns nginx HTML
```

All pods can communicate — no policies in place yet.

---

## Exercise 3 — After deny-all-backend

```bash
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← blocked by NetworkPolicy

kubectl exec testpod -- wget -qO- http://frontend-svc
# returns nginx HTML   ← unaffected
```

---

## Exercise 4 — Frontend reaches backend

```bash
kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← testpod still blocked

kubectl exec frontend -- wget -qO- http://backend-svc
# returns nginx HTML   ← frontend allowed by new policy
```

---

## Exercise 5 — Frontend egress blocked

```bash
kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← egress to port 80 now blocked
```

DNS still works (UDP 53 allowed), but HTTP traffic from frontend is blocked.

---

## Challenge Answers

**1. Pod with no NetworkPolicy?**
All traffic is allowed — both ingress and egress. NetworkPolicy is opt-in. Only when a pod is **selected by at least one NetworkPolicy** does traffic become restricted. Unselected pods are in the "open" state.

**2. OR vs AND in from rules?**
- Two separate list items = OR: traffic is allowed if it matches either selector.
- One list item with both selectors = AND: traffic must match BOTH (the pod must be in a matching namespace AND have matching labels).
```yaml
# OR
- from:
  - namespaceSelector: {matchLabels: {ns: prod}}
  - podSelector: {matchLabels: {role: admin}}

# AND
- from:
  - namespaceSelector: {matchLabels: {ns: prod}}
    podSelector: {matchLabels: {role: admin}}
```

**3. Always allow UDP 53?**
DNS resolution uses UDP port 53 to resolve service names to IP addresses. If you block all egress and don't allow DNS, pods can't resolve `backend-svc` to an IP. They get `Name or service not known` errors even before attempting the connection.

**4. Block traffic from the same pod?**
No. NetworkPolicy does not filter traffic within the same pod (loopback). It only applies to traffic between pods. A container can always reach other containers in the same pod via localhost.

**5. Without compatible CNI?**
NetworkPolicy objects can be created and stored in etcd, but they have **no effect**. The CNI plugin is responsible for actually enforcing the rules at the kernel level. With Flannel (no NetworkPolicy support), traffic flows freely regardless of what policies you define. Always verify your CNI supports NetworkPolicy before relying on it for security.
