# 60 — Ingress Controllers | Solutions

---

## Exercise 3 — Ingress created

```bash
kubectl get ingress path-ingress
```
```
NAME           CLASS   HOSTS   ADDRESS        PORTS   AGE
path-ingress   nginx   *       192.168.49.2   80      10s
```

```bash
kubectl describe ingress path-ingress
```
```
Rules:
  Host        Path          Backends
  ----        ----          --------
  *           /frontend     frontend-svc:80
              /api          api-svc:80
```

---

## Exercise 4 — Path routing results

```bash
curl http://$INGRESS_IP/frontend
# Returns nginx HTML (Welcome to nginx!)

curl http://$INGRESS_IP/api
# Returns Apache HTML (It works!)
```

Different backends serve different paths through the single Ingress IP.

---

## Exercise 5 — Host routing

```bash
curl -H "Host: frontend.local" http://$INGRESS_IP
# Returns nginx (frontend-svc)

curl -H "Host: api.local" http://$INGRESS_IP
# Returns Apache (api-svc)

curl http://$INGRESS_IP
# 404 Not Found — no rule for the bare IP
```

---

## Exercise 6 — TLS

```bash
curl -k -H "Host: myapp.local" https://$INGRESS_IP
# Returns nginx HTML over HTTPS
# -k ignores self-signed cert warning
```

---

## Challenge Answers

**1. Ingress resource vs Ingress controller?**
- Ingress resource: a Kubernetes API object (YAML) that declares routing rules — which hostnames and paths go to which services. It is just configuration stored in etcd.
- Ingress controller: a running pod (usually NGINX or similar) that reads Ingress resources and actually implements the routing using a reverse proxy. Without a controller, Ingress resources have no effect.

**2. ingressClassName?**
Specifies which Ingress controller should handle this Ingress resource. Needed because a cluster can have multiple controllers (e.g. NGINX for internal, AWS ALB for external). Without `ingressClassName`, behaviour depends on the cluster's default IngressClass annotation.

**3. Prefix vs Exact pathType?**
- `Prefix`: matches the path and any sub-paths. `/api` matches `/api`, `/api/v1`, `/api/users/123`.
- `Exact`: only the exact path matches. `/api` matches only `/api` — not `/api/` or `/api/v1`.

**4. TLS termination in Ingress?**
The Ingress controller terminates the TLS connection using the cert/key from the referenced Secret. The controller decrypts the traffic and forwards plain HTTP to the backend service. The backend service doesn't need to handle TLS. The TLS Secret must be in the same namespace as the Ingress.

**5. Why Ingress over multiple LoadBalancers?**
Each LoadBalancer service provisions a separate cloud load balancer ($$$) with its own IP. Ingress uses a single LoadBalancer for the controller, then routes internally by host/path — dramatically cheaper and easier to manage. For 10 services: 10 LBs vs 1 LB + Ingress rules.
