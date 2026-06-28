# 67 — Ingress Advanced | Exercises

> **Cluster:** Cluster with nginx ingress controller installed
> **Estimated time:** 25 minutes

---

## Exercise 1 — Two backend services and path-based routing

```bash
kubectl create deployment api --image=hashicorp/http-echo -- -text="API response"
kubectl create deployment web --image=hashicorp/http-echo -- -text="Web response"
kubectl expose deployment api --port=5678
kubectl expose deployment web --port=5678
```

```yaml
# ingress-path.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api
            port:
              number: 5678
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-path.yaml
kubectl get ingress path-ingress
curl http://<ingress-ip>/api
curl http://<ingress-ip>/
```

---

## Exercise 2 — TLS Ingress with a self-signed cert

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=app.example.com"

kubectl create secret tls app-tls-secret --cert=tls.crt --key=tls.key
```

```yaml
# ingress-tls.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts: ["app.example.com"]
    secretName: app-tls-secret
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-tls.yaml
curl -k -H "Host: app.example.com" https://<ingress-ip>
```

---

## Exercise 3 — Canary deployment

```bash
kubectl create deployment web-canary --image=hashicorp/http-echo -- -text="CANARY response"
kubectl expose deployment web-canary --port=5678
```

```yaml
# ingress-canary.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "30"
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-canary
            port:
              number: 5678
```

```bash
kubectl apply -f ingress-canary.yaml
for i in $(seq 1 10); do curl -s -H "Host: app.example.com" http://<ingress-ip>; done
```

> **Observe:** roughly 3/10 requests return "CANARY response".

---

## Exercise 4 — Cleanup

```bash
kubectl delete ingress path-ingress tls-ingress web-canary-ingress
kubectl delete deployment api web web-canary
kubectl delete svc api web web-canary
kubectl delete secret app-tls-secret
```

---

## Challenge Questions

1. What is the purpose of `ingressClassName` when multiple controllers are installed?
2. What is the difference between `Prefix` and `Exact` path types?
3. How does a canary Ingress avoid needing changes to the main Ingress object?
4. What does `rewrite-target` do and why is it often needed?
5. What happens if a request matches no Ingress rule and no defaultBackend is set?

---

# Solutions

**Exercise 1:** `/api` returns "API response", `/` returns "Web response" — nginx routes based on longest-prefix match.

**Exercise 3:** Roughly 3 of 10 requests hit the canary service due to the 30% canary-weight annotation; this is probabilistic, not exact in small samples.

## Challenge Answers

**1. ingressClassName purpose?**
When multiple Ingress controllers run in the same cluster (e.g. nginx and traefik), this field tells Kubernetes which controller should process the Ingress. Without it, behavior depends on which controller claims to be "default" — ambiguous in multi-controller setups.

**2. Prefix vs Exact?**
`Exact` requires the request path to match character-for-character. `Prefix` matches if the request path starts with the rule's path, split on `/` boundaries (so `/api` matches `/api/v1` but not `/apiextra`).

**3. Canary without touching main Ingress?**
The canary Ingress is a SEPARATE object with the same host/path but the `canary: true` annotation. The nginx controller merges canary and main rules internally, splitting traffic by the configured weight — the original Ingress definition never needs modification, making canary rollout/rollback trivial (just delete the canary Ingress).

**4. rewrite-target purpose?**
Backend applications often don't expect the Ingress path prefix (e.g. app expects `/users`, not `/api/users`). `rewrite-target` strips/rewrites the matched prefix before forwarding, so the backend sees the path it expects.

**5. No matching rule, no defaultBackend?**
The Ingress controller's own built-in default backend handles it — typically returning a generic 404. If you want custom error pages, explicitly set `spec.defaultBackend`.
