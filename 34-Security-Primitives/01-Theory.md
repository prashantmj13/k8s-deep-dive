# 34 — Kubernetes Security Primitives

## Security Layers in Kubernetes

Kubernetes security is built in layers — each layer protects against different threats:

```
External Traffic
     ↓
[ Network Firewall / Cloud Security Groups ]
     ↓
[ API Server — AuthN + AuthZ + Admission Control ]
     ↓
[ RBAC — Who can do what ]
     ↓
[ Namespaces — Resource isolation ]
     ↓
[ Pod Security — Security Contexts, PSA ]
     ↓
[ Network Policies — Pod-to-Pod traffic ]
     ↓
[ Secrets & ConfigMaps — Sensitive data ]
     ↓
[ Container Runtime — Seccomp, AppArmor ]
```

---

## The kube-apiserver is the Front Door

All communication in Kubernetes goes through the API server. Securing it means:

1. **Who can access the API?** → Authentication (AuthN)
2. **What can they do?** → Authorization (AuthZ)
3. **Is the request valid?** → Admission Controllers

---

## Authentication Methods

| Method | How |
|--------|-----|
| Static password files | `--basic-auth-file` (deprecated, insecure) |
| Static token files | `--token-auth-file` (deprecated) |
| X.509 certificates | `--client-ca-file` (most common) |
| Service Account tokens | JWT tokens (for pods) |
| OIDC tokens | External identity providers (Google, LDAP) |
| Webhook token | External auth service |

---

## Authorization Modes

Configured on the API server with `--authorization-mode`:

| Mode | Description |
|------|-------------|
| `RBAC` | Role-Based Access Control (recommended) |
| `Node` | Special mode for kubelets |
| `ABAC` | Attribute-Based (legacy, file-based) |
| `Webhook` | External authorization service |
| `AlwaysAllow` | No checks (dev only) |
| `AlwaysDeny` | Deny everything |

Default on kubeadm: `--authorization-mode=Node,RBAC`

---

## TLS Everywhere

All communication in the cluster uses TLS:

| Communication | Certificate |
|---------------|-------------|
| User → API server | Client cert / token |
| API server → etcd | API server client cert |
| API server → kubelet | API server kubelet cert |
| kubelet → API server | Kubelet client cert |
| Controller manager → API server | CM client cert |
| Scheduler → API server | Scheduler client cert |

---

## Admission Controllers

After AuthN and AuthZ, admission controllers validate/mutate requests:

```
Request → AuthN → AuthZ → Mutating AC → Validating AC → etcd
```

Key admission controllers:
- `NamespaceLifecycle` — prevents resource creation in terminating namespaces
- `LimitRanger` — enforces resource limits
- `ResourceQuota` — enforces namespace quotas
- `PodSecurity` — enforces pod security standards
- `ServiceAccount` — auto-assigns service accounts

---

## Securing the Hosts

- Disable password-based SSH, use key-based only
- Disable unnecessary services and ports
- Keep OS patched
- Use CIS Kubernetes Benchmark for hardening

---

## Key Security Checklist

- [ ] RBAC enabled and least-privilege roles
- [ ] Network policies to restrict pod traffic
- [ ] Secrets encrypted at rest
- [ ] Audit logging enabled
- [ ] Image scanning in CI/CD pipeline
- [ ] Non-root containers
- [ ] Read-only root filesystems where possible
- [ ] Pod Security Admission enforced
- [ ] Regular etcd backups
