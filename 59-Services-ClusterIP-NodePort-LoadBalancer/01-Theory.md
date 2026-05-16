# 59 — Services: ClusterIP, NodePort, LoadBalancer

## Why Services Exist

Pods are ephemeral — they come and go, their IPs change constantly. A **Service** provides a **stable IP and DNS name** that load-balances traffic to a set of pods, regardless of pod churn.

```
Without Service:          With Service:
Pod A (10.0.0.1) ←       Client → my-svc:80 → Pod A (10.0.0.1)
Pod A dies                                    → Pod B (10.0.0.2)
Pod B (10.0.0.2) ←       Stable endpoint regardless of pod IPs
```

---

## How Services Select Pods

Services use **label selectors** to find their backend pods:

```yaml
# Service selects pods with app=nginx
spec:
  selector:
    app: nginx    # matches pods with this label

# Pod must have matching label
metadata:
  labels:
    app: nginx
```

---

## Service Types

| Type | Accessible From | Use Case |
|------|----------------|----------|
| `ClusterIP` | Inside cluster only | Inter-service communication |
| `NodePort` | Outside cluster via node IP | Dev/test, bare-metal |
| `LoadBalancer` | Outside cluster via cloud LB | Production on cloud |
| `ExternalName` | Inside cluster | Map to external DNS name |
| `Headless` | DNS only, no VIP | StatefulSets, direct pod access |

---

## 1. ClusterIP (default)

Assigns a **virtual IP** reachable only from within the cluster. This is the most common service type.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  type: ClusterIP        # default, can be omitted
  selector:
    app: backend
  ports:
  - port: 80             # port the Service listens on
    targetPort: 8080     # port the pod is listening on
    protocol: TCP
```

```bash
# Pods reach it via:
curl http://backend-svc        # DNS within same namespace
curl http://backend-svc.default.svc.cluster.local  # full DNS
curl http://10.96.100.50       # ClusterIP
```

---

## 2. NodePort

Exposes the service on a **static port on every node** in the cluster (range 30000–32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80             # ClusterIP port (internal)
    targetPort: 8080     # pod port
    nodePort: 30080      # external port on every node (30000-32767)
    protocol: TCP
```

```
External client → <NodeIP>:30080 → ClusterIP:80 → Pod:8080
```

```bash
curl http://192.168.1.10:30080   # access via any node IP
```

---

## 3. LoadBalancer

Provisions a **cloud load balancer** automatically. Builds on NodePort internally.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-svc
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
  - port: 80
    targetPort: 8080
```

```bash
kubectl get service app-svc
# EXTERNAL-IP column shows the cloud LB IP once provisioned
```

```
Internet → cloud-lb:80 → NodePort → ClusterIP → Pod:8080
```

Cloud providers (GKE, EKS, AKS) automatically create the load balancer. On bare-metal clusters you need **MetalLB** or similar.

---

## Port Terminology

```
                  Service          Pod
                  ┌──────┐        ┌──────┐
Client ──→ port:80│      │─────→ │:8080 │
           nodePort:30080│        └──────┘
                  └──────┘
                  targetPort: 8080

port       = Service's listening port (what clients connect to)
targetPort = Pod's listening port (where traffic is forwarded)
nodePort   = Node's listening port (NodePort/LoadBalancer only)
```

---

## Named targetPort

Reference container port by name instead of number — more resilient:

```yaml
# In pod spec:
ports:
- name: http
  containerPort: 8080

# In service spec:
ports:
- port: 80
  targetPort: http    # name reference
```

---

## Headless Service

Set `clusterIP: None` — no VIP is assigned. DNS returns individual pod IPs instead. Used by StatefulSets:

```yaml
spec:
  clusterIP: None
  selector:
    app: mysql
```

```bash
# DNS returns all pod IPs, not a single VIP
nslookup mysql-svc
# mysql-0.mysql-svc.default.svc.cluster.local → 10.0.0.1
# mysql-1.mysql-svc.default.svc.cluster.local → 10.0.0.2
```

---

## ExternalName Service

Maps a service to an external DNS name:

```yaml
spec:
  type: ExternalName
  externalName: database.prod.example.com
```

Pods use `my-db-svc` and Kubernetes resolves it to the external name. Good for migrating workloads.

---

## Useful Commands

```bash
# List services
kubectl get services
kubectl get svc

# Describe a service (see endpoints)
kubectl describe svc backend-svc

# See which pods are behind a service
kubectl get endpoints backend-svc

# Create a service imperatively
kubectl expose deployment nginx --port=80 --target-port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Test from inside cluster
kubectl run tmp --image=busybox --restart=Never --rm -it -- wget -qO- http://backend-svc
```

---

## kube-proxy

`kube-proxy` runs on every node and implements Service networking using iptables or IPVS rules. It watches the API server for Service and Endpoint changes and updates routing rules accordingly.

```bash
kubectl get pods -n kube-system | grep kube-proxy
```
