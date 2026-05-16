# 60 — Ingress Controllers

## Why Ingress?

A LoadBalancer service creates one cloud LB per service — expensive and hard to manage at scale. **Ingress** provides a single entry point that routes HTTP/HTTPS traffic to multiple services based on host name or URL path.

```
Without Ingress:
  app-svc   → LB1 ($$$)
  api-svc   → LB2 ($$$)
  admin-svc → LB3 ($$$)

With Ingress:
  Single LB → Ingress Controller → /app   → app-svc
                                 → /api   → api-svc
                                 → /admin → admin-svc
```

---

## Two Parts: Ingress Resource + Ingress Controller

| Part | What it is |
|------|-----------|
| **Ingress resource** | A Kubernetes object declaring routing rules (YAML) |
| **Ingress controller** | A running pod that reads the rules and implements them |

Kubernetes does NOT include an Ingress controller by default — you must install one.

---

## Popular Ingress Controllers

| Controller | Notes |
|-----------|-------|
| **NGINX Ingress** | Most popular, maintained by Kubernetes community |
| **Traefik** | Cloud-native, auto-discovers routes |
| **HAProxy** | High performance |
| **AWS ALB Ingress** | Native AWS Application Load Balancer |
| **GKE Ingress (GCLB)** | Native GCP Global Load Balancer |
| **Istio Gateway** | Part of Istio service mesh |

---

## Installing NGINX Ingress Controller

```bash
# Using helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Using kubectl (bare-metal)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

---

## Basic Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx          # which controller to use
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

---

## Path-based Routing

Route different paths to different services:

```yaml
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

---

## Host-based Routing

Route different hostnames to different services:

```yaml
spec:
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
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 8080
```

---

## TLS Termination

```yaml
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-secret    # TLS cert stored as a Secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

```bash
# Create TLS secret from cert files
kubectl create secret tls myapp-tls-secret \
  --cert=tls.crt \
  --key=tls.key
```

---

## pathType Options

| pathType | Behaviour |
|----------|-----------|
| `Exact` | Exact match only: `/foo` matches only `/foo` |
| `Prefix` | Prefix match: `/foo` matches `/foo`, `/foo/bar`, `/foobar` |
| `ImplementationSpecific` | Controller-defined behaviour |

---

## Default Backend

Handles requests that don't match any rule:

```yaml
spec:
  defaultBackend:
    service:
      name: default-svc
      port:
        number: 80
  rules:
  - ...
```

---

## Useful Annotations (NGINX)

```yaml
annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /           # strip path prefix
  nginx.ingress.kubernetes.io/ssl-redirect: "true"        # force HTTPS
  nginx.ingress.kubernetes.io/proxy-body-size: "10m"      # upload size limit
  nginx.ingress.kubernetes.io/rate-limit: "100"           # rate limiting
  nginx.ingress.kubernetes.io/auth-url: "http://auth-svc" # external auth
```

---

## Useful Commands

```bash
kubectl get ingress
kubectl get ing              # shorthand
kubectl describe ingress my-ingress
kubectl get ingressclass     # list available ingress controllers

# Check ingress controller logs
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller
```
