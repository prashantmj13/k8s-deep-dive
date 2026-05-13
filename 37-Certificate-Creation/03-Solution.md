# 37 — Certificate Creation | Solutions

---

## Exercise 1 — CA

```
Subject: CN = MY-K8S-CA
Issuer: CN = MY-K8S-CA      ← same = self-signed
Not Before: May 13 00:00:00 2026 GMT
Not After : May 11 00:00:00 2036 GMT
```

---

## Exercise 2 — Admin cert

```
Subject: CN = kube-admin, O = system:masters
Issuer: CN = MY-K8S-CA
```

Kubernetes will see `kube-admin` as the username and `system:masters` as the group — which maps to cluster-admin rights.

---

## Exercise 4 — API server SANs

```
X509v3 Subject Alternative Name:
    DNS:kubernetes, DNS:kubernetes.default,
    DNS:kubernetes.default.svc,
    DNS:kubernetes.default.svc.cluster.local,
    IP Address:10.96.0.1, IP Address:127.0.0.1
```

---

## Exercise 5 — Verification

```
admin.crt: OK
dev1.crt: OK
apiserver.crt: OK
```

If output shows `OK`, the cert was correctly signed by your CA.

---

## Challenge Answers

**1. CN and O fields?**
- `CN` (Common Name) → Kubernetes **username**
- `O` (Organization) → Kubernetes **group**
A user can have multiple O fields for multiple group memberships.

**2. Why SANs for API server?**
The API server is accessed via multiple hostnames (kubernetes, kubernetes.default, the node hostname) and multiple IPs (cluster IP 10.96.0.1, node IP). TLS verification checks that the server's cert covers the name the client used to connect. Without all SANs, clients get a certificate mismatch error.

**3. CAcreateserial?**
Creates a serial number file (`ca.srl`) that tracks the serial numbers of all certificates issued by this CA. Each certificate gets a unique serial. Required when signing multiple certs with the same CA.

**4. Cluster-admin group?**
`system:masters`. This group is bound to the `cluster-admin` ClusterRole by default in kubeadm clusters. Any user with `O=system:masters` in their cert gets full cluster access.

**5. Verify cert against CA?**
```bash
openssl verify -CAfile ca.crt component.crt
# Output: component.crt: OK
```
Returns `OK` if the cert chain is valid. Returns an error if the cert was signed by a different CA or has been tampered with.
