# 1. Bootstrap the workshop apps

ArgoCD is already installed on your cluster (see
[prerequisites](./00-prerequisites.md)), so this step just creates the
workshop's namespaces, the Konnect PAT secret, and the root `Application`
that fans out to everything else.

## Create the workshop namespaces

```bash
kubectl apply -f argocd/bootstrap/namespaces.yaml
```

- `kong-operator-system` -- Kong Operator (the controller) runs here, in its
  own dedicated namespace.
- `kong-ai-gateway` -- the AI Gateway `DataPlane` and all Konnect CRs
  (`KonnectAPIAuthConfiguration`, `KonnectGatewayControlPlane`,
  `KonnectExtension`) live here.

## Create the Konnect PAT secret

Before syncing, create the real Secret holding your Konnect PAT (do **not**
commit a real token to git -- `manifests/ai-gateway/00-konnect-pat-secret.yaml`
is a placeholder/reference only and is excluded from the kustomization that
ArgoCD syncs):

```bash
export KONNECT_TOKEN="<your Konnect PAT>"

kubectl create secret generic konnect-pat \
  --namespace kong-ai-gateway \
  --from-literal=token="$KONNECT_TOKEN"

kubectl label secret konnect-pat -n kong-ai-gateway konghq.com/secret=true
```

Kong Operator only reconciles Secrets labeled `konghq.com/secret: "true"` --
without that label the `KonnectAPIAuthConfiguration` will fail to resolve the
token.

## Bootstrap the app-of-apps

```bash
kubectl apply -f argocd/bootstrap/root-app.yaml
```

This creates one root `Application` (`kong-ai-gateway-workshop`) in the
`argocd` namespace that auto-discovers and syncs every Application manifest
in `argocd/apps/`:

| Order (sync-wave) | Application            | What it deploys                                                        |
|--------------------|-------------------------|--------------------------------------------------------------------------|
| -2                 | `kong-operator-crds`    | Kong Operator / Konnect CRDs (DataPlane, KonnectExtension, etc.)        |
| -1                 | `kong-operator`         | Kong Operator controller, Helm chart, into `kong-operator-system`       |
| 0                  | `kong-ai-gateway`       | Konnect auth/control-plane/extension + AI Gateway `DataPlane`           |

Watch them come up:

```bash
kubectl -n argocd get applications -w
```

Next: [2. Connect to Konnect](./02-connect-konnect.md)
