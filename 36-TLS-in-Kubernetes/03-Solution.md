# 36 — TLS in Kubernetes | Solutions

---

## Exercise 2 — API server certificate

```
Subject Alternative Name:
    DNS:controlplane, DNS:kubernetes, DNS:kubernetes.default,
    DNS:kubernetes.default.svc,
    DNS:kubernetes.default.svc.cluster.local,
    IP Address:10.96.0.1, IP Address:192.168.1.10

Not Before: Jan  1 00:00:00 2026 GMT
Not After : Jan  1 00:00:00 2027 GMT
Subject: CN = kube-apiserver
Issuer: CN = kubernetes
```

---

## Exercise 3 — kubeadm cert check

```
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY
admin.conf                 Jan 01, 2027 00:00 UTC   364d            ca
apiserver                  Jan 01, 2027 00:00 UTC   364d            ca
apiserver-etcd-client      Jan 01, 2027 00:00 UTC   364d            etcd-ca
apiserver-kubelet-client   Jan 01, 2027 00:00 UTC   364d            ca
controller-manager.conf    Jan 01, 2027 00:00 UTC   364d            ca
etcd-healthcheck-client    Jan 01, 2027 00:00 UTC   364d            etcd-ca
etcd-peer                  Jan 01, 2027 00:00 UTC   364d            etcd-ca
etcd-server                Jan 01, 2027 00:00 UTC   364d            etcd-ca
front-proxy-client         Jan 01, 2027 00:00 UTC   364d            front-proxy-ca
scheduler.conf             Jan 01, 2027 00:00 UTC   364d            ca
```

---

## Challenge Answers

**1. What if API server cert expires?**
The API server will reject all connections — kubectl and all components lose access to the cluster. You must renew the cert and restart the API server. This is a cluster outage. Monitor expiry dates with `kubeadm certs check-expiration`.

**2. SANs and why multiple?**
A SAN (Subject Alternative Name) lists all valid names/IPs for a server cert. The API server is reached by many different DNS names and IPs (by pods using `kubernetes.default`, by admin using the node IP, by services using the cluster IP). Without all SANs, TLS verification fails.

**3. Why separate etcd CA?**
Defense in depth. If the Kubernetes CA is compromised, an attacker can't immediately forge etcd client certificates (different CA). Isolation also limits blast radius — a compromised component can only trust things signed by its own CA.

**4. Controller-manager and scheduler credentials?**
Stored in kubeconfig files:
- `/etc/kubernetes/controller-manager.conf`
- `/etc/kubernetes/scheduler.conf`

These contain embedded client certificates signed by the cluster CA.

**5. Renew all certificates?**
```bash
kubeadm certs renew all
# Then restart control plane components:
systemctl restart kubelet
```
On kubeadm clusters, control plane components will pick up the new certs after restart.
