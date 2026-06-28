# 68 — Gateway API | Exercises

> **Cluster:** Any cluster + a Gateway API implementation (e.g. nginx-gateway-fabric, Cilium, Istio)
> **Estimated time:** 25 minutes

---

## Exercise 1 — Install Gateway API CRDs

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml
kubectl get crds | grep gateway.networking.k8s.io
```

---

## Exercise 2 — Install a controller implementation (example: nginx-gateway-fabric)

```bash
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/main/deploy/crds.yaml
kubectl apply -f https://raw.githubusercontent.com/nginxinc/nginx-gateway-fabric/main/deploy/default/deploy.yaml
kubectl get pods -n nginx-gateway
```

---

## Exercise 3 — Create GatewayClass and Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
```

```bash
kubectl apply -f gatewayclass-gateway.yaml
kubectl get gateway my-gateway
```

---

## Exercise 4 — Deploy backend and HTTPRoute

```bash
kubectl create deployment web --image=hashicorp/http-echo -- -text="hello from gateway api"
kubectl expose deployment web --port=5678
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web
      port: 5678
```

```bash
kubectl apply -f web-route.yaml
kubectl get httproute web-route
GATEWAY_IP=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
curl http://$GATEWAY_IP/
```

---

## Exercise 5 — Native weighted traffic split

```bash
kubectl create deployment web-v2 --image=hashicorp/http-echo -- -text="hello v2"
kubectl expose deployment web-v2 --port=5678
```

```yaml
# patch web-route.yaml rules.backendRefs:
backendRefs:
- name: web
  port: 5678
  weight: 70
- name: web-v2
  port: 5678
  weight: 30
```

```bash
kubectl apply -f web-route.yaml
for i in $(seq 1 10); do curl -s http://$GATEWAY_IP/; done
```

---

## Exercise 6 — Cleanup

```bash
kubectl delete httproute web-route
kubectl delete gateway my-gateway
kubectl delete gatewayclass nginx
kubectl delete deployment web web-v2
kubectl delete svc web web-v2
```

---

## Challenge Questions

1. What is the role split between Gateway and HTTPRoute, and why does it matter?
2. How does Gateway API implement canary traffic splitting without annotations?
3. What CRD allows an HTTPRoute in one namespace to attach to a Gateway in another?
4. Name two protocols Gateway API supports that Ingress does not.
5. Is Gateway API a replacement that removes Ingress, or do they coexist?

---

# Solutions

**Exercise 4:** `curl` returns `hello from gateway api` — the HTTPRoute successfully attached to the Gateway and routed to the backend Service.

**Exercise 5:** Roughly 7/10 requests return "hello from gateway api" (v1) and 3/10 return "hello v2" — matching the 70/30 weight split, using only native fields.

## Challenge Answers

**1. Gateway vs HTTPRoute role split?**
`Gateway` is typically owned by platform/infra teams — it defines the actual load balancer, listener ports, and TLS config. `HTTPRoute` is owned by application teams — they define routing rules without needing access to or knowledge of the underlying infrastructure. This separation enables multi-tenant clusters where app teams self-serve routing without touching shared infra objects.

**2. Native canary without annotations?**
The `backendRefs` field on an HTTPRoute rule accepts a `weight` per backend. The Gateway controller implements weighted load balancing across the listed backends natively — no vendor-specific annotation syntax required, making it portable across implementations.

**3. Cross-namespace attachment CRD?**
`ReferenceGrant` — created in the target namespace (where the Gateway lives), explicitly allowing routes from a specified source namespace/kind to reference resources there. Without it, cross-namespace attachment is denied by default for security.

**4. Protocols Ingress doesn't support?**
TCP and UDP (via `TCPRoute`/`UDPRoute`), and gRPC has first-class support via `GRPCRoute` with method/service matching. Ingress is HTTP/HTTPS-only by design.

**5. Replacement or coexistence?**
They coexist. Ingress remains supported and stable but is feature-frozen — no major new capabilities are being added. Gateway API is the actively developed path forward and is recommended for new projects, but migration is optional and many clusters run both side by side.
