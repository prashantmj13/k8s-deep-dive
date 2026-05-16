# 62 — Network Policies | Solutions

---

## Exercise 3 — Before any policy

```bash
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc
# returns nginx HTML — all open

kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# returns nginx HTML — all open
```

---

## Exercise 4 — After deny-all

```bash
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc
# wget: download timed out   ← blocked

kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← blocked
```

Even testpod cannot reach anything — egress from testpod is also blocked.

---

## Exercise 5 — After allowing frontend ingress

```bash
kubectl exec testpod -- wget -qO- --timeout=3 http://frontend-svc
# returns nginx HTML   ← allowed

kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← still blocked
```

---

## Exercise 6 — Frontend to backend

```bash
kubectl exec frontend -- wget -qO- --timeout=3 http://backend-svc
# returns nginx HTML   ← allowed (frontend pod → backend pod)

kubectl exec testpod -- wget -qO- --timeout=3 http://backend-svc
# wget: download timed out   ← testpod still blocked
```

---

## Exercise 8 — Database protection

```bash
kubectl exec backend -- wget -qO- --timeout=3 http://database-svc
# returns nginx HTML   ← backend can reach database

kubectl exec frontend -- wget -qO- --timeout=3 http://database-svc
# wget: download timed out   ← frontend cannot bypass backend layer
```

Final state of policies:
```
kubectl get networkpolicies
NAME                         POD-SELECTOR           AGE
allow-dns-egress             <none>                 2m
allow-frontend-ingress       tier=frontend          3m
allow-frontend-to-backend    tier=backend           2m
deny-all                     <none>                 5m
protect-database             tier=database          1m
```

---

## Challenge Answers

**1. Pod matching NO NetworkPolicy?**
All traffic is allowed — ingress and egress. NetworkPolicy is additive and opt-in. Only pods selected by at least one policy have their traffic restricted. Unselected pods run with the default open network.

**2. Why DNS egress rule?**
When egress is blocked, pods can't reach UDP port 53 (kube-dns). Service name resolution (e.g. `http://backend-svc`) fails with `Name or service not known` before even attempting the TCP connection. Even if you allowed the destination pod, without DNS the pod can't look up its address. Always add UDP/TCP port 53 egress to any namespace that blocks egress.

**3. podSelector vs namespaceSelector in from?**
- `podSelector`: matches pods by label **within the same namespace** as the NetworkPolicy (unless combined with namespaceSelector).
- `namespaceSelector`: matches pods in **any namespace** whose namespace has the matching label. Allows cross-namespace traffic control.
Combined (AND logic, same list item): only pods with matching labels in a matching namespace.

**4. Block traffic between containers in same pod?**
No. NetworkPolicy operates at the pod boundary, not the container boundary. Containers in the same pod share a network namespace and communicate via `localhost` — this traffic is never inspected by NetworkPolicy.

**5. AND logic — specific pod in specific namespace?**
Use a single `from` list item with BOTH selectors on the same object:
```yaml
ingress:
- from:
  - namespaceSelector:
      matchLabels:
        env: production    # namespace must have this label
    podSelector:           # AND pod must have this label
      matchLabels:
        role: prometheus
```
Two separate list items would be OR logic (less restrictive).
