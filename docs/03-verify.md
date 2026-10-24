# 3. Verify the AI Gateway is working

## Check pods

```bash
kubectl -n kong-operator-system get pods      # the Kong Operator controller
kubectl -n kong-ai-gateway get pods           # the AI Gateway proxy pod(s)
```

## Reach the proxy

The `AIGatewayDataPlane`'s ingress Service defaults to `ClusterIP`
(`ai-gateway-dataplane-ingress`), exposing port `8000`:

```bash
kubectl -n kong-ai-gateway get svc ai-gateway-dataplane-ingress

kubectl -n kong-ai-gateway port-forward svc/ai-gateway-dataplane-ingress 8000:8000
```

Then, in another terminal:

```bash
curl -i http://localhost:8000/
# Expect a 404 from Kong ("no Route matched") -- this confirms the proxy
# itself is up and responding; routes/AI plugins are configured next.
```

> Prefer not to port-forward? Patch the ingress Service type once the
> `AIGatewayDataPlane` is `Ready`: `spec.network.services.ingress.type:
> NodePort` or `LoadBalancer`, depending on what your cluster supports.

## Next steps (beyond this workshop)

- Configure an AI Proxy / AI Semantic plugin + `Route`/`Service` (or a
  Gateway API `Gateway`/`HTTPRoute`) pointing at an LLM provider, and attach
  your provider credentials as a Konnect vault/secret.
- Import/manage that configuration from Konnect's **AI Gateway** section,
  since the data plane is now connected to your `KonnectAIGateway` control
  plane -- see
  <https://developer.konghq.com/operator/get-started/ai-gateway/deploy/>
  for adding `AIGatewayModelProvider`/`AIGatewayModel` resources.
- See <https://developer.konghq.com/ai-gateway/> for plugin configuration.

## Tear down (workshop resources only)

```bash
kubectl delete -f argocd/bootstrap/root-app.yaml
kubectl delete -f argocd/bootstrap/namespaces.yaml
```

This leaves ArgoCD and the underlying cluster untouched -- only the
workshop's Applications and namespaces are removed.

(Remember to also delete the `ai-gateway-control-plane` AI Gateway in the
Konnect UI if you no longer need it.)
