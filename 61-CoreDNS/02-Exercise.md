# 61 — CoreDNS | Exercises

> **Cluster:** Any kubectl-accessible cluster
> **Estimated time:** 20 minutes

---

## Exercise 1 — Inspect CoreDNS setup

```bash
# Find CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Find CoreDNS service IP
kubectl get svc -n kube-system kube-dns

# View the Corefile
kubectl get configmap coredns -n kube-system -o yaml
```

---

## Exercise 2 — Check pod DNS config

```bash
kubectl run dns-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec dns-pod -- cat /etc/resolv.conf
```

Note the `nameserver` and `search` lines.

---

## Exercise 3 — Test service DNS resolution

```bash
# Create a test service
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --name=nginx-svc

# Resolve it using different DNS name forms
kubectl exec dns-pod -- nslookup nginx-svc
kubectl exec dns-pod -- nslookup nginx-svc.default
kubectl exec dns-pod -- nslookup nginx-svc.default.svc.cluster.local
```

---

## Exercise 4 — Cross-namespace DNS resolution

```bash
# Create a service in another namespace
kubectl create namespace dns-test
kubectl create deployment httpd --image=httpd -n dns-test
kubectl expose deployment httpd --port=80 --name=httpd-svc -n dns-test

# Try to reach it from the dns-pod (in default namespace)
kubectl exec dns-pod -- nslookup httpd-svc              # fails (wrong ns)
kubectl exec dns-pod -- nslookup httpd-svc.dns-test     # works
kubectl exec dns-pod -- nslookup httpd-svc.dns-test.svc.cluster.local  # works
```

---

## Exercise 5 — Resolve external DNS

```bash
kubectl exec dns-pod -- nslookup google.com
kubectl exec dns-pod -- nslookup github.com
```

> CoreDNS forwards external queries to upstream resolvers.

---

## Exercise 6 — Check CoreDNS logs

```bash
kubectl logs -n kube-system -l k8s-app=kube-dns --tail=20
```

---

## Exercise 7 — Pod with custom DNS config

```yaml
# pod-custom-dns.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-custom-dns
spec:
  dnsPolicy: None
  dnsConfig:
    nameservers:
    - 8.8.8.8
    - 8.8.4.4
    searches:
    - mycompany.internal
    options:
    - name: ndots
      value: "2"
  containers:
  - name: app
    image: busybox
    command: ["sleep", "3600"]
```

```bash
kubectl apply -f pod-custom-dns.yaml
kubectl exec pod-custom-dns -- cat /etc/resolv.conf
```

---

## Exercise 8 — Cleanup

```bash
kubectl delete pod dns-pod pod-custom-dns
kubectl delete namespace dns-test
kubectl delete deployment nginx
kubectl delete svc nginx-svc
```

---

## Challenge Questions

1. What is the full DNS name for a service named `redis` in the `cache` namespace?
2. Why does a short name like `nginx-svc` work from within the same namespace?
3. What does the `search` field in `/etc/resolv.conf` do?
4. What is `ndots` and why does setting it too high cause slow DNS?
5. How do you make CoreDNS forward queries for `mycompany.internal` to an internal DNS server?
