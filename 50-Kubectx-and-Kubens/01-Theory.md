# 50 — Kubectx and Kubens

## What are They?

`kubectx` and `kubens` are **command-line utilities** that make switching between Kubernetes contexts and namespaces faster and easier. They are thin wrappers around `kubectl config` commands.

| Tool | Purpose |
|------|---------|
| `kubectx` | Switch between kubeconfig contexts (clusters) |
| `kubens` | Switch between namespaces |

---

## Installation

```bash
# Via package manager (macOS)
brew install kubectx

# Via apt (Ubuntu/Debian)
sudo apt install kubectx

# Via git (all platforms)
git clone https://github.com/ahmetb/kubectx ~/.kubectx
sudo ln -s ~/.kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s ~/.kubectx/kubens /usr/local/bin/kubens

# Verify
kubectx --version
kubens --version
```

---

## kubectx — Context Switching

```bash
# List all contexts (current marked with *)
kubectx

# Switch to a context
kubectx my-cluster

# Switch back to previous context
kubectx -

# Rename a context
kubectx new-name=old-name

# Delete a context
kubectx -d old-context

# Create alias for a long context name
kubectx prod=gke_my-project_us-central1_prod-cluster
kubectx prod   # now just type 'kubectx prod'
```

---

## kubens — Namespace Switching

```bash
# List all namespaces (current marked with *)
kubens

# Switch to a namespace
kubens kube-system

# Switch back to previous namespace
kubens -
```

---

## kubectl Equivalents

| kubectx/kubens | kubectl equivalent |
|----------------|-------------------|
| `kubectx` | `kubectl config get-contexts` |
| `kubectx my-cluster` | `kubectl config use-context my-cluster` |
| `kubectx -` | `kubectl config use-context $(previous)` |
| `kubens` | `kubectl get namespaces` |
| `kubens kube-system` | `kubectl config set-context --current --namespace=kube-system` |
| `kubens -` | revert to previous namespace |

---

## Interactive Mode with fzf

If `fzf` (fuzzy finder) is installed, both tools support **interactive fuzzy search**:

```bash
# Install fzf
brew install fzf
# or
apt install fzf

# Now just type kubectx or kubens with no args
# → opens an interactive picker to search and select
kubectx    # fuzzy search across all contexts
kubens     # fuzzy search across all namespaces
```

---

## Shell Prompt Integration

Show current context/namespace in your shell prompt by installing **kube-ps1**:

```bash
# Install
brew install kube-ps1

# Add to ~/.bashrc or ~/.zshrc
source "/usr/local/opt/kube-ps1/share/kube-ps1.sh"
PS1='$(kube_ps1)'$PS1

# Result: prompt shows
(⎈ my-cluster:production) $
```

---

## Exercises

```bash
# Exercise 1: Install kubectx and kubens
sudo apt install kubectx   # or brew install kubectx

# Exercise 2: List contexts
kubectx

# Exercise 3: Switch namespace to kube-system
kubens kube-system
kubectl get pods   # shows kube-system pods

# Exercise 4: Switch back
kubens -
kubectl get pods   # back to previous namespace

# Exercise 5: List namespaces
kubens

# Exercise 6: Compare with kubectl
kubectl config get-contexts
kubectl config current-context
```

---

## Key Benefits

- **Speed**: `kubens production` vs `kubectl config set-context --current --namespace=production`
- **Safety**: Easily see and switch the active context — reduces accidental commands on wrong cluster
- **History**: `-` flag to switch back is muscle memory for multi-cluster work
