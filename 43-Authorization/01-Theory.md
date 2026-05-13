# 43 — Authorization

## What is Authorization?

Authorization happens **after** authentication. It answers: "Now that we know who you are, are you **allowed** to do what you're asking?"

```
Request → Authentication (who are you?) → Authorization (are you allowed?) → Admission Control → etcd
```

---

## Authorization Modes

Configured on the API server with `--authorization-mode`. Multiple modes are evaluated **in order** — access is granted if **any** mode allows it.

```bash
# Check current mode
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep authorization-mode
# Typical: --authorization-mode=Node,RBAC
```

| Mode | Description |
|------|-------------|
| `Node` | Special mode for kubelet access — kubelets can only manage their own node's pods |
| `RBAC` | Role-Based Access Control — recommended |
| `ABAC` | Attribute-Based — policy files, legacy |
| `Webhook` | External authorization service |
| `AlwaysAllow` | Allow everything — dev only, never production |
| `AlwaysDeny` | Deny everything |

---

## Node Authorization

The `Node` authorizer is specifically designed for kubelets. It allows a kubelet to:
- Read secrets, configmaps, and PVCs for pods **scheduled on its node only**
- Read node and pod status
- Write node status and pod status for **its own node**

Kubelets must have a credential with `system:node:<nodename>` as CN and `system:nodes` group. Without the `Node` authorizer, kubelets couldn't function.

---

## RBAC Authorization

RBAC grants permissions through Roles and RoleBindings (covered in detail in topic 44). The key flow:

```
Request comes in
    ↓
API server checks: does a RoleBinding/ClusterRoleBinding exist
that grants this user/group/SA the requested verb on this resource?
    ↓
Yes → Allow   |   No → Deny
```

RBAC is **deny by default** — no permissions unless explicitly granted.

---

## ABAC Authorization (legacy)

Attribute-Based Access Control uses a policy file:

```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy",
 "spec": {"user": "jane", "namespace": "*", "resource": "pods", "readonly": true}}
```

Requires API server restart to update policies. Much harder to manage than RBAC. Not recommended for new clusters.

---

## Webhook Authorization

Delegates authorization to an external HTTP service:

```
API server → sends SubjectAccessReview → external webhook service
                                              ↓
                                   returns allowed: true/false
```

```yaml
# API server flag
--authorization-webhook-config-file=/etc/kubernetes/webhook-authz.yaml
```

Used for integrating with external policy engines like Open Policy Agent (OPA).

---

## Checking Authorization with SubjectAccessReview

```bash
# Can jane create pods?
kubectl auth can-i create pods --as jane

# Check via SubjectAccessReview API directly
kubectl create -f - <<EOF
apiVersion: authorization.k8s.io/v1
kind: SubjectAccessReview
spec:
  user: jane
  resourceAttributes:
    verb: create
    resource: pods
    namespace: default
EOF
```

---

## Authorization vs Admission Control

| Phase | Purpose | Examples |
|-------|---------|---------|
| Authorization | Can this user do this action? | RBAC, Node, Webhook |
| Admission Control | Should this request be allowed/modified? | LimitRanger, PodSecurity, ResourceQuota |

Authorization is binary (allow/deny). Admission control can also **mutate** requests.

---

## Checking Current Authorization Mode

```bash
kubectl describe pod kube-apiserver-controlplane -n kube-system | grep authorization-mode
# --authorization-mode=Node,RBAC
```

---

## Summary: Request Flow

```
kubectl create pod ...
        ↓
1. TLS handshake (encryption)
        ↓
2. Authentication (is this a known user?)
        ↓
3. Authorization (is this user allowed to create pods here?)
        ↓
4. Admission Control (is this pod spec valid/compliant?)
        ↓
5. Write to etcd
        ↓
6. Controller manager + scheduler + kubelet act on it
```
