# 3. Verify the AI Gateway is working

## Check pods

```bash
kubectl -n kong-operator-system get pods      # the Kong Operator controller
kubectl -n kong-ai-gateway get pods           # the AI Gateway proxy pod(s)
```

## Reach the proxy

The `DataPlane`'s ingress Service defaults to `ClusterIP`. Find it and
port-forward:

```bash
kubectl -n kong-ai-gateway get svc -l app=ai-gateway-dataplane

kubectl -n kong-ai-gateway port-forward svc/<ingress-service-name> 8000:80
```

Then, in another terminal:

```bash
curl -i http://localhost:8000/
# Expect a 404 from Kong ("no Route matched") -- this confirms the proxy
# itself is up and responding; routes/AI plugins are configured next.
```

> Prefer not to port-forward? Patch the ingress Service type once the
> DataPlane is `Ready`: `spec.network.services.ingress.type: NodePort` or
> `LoadBalancer`, depending on what your cluster supports.

## Next steps (beyond this workshop)

- Configure an AI Proxy / AI Semantic plugin + `Route`/`Service` (or a
  Gateway API `Gateway`/`HTTPRoute`) pointing at an LLM provider, and attach
  your provider credentials as a Konnect vault/secret.
- Import/manage that configuration from Konnect's **Gateway Manager**, since
  the data plane is now hybrid-connected to your Konnect control plane.
- See <https://developer.konghq.com/ai-gateway/> for plugin configuration.

## Tear down (workshop resources only)

```bash
kubectl delete -f argocd/bootstrap/root-app.yaml
kubectl delete -f argocd/bootstrap/namespaces.yaml
```

This leaves ArgoCD and the underlying cluster untouched -- only the
workshop's Applications and namespaces are removed.

(Remember to also delete the `ai-gateway-control-plane` control plane in the
Konnect UI if you no longer need it.)
