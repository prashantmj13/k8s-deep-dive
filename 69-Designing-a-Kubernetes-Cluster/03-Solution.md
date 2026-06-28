# 69 — Designing a Kubernetes Cluster | Exercises

> **Format:** Design exercises — planning and decision-making, no cluster required
> **Estimated time:** 20 minutes

---

## Exercise 1 — Design a 3-AZ HA cluster on a diagram

Sketch (on paper or in a tool) a cluster with:
- 3 control plane nodes spread across 3 availability zones
- 6 worker nodes, 2 per AZ
- A load balancer in front of the API servers
- Stacked etcd

---

## Exercise 2 — Choose CIDR ranges

Given this existing network:
```
Office network: 10.0.0.0/8
VPN clients:    192.168.50.0/24
```

Propose non-conflicting Pod CIDR and Service CIDR ranges.

---

## Exercise 3 — Calculate node capacity

A node has 8 vCPU and 32GB RAM. Your average pod requests 200m CPU and 256Mi memory.
- How many pods can the node theoretically support based on CPU?
- Based on memory?
- What other limit might cap this lower (hint: default maxPods)?

---

## Exercise 4 — Write an HAProxy config

Write a load balancer config fronting 3 API servers at `10.0.1.10`, `10.0.1.11`, `10.0.1.12` on port 6443.

---

## Exercise 5 — Decision matrix

For each scenario, decide managed vs self-hosted and justify:
1. A startup with no dedicated platform team, needs to ship fast
2. A bank with strict on-prem data residency requirements
3. A research lab needing heavy customization of the scheduler

---

## Challenge Questions

1. Why must the number of control plane nodes be odd?
2. What is the main risk of stacked etcd vs external etcd?
3. Why should pod and service CIDRs never overlap with the physical/VPC network?
4. What single component, if not load-balanced, becomes a single point of failure in an HA cluster?
5. What's the operational cost difference between managed and self-hosted Kubernetes?

---

# Solutions

**Exercise 2:**
```
Pod CIDR:     172.16.0.0/16     (doesn't overlap 10.0.0.0/8 or 192.168.50.0/24)
Service CIDR: 172.17.0.0/16
```
Note: 10.244.0.0/16 would actually CONFLICT here since it's a subset of 10.0.0.0/8 — must pick outside that range.

**Exercise 3:**
- CPU: 8000m / 200m = 40 pods (CPU-bound)
- Memory: 32768Mi / 256Mi = 128 pods (memory-bound)
- Default `maxPods` per node is 110 — this caps it below the memory-based estimate, so **110 is the actual limit** unless you raise maxPods.

**Exercise 4:**
```
frontend k8s-api
    bind *:6443
    mode tcp
    default_backend k8s-api-backend
backend k8s-api-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server cp1 10.0.1.10:6443 check
    server cp2 10.0.1.11:6443 check
    server cp3 10.0.1.12:6443 check
```

**Exercise 5:**
1. Managed (GKE/EKS) — fast to ship, no ops overhead, team can focus on product
2. Self-hosted (kubeadm/on-prem) — data residency rules out cloud-managed control planes
3. Self-hosted — full control over scheduler config, custom plugins, kube-scheduler extender

---

## Challenge Answers

**1. Odd number of control plane nodes?**
etcd uses Raft consensus requiring a majority (quorum) vote to commit writes. With an even number, you can have ties when the cluster splits, and you gain no extra fault tolerance over the next lower odd number (4 nodes tolerate 1 failure, same as 3; but cost more).

**2. Stacked etcd risk?**
If a control-plane node fails, you lose BOTH an API server replica AND an etcd member simultaneously. With 3 stacked nodes, losing 2 nodes (even from unrelated causes) takes etcd below quorum and the cluster becomes unavailable for writes. External etcd isolates this risk.

**3. Why CIDRs must not overlap?**
If pod/service IP ranges overlap with real network ranges (office, VPN, other VPCs), routing becomes ambiguous — packets meant for a pod might get routed to a real host on that subnet (and vice versa), causing connectivity failures that are very hard to debug.

**4. Unprotected SPOF in HA cluster?**
The API server / load balancer endpoint. Even with 3 control plane nodes, if `kubectl`/kubelets only know one static API server IP without a load balancer, that single node failing breaks all cluster access despite the other two being healthy.

**5. Operational cost difference?**
Managed: lower ops burden (provider patches/upgrades the control plane, handles HA), but less flexibility and per-node + control plane fees. Self-hosted: you own all upgrade/patch/backup/HA work (real engineering time cost) but pay only for compute and have full customization freedom.
