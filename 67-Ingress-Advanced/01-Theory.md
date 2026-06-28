# 67 — Ingress Advanced Patterns

> Builds on topic 60 (Ingress Controllers basics). This covers advanced routing, TLS, and multi-controller patterns.

## IngressClass — Multiple Controllers in One Cluster

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s.io/ingress-nginx
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: traefik
spec:
  controller: traefik.io/ingress-controller
```

Reference it in an Ingress:
```yaml
spec:
  ingressClassName: nginx
```

---

## Path Types

| pathType | Matching |
|----------|----------|
| `Exact` | Must match the URL path exactly |
| `Prefix` | Matches based on a URL path prefix split by `/` |
| `ImplementationSpecific` | Controller decides (legacy nginx regex behaviour) |

```yaml
paths:
- path: /api
  pathType: Prefix       # matches /api, /api/v1, /api/users etc.
- path: /health
  pathType: Exact         # matches only /health exactly
```

---

## TLS Termination

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls-secret    # cert-manager populates this automatically
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-svc
            port:
              number: 80
```

---

## Canary Deployments (nginx annotations)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"   # 20% of traffic to canary
```

Apply this on a SECOND Ingress pointing to the canary service — the main Ingress stays unchanged and gets 80% of traffic.

---

## Rewrite and Redirect

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

Request to `/api/users` is rewritten to `/users` before reaching the backend.

---

## Default Backend

```yaml
spec:
  defaultBackend:
    service:
      name: default-404-svc
      port:
        number: 80
```

Catches any request that doesn't match a rule — usually a custom 404 page.

---

## Rate Limiting and Auth (nginx annotations)

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
```

---

## Debugging Ingress

```bash
kubectl describe ingress my-ingress
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50

# Check generated nginx config (nginx controller specific)
kubectl exec -n ingress-nginx deploy/ingress-nginx-controller -- cat /etc/nginx/nginx.conf | grep -A20 "server_name app.example.com"
```
