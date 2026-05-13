# 50 — Kubectx and Kubens | Exercises & Solutions

## Exercises

```bash
# Install
sudo apt install kubectx   # Linux
# brew install kubectx     # macOS

# List contexts
kubectx

# Switch namespace
kubens kube-system
kubectl get pods

# Switch back
kubens -

# List namespaces interactively
kubens

# Compare speed: kubens vs kubectl
time kubens default
time kubectl config set-context --current --namespace=default
```

## Solutions / Expected Outputs

```bash
kubectx
# * kubernetes-admin@kubernetes   ← current (marked with *)
#   staging-cluster

kubens
# default
# kube-node-lease
# kube-public
# * kube-system                   ← current after switching

kubens -
# Switched to namespace "default".
```

## Challenge Answers

**Why use kubectx/kubens over raw kubectl config commands?**
Shorter commands, `-` flag for quick back-switching, and fzf integration for interactive selection — especially valuable when managing 10+ clusters/namespaces daily.

**Is it safe to use in production?**
Yes. They are purely local tools that modify `~/.kube/config` — no cluster-side changes. The `-` switch-back feature is particularly useful to avoid accidentally running commands on the wrong cluster.
