# 68 — Gateway API

## What is Gateway API?

Gateway API is the **next-generation successor to Ingress** — a more expressive, role-oriented, extensible standard for managing ingress traffic. It's a separate set of CRDs (`gateway.networking.k8s.io`), not built into core Kubernetes.

---

## Why Gateway API over Ingress?

| Limitation of Ingress | Gateway API Solution |
|------------------------|----------------------|
| Vendor-specific annotations for everything (canary, rewrite, auth) | Native typed fields for common features |
| One resource mixes infra + routing concerns | Split into Gateway (infra) + Routes (app) |
| No native support for TCP/UDP/gRPC | GRPCRoute, TCPRoute, UDPRoute, TLSRoute |
| Limited cross-namespace control | ReferenceGrant for explicit cross-ns permissions |

---

## Core Resource Types

| Resource | Owned by | Purpose |
|----------|----------|---------|
| `GatewayClass` | Platform admin | Defines a type of load balancer/controller |
| `Gateway` | Cluster/platform admin | A deployed instance of a load balancer — listeners, ports, TLS |
| `HTTPRoute` | App developer | HTTP routing rules attached to a Gateway |
| `GRPCRoute` | App developer | gRPC-specific routing |
| `TCPRoute`/`UDPRoute` | App developer | L4 routing |
| `ReferenceGrant` | Namespace owner | Allow cross-namespace route attachment |

---

## GatewayClass (defines the controller)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: k8s.io/nginx-gateway-controller
```

---

## Gateway (the load balancer instance)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: infra
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: app-tls-secret
    allowedRoutes:
      namespaces:
        from: Same
```

---

## HTTPRoute (the routing rules)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
    namespace: infra
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-svc
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-svc
      port: 80
```

---

## Native Traffic Splitting (Canary, no annotations!)

```yaml
rules:
- backendRefs:
  - name: web-svc-v1
    port: 80
    weight: 80
  - name: web-svc-v2
    port: 80
    weight: 20
```

No vendor-specific annotations needed — weighted backends are a native field.

---

## Header-based Routing

```yaml
rules:
- matches:
  - headers:
    - name: X-Canary
      value: "true"
  backendRefs:
  - name: web-svc-v2
    port: 80
```

---

## Cross-Namespace Routing with ReferenceGrant

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-default-to-infra
  namespace: infra
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: default
  to:
  - group: ""
    kind: Service
```

---

## Ingress vs Gateway API Summary

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| Traffic splitting | Annotations (vendor-specific) | Native `weight` field |
| Header matching | Annotations | Native `matches.headers` |
| Protocol support | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC, TLS |
| Role separation | Single object, mixed concerns | Gateway (infra) vs Route (app) split |
| Cross-namespace | Implicit, hard to control | Explicit via ReferenceGrant |
| Status | GA, but feature-frozen | GA (v1), actively evolving |

---

## Checking Gateway API Support

```bash
kubectl get crds | grep gateway.networking.k8s.io
kubectl get gatewayclass
kubectl get gateway -A
kubectl get httproute -A
kubectl describe gateway my-gateway -n infra
```
