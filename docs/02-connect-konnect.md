# 2. Connect to Konnect

Once `kong-operator-crds` and `kong-operator` are `Synced`/`Healthy`, the
`kong-ai-gateway` Application will apply, in order:

1. `01-konnect-api-auth.yaml` -- `KonnectAPIAuthConfiguration` that
   authenticates the cluster to Konnect using the `konnect-pat` Secret you
   created in step 1.
2. `02-konnect-control-plane.yaml` -- `KonnectGatewayControlPlane` that
   creates a brand-new **Hybrid mode** control plane in Konnect named
   `ai-gateway-control-plane`.
3. `03-konnect-extension.yaml` -- `KonnectExtension` that links a `DataPlane`
   to that control plane and lets Kong Operator provision/manage the mTLS
   client certificate automatically.
4. `04-dataplane.yaml` -- the `DataPlane` itself, running
   `kong/kong-ai-gateway:2.2.0`.

## Verify auth succeeded

```bash
kubectl -n kong-ai-gateway get konnectapiauthconfiguration konnect-api-auth
```

Expect `VALID: True`:

```
NAME               VALID   ORGID                                  SERVERURL
konnect-api-auth   True    <your-org-id>                          https://us.api.konghq.com
```

## Verify the control plane was created

```bash
kubectl -n kong-ai-gateway get konnectgatewaycontrolplanes.konnect.konghq.com
```

Expect `PROGRAMMED: True`. You should also see `ai-gateway-control-plane`
appear under **Gateway Manager** in the Konnect UI.

## Verify the extension and DataPlane

```bash
kubectl -n kong-ai-gateway get konnectextensions.konnect.konghq.com
kubectl -n kong-ai-gateway get dataplane ai-gateway-dataplane
```

Expect `READY: True` on the extension, and the `DataPlane`'s `Ready`
condition to be `True`:

```bash
kubectl -n kong-ai-gateway get dataplane ai-gateway-dataplane \
  -o=jsonpath='{.status.conditions[?(@.type=="Ready")]}' | jq
```

If the `DataPlane` is `Ready`, check the Konnect UI -- under the
`ai-gateway-control-plane` control plane, you should see a connected data
plane node.

Next: [3. Verify the AI Gateway is working](./03-verify.md)
