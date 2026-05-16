# 60 — Ingress Controllers | Exercises

> **Cluster:** minikube (easiest) or any cluster with NGINX Ingress installed
> **Estimated time:** 30 minutes
> **Namespace:** `ingress-lab`

---

## Exercise 1 — Setup: Enable Ingress on minikube

```bash
# minikube
minikube addons enable ingress
kubectl get pods -n ingress-nginx   # wait until Running

# OR install NGINX Ingress on any cluster
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

---

## Exercise 2 — Deploy two applications

```bash
kubectl create namespace ingress-lab
kubectl config set-context --current --namespace=ingress-lab

# App 1: frontend
kubectl create deployment frontend --image=nginx
kubectl expose deployment frontend --port=80 --name=frontend-svc

# App 2: api (using httpd as a stand-in)
kubectl create deployment api --image=httpd
kubectl expose deployment api --port=80 --name=api-svc

kubectl get pods
kubectl get svc
```

---

## Exercise 3 — Path-based Ingress

```yaml
# path-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
  namespace: ingress-lab
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /frontend
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

```bash
kubectl apply -f path-ingress.yaml
kubectl get ingress path-ingress
kubectl describe ingress path-ingress
```

---

## Exercise 4 — Test path routing

```bash
# Get ingress IP
INGRESS_IP=$(kubectl get ingress path-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_IP

# On minikube
INGRESS_IP=$(minikube ip)

# Test both paths
curl http://$INGRESS_IP/frontend
curl http://$INGRESS_IP/api
```

---

## Exercise 5 — Host-based Ingress

```yaml
# host-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  rules:
  - host: frontend.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
  - host: api.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

```bash
kubectl apply -f host-ingress.yaml

# Test with Host header (simulates DNS)
curl -H "Host: frontend.local" http://$INGRESS_IP
curl -H "Host: api.local" http://$INGRESS_IP
```

---

## Exercise 6 — TLS with self-signed cert

```bash
# Generate self-signed cert
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key -out /tmp/tls.crt \
  -subj "/CN=myapp.local"

# Create TLS secret
kubectl create secret tls myapp-tls \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key \
  -n ingress-lab
```

```yaml
# tls-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  namespace: ingress-lab
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls
  rules:
  - host: myapp.local
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
kubectl apply -f tls-ingress.yaml
curl -k -H "Host: myapp.local" https://$INGRESS_IP
```

---

## Exercise 7 — Cleanup

```bash
kubectl delete namespace ingress-lab
kubectl config set-context --current --namespace=default
```

---

## Challenge Questions

1. What is the difference between an Ingress resource and an Ingress controller?
2. What is `ingressClassName` and why is it needed?
3. What is the difference between `Prefix` and `Exact` pathType?
4. How does TLS termination work in Ingress?
5. Why is Ingress preferred over multiple LoadBalancer services?
