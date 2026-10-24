# 2. Connect to Konnect

Once `kong-operator-crds` and `kong-operator` are `Synced`/`Healthy`, the
`kong-ai-gateway` Application will apply, in order:

1. `01-konnect-api-auth.yaml` -- `KonnectAPIAuthConfiguration` that
   authenticates the cluster to Konnect using the `konnect-pat` Secret you
   created in step 1.
2. `02-konnect-control-plane.yaml` -- `KonnectAIGateway` that creates a
   brand-new **AI Gateway**-typed control plane in Konnect named
   `ai-gateway-control-plane`. This is a distinct Konnect entity type from a
   regular Gateway control plane (`KonnectGatewayControlPlane`), and is
   required for AI Gateway-specific resources (`AIGatewayModelProvider`,
   `AIGatewayModel`) if you extend this workshop with LLM routing.
3. `04-dataplane.yaml` -- the `AIGatewayDataPlane`, running
   `kong/kong-ai-gateway:2.2.0`, wired directly to the `KonnectAIGateway`
   above via `spec.controlPlaneRef`. Kong Operator provisions and manages
   the mTLS client certificate automatically -- no separate
   `KonnectExtension` resource is needed for this path (that's only used
   with the generic gateway-operator `DataPlane` kind).

## Verify auth succeeded

```bash
kubectl -n kong-ai-gateway get konnectapiauthconfiguration konnect-api-auth
```

Expect `VALID: True`:

```
NAME               VALID   ORGID                                  SERVERURL
konnect-api-auth   True    <your-org-id>                          https://au.api.konghq.com
```

## Verify the control plane was created

```bash
kubectl -n kong-ai-gateway get konnectaigateways.konnect.konghq.com
```

Expect `PROGRAMMED: True`. You should also see `ai-gateway-control-plane`
appear under the **AI Gateway** section (not Gateway Manager) in the Konnect
UI -- it's a different Konnect entity type from a regular Gateway control
plane.

## Verify the AIGatewayDataPlane

```bash
kubectl -n kong-ai-gateway get aigatewaydataplanes.aigateway.konghq.com ai-gateway-dataplane
```

Expect the `Ready` condition to be `True`:

```bash
kubectl -n kong-ai-gateway get aigatewaydataplane ai-gateway-dataplane \
  -o=jsonpath='{.status.conditions[?(@.type=="Ready")]}' | jq
```

You should also see conditions `KonnectAIGatewayResolved`,
`CertificateProvisioned` and `KonnectCertificateRegistered` all `True` --
these confirm the mTLS cert was auto-provisioned and registered against the
`KonnectAIGateway` control plane.

If the `AIGatewayDataPlane` is `Ready`, check the Konnect UI -- under the
`ai-gateway-control-plane` AI Gateway, you should see a connected data plane
node.

Next: [3. Verify the AI Gateway is working](./03-verify.md)
