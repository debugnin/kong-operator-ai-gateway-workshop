# Kong Operator + Kong AI Gateway 2.2.0 Workshop

An interactive, GitOps-driven workshop showing how to use **Kong Operator**
to deploy a **Kong AI Gateway** data plane connected to a **Konnect AI
Gateway** control plane -- everything managed through **ArgoCD**.

This workshop assumes you already have a Kubernetes cluster and an existing
ArgoCD installation on it; it only adds namespaces, Applications, and the
AI Gateway/Konnect resources on top.

## What you'll build

```
your existing k8s cluster
├── namespace: argocd (pre-existing)
│   └── app-of-apps -> 3 child Applications, ordered via sync-waves
├── namespace: kong-operator-system
│   ├── Kong Operator CRDs           (sync-wave -2)
│   └── Kong Operator controller      (sync-wave -1, Helm chart)
└── namespace: kong-ai-gateway
    └── (sync-wave 0)
        ├── KonnectAPIAuthConfiguration  (auth to Konnect via PAT)
        ├── KonnectAIGateway             (new AI Gateway-typed CP in Konnect: ai-gateway-control-plane)
        └── AIGatewayDataPlane           (runs the AI Gateway binary; auto mTLS via controlPlaneRef)
```

- **Kong Operator** runs isolated in its own namespace (`kong-operator-system`),
  separate from the gateway workload.
- The **AIGatewayDataPlane** runs the AI Gateway binary directly (no
  separate `KonnectExtension`/generic `DataPlane` needed -- that pattern is
  only for the regular gateway-operator `DataPlane` kind, paired with a
  `KonnectGatewayControlPlane`).
- A new **AI Gateway control plane is created in Konnect** via
  `KonnectAIGateway` -- this is a distinct Konnect entity type from a
  regular (`KonnectGatewayControlPlane`) control plane, and is required for
  the AI Gateway-specific resources (`AIGatewayModelProvider`,
  `AIGatewayModel`) used to configure LLM routing.
- Everything is deployed as **ArgoCD Applications**: one for the Kong
  Operator CRDs, one for Kong Operator itself (Helm chart), one for the
  Konnect/AI Gateway resources.

## Layout

```
.
├── argocd/
│   ├── bootstrap/
│   │   ├── namespaces.yaml           # kong-operator-system, kong-ai-gateway
│   │   └── root-app.yaml             # app-of-apps root Application
│   └── apps/
│       ├── 01-kong-operator-crds.yaml  # CRDs Application (sync-wave -2)
│       ├── 02-kong-operator.yaml       # Kong Operator Helm chart (sync-wave -1)
│       └── 03-ai-gateway.yaml          # AI Gateway/Konnect manifests (sync-wave 0)
├── manifests/ai-gateway/             # synced by the 03-ai-gateway Application
│   ├── 00-konnect-pat-secret.yaml    # placeholder/reference only, not synced
│   ├── 01-konnect-api-auth.yaml
│   ├── 02-konnect-control-plane.yaml # KonnectAIGateway (AI Gateway-typed CP)
│   ├── 04-dataplane.yaml             # AIGatewayDataPlane
│   └── kustomization.yaml
└── docs/
    ├── 00-prerequisites.md
    ├── 01-install-argocd.md
    ├── 02-connect-konnect.md
    └── 03-verify.md
```

## Quick start

See the full walkthrough in [`docs/`](./docs/00-prerequisites.md), or at a
glance (assumes an existing cluster + existing ArgoCD):

```bash
# 0. tools + Konnect PAT -- see docs/00-prerequisites.md
# edit REPO_URL/TARGET_REVISION in argocd/bootstrap/root-app.yaml
# and argocd/apps/03-ai-gateway.yaml first, then push this repo

# 1. namespaces + Konnect PAT secret + bootstrap -- see docs/01-install-argocd.md
kubectl apply -f argocd/bootstrap/namespaces.yaml
kubectl create secret generic konnect-pat -n kong-ai-gateway --from-literal=token="$KONNECT_TOKEN"
kubectl label secret konnect-pat -n kong-ai-gateway konghq.com/secret=true

kubectl apply -f argocd/bootstrap/root-app.yaml

# 2 & 3. verify -- see docs/02-connect-konnect.md and docs/03-verify.md
kubectl -n argocd get applications -w
```

## Notes / customization

- **Chart/CRD versions pinned**: `kong-operator` Helm chart `1.5.0-rapid.2.0`
  (operator `2.4.0-rapid.2.0`) and Kong Operator/Konnect CRDs from
  `Kong/kong-operator@v2.4.0-rapid.2.0`. Bump both together when you
  upgrade -- check the [version support
  matrix](https://developer.konghq.com/operator/) first.
- **CRD ownership**: the Kong Operator Helm chart can install its own CRDs
  via its `ko-crds` subchart, but this workshop disables that
  (`ko-crds.enabled: false`) and manages CRDs as a dedicated, independently
  synced/versioned ArgoCD Application instead (`01-kong-operator-crds.yaml`),
  per the requirement to have a distinct Application for the AI Gateway DP
  CRDs.
- **Image**: `AIGatewayDataPlane` doesn't take an explicit image field by
  default in this setup -- Kong Operator selects the AI Gateway runtime
  image itself. To pin a specific version, set it via
  `spec.deployment.podTemplateSpec`. Check
  <https://hub.docker.com/r/kong/kong-ai-gateway/tags> for available tags.
- **Secrets**: the Konnect PAT is never committed to git -- create it
  manually (or via a secrets manager/External Secrets Operator in a real
  deployment) before the `kong-ai-gateway` Application syncs.
