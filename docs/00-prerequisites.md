# 0. Prerequisites

This workshop uses Kong Operator to deploy a Kong AI Gateway 2.2.0 data
plane, connected to a **Konnect** control plane, driven entirely through
**ArgoCD**.

It assumes you already have:

- A working Kubernetes cluster, with `kubectl` configured to point at it
  (local or otherwise -- `kind`/`minikube`/managed cluster all work the same
  way; nothing here is cluster-type-specific).
- **ArgoCD already installed** on that cluster, with its `argocd` namespace
  up and the `argocd-server`/`argocd-repo-server`/`argocd-application-controller`
  pods running.
- Enough cluster-admin-ish RBAC to create namespaces, CRDs, and ArgoCD
  `Application` resources.

Quick sanity check before you start:

```bash
kubectl cluster-info
kubectl -n argocd get pods
```

## Tools

- `kubectl`
- `git` -- push this workshop directory to a repo ArgoCD can reach (a public
  GitHub repo, or any git server reachable from your cluster)
- `helm` (optional, only needed if you want to inspect/template the chart
  locally)
- `argocd` CLI (optional, the UI/`kubectl` are enough)

## Konnect account

1. Sign up / log in at <https://cloud.konghq.com>.
2. Create a **Personal Access Token (PAT)**: Konnect UI -> top-right account
   menu -> **Personal Access Tokens** -> **Generate Token**. Copy it
   somewhere safe -- you'll use it in step 1.

## Repo setup

1. Push this directory to a git repo (fork or new repo) that your cluster's
   ArgoCD instance can reach.
2. Edit the two `REPO_URL` / `TARGET_REVISION` placeholders to point at your
   repo/branch:
   - `argocd/bootstrap/root-app.yaml`
   - `argocd/apps/03-ai-gateway.yaml`

Next: [1. Bootstrap the workshop apps](./01-install-argocd.md)
