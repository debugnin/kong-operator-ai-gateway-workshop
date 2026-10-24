# Deploying Kong AI Gateway on Kubernetes: Kong Operator vs. the Standalone Helm Chart

## Purpose

This document compares two supported ways to run **Kong AI Gateway** on
Kubernetes against a **Konnect** control plane, so you can choose the
right one for your environment:

1. **Kong Operator** -- `KonnectAIGateway` + `AIGatewayDataPlane` custom
   resources, via the `kong/kong-operator` Helm chart.
2. **Standalone Helm chart** -- `kong/kong-ai-gateway`, a direct
   data-plane-only chart derived from the long-standing `kong/kong` chart.

Both are official, Kong-supported paths. They are **not functionally
equivalent** -- they draw the line between "what Kubernetes manages" and
"what Konnect/you manage" in different places. Picking the right one
depends on your team's operating model, not just feature checkboxes.

## TL;DR

| If you... | Choose |
|---|---|
| Want the Konnect control plane, certificates, and autoscaling all declared and reconciled from Kubernetes | **Kong Operator** |
| Already provision Konnect control planes centrally (UI, API, Terraform) and just need to run data planes | **Standalone chart** |
| Want the smallest possible footprint and fewest moving parts | **Standalone chart** |
| Are comfortable operating a cluster-wide controller + CRDs, and want everything in git/GitOps | **Kong Operator** |
| Need production stability today and want to avoid rapid-release/prerelease components | **Standalone chart** (evaluate Operator's AI Gateway support maturity first) |
| Run other Kong products (Gateway, mesh) already via Kong Operator | **Kong Operator** (consistent model) |

## What each option actually manages

| Capability | Kong Operator | Standalone `kong-ai-gateway` chart |
|---|---|---|
| Creates the Konnect **AI Gateway control plane** | Yes -- `KonnectAIGateway` CR | No -- must already exist; create it manually in Konnect (UI, API, or your own IaC) first |
| Creates AI **model providers / models** in Konnect | Yes -- `AIGatewayModelProvider` / `AIGatewayModel` CRs | No -- out of scope; configure separately in Konnect |
| Deploys the AI Gateway **data-plane workload** | Yes -- `AIGatewayDataPlane` CR | Yes -- this is the chart's entire job |
| mTLS client certificate (DP \<-\> Konnect) | Auto-generated, auto-rotated, auto-registered to Konnect | You provide it: a hand-created Secret, or your own cert-manager wiring |
| Ongoing reconciliation / drift correction | Continuous -- a controller watches and corrects drift | None -- `helm upgrade` is a one-shot action; nothing watches afterward |
| Requires installing CRDs | Yes -- a dedicated CRD install/upgrade step, version-locked to the operator | No -- zero CRDs |
| Requires a cluster-wide controller pod | Yes | No |

## Kong Operator

### How it works

You declare three Kubernetes custom resources; the operator's controller
reconciles each continuously against both the Kubernetes API and the
Konnect API:

```
KonnectAPIAuthConfiguration   -- authenticates the cluster to Konnect (PAT)
        |
KonnectAIGateway              -- creates/owns the AI Gateway control plane in Konnect
        |
AIGatewayDataPlane            -- runs the AI Gateway binary; wired to the
                                  control plane above via controlPlaneRef
```

The operator automatically:
- Generates a per-DataPlane mTLS client certificate signed by its own
  cluster CA, re-signs it on expiry (configurable TTL/margin), and
  registers the public certificate with Konnect on every change.
- Provisions and maintains a `HorizontalPodAutoscaler` if you declare
  `spec.deployment.scaling.horizontal`, including drift correction if it's
  edited or removed out of band.
- Cascades deletes: removing the `AIGatewayDataPlane` deregisters its
  certificate from Konnect; removing the `KonnectAIGateway` deprovisions
  the control plane itself from Konnect.

### Pros

1. **Single source of truth, fully Kubernetes-native.** Control plane
   lifecycle, certificates, and data-plane workload are all declared and
   reconciled from the same set of Kubernetes objects -- no out-of-band
   steps to remember (e.g. deleting a stale control plane in the Konnect
   UI after a `helm uninstall`).
2. **Zero manual certificate handling.** CA bootstrap, per-DataPlane
   signing, expiry tracking, background rotation, and re-registration with
   Konnect are all automatic.
3. **Declarative autoscaling as a first-class field,** not a hand-maintained
   side-car manifest -- the HPA is generated, kept in sync, and removed
   automatically if you stop requesting it.
4. **GitOps-friendly by construction.** Custom resources are plain
   Kubernetes objects; tools like ArgoCD/Flux can own them with standard
   drift detection, the same way they'd manage any other workload.
5. **One consistent operational model across Kong products.** If you also
   run Kong Gateway, Ingress, or mesh via Kong Operator, AI Gateway uses
   the same CRD vocabulary and reconciler patterns -- less to learn per
   product line.
6. **Config mistakes caught earlier.** CRD schemas and admission-time
   validation rules (e.g. a fixed `replicas` count and autoscaling
   `scaling` config are mutually exclusive and rejected together) catch
   misconfigurations before they ever reach the cluster.

### Cons

1. **More moving parts.** A full setup requires: installing the operator's
   CRDs (a separate, version-pinned step), installing the operator Helm
   chart, enabling feature-gate environment variables, and understanding
   three separate custom resources and their individual reconcile chains,
   versus one `helm install` for the chart.
2. **The operator itself becomes something you operate.** You're now
   responsible for its own lifecycle: Helm chart upgrades, keeping CRD
   versions locked to the controller version, its own CA secret, its RBAC
   footprint, and its webhook configuration. If the operator pod is down,
   nothing reconciles. Its Deployment uses a `Recreate` update strategy, so
   controller upgrades include a brief downtime window for the controller
   itself (though already-running data planes are unaffected).
3. **More indirection when troubleshooting.** Diagnosing an issue means
   walking a chain of status conditions across multiple resource kinds
   (e.g. control-plane-reference-resolved -\> certificate-provisioned -\>
   certificate-registered-with-Konnect -\> ready) instead of reading one
   Deployment's events and logs directly.
4. **Rapid-release maturity risk.** At the time of writing, AI Gateway
   support in Kong Operator ships on **rapid-release** chart/CRD lines
   (explicitly marked prerelease), not the GA release train. Teams with
   strict production-stability requirements should evaluate maturity
   before committing.
5. **Less direct access to raw Kong configuration primitives.** The
   standalone chart's `env.*` values map 1:1 to the `KONG_*` environment
   variables Kong Gateway understands natively. The operator abstracts
   this behind CRD fields, and anything not explicitly modeled by the CRD
   schema has to go through a Kubernetes `podTemplateSpec` strategic-merge
   patch -- an extra translation layer for anything outside the common
   path (e.g. setting the container image/version or CPU/memory requests).
6. **The effective running configuration isn't fully visible from your
   manifest alone.** Operator-side defaults (e.g. a specific AI Gateway
   image version baked into the operator binary, used whenever you don't
   override it) mean you sometimes have to inspect the generated
   `Deployment` directly to see ground truth.

## Standalone `kong/kong-ai-gateway` Helm chart

### How it works

A single Helm release deploys the AI Gateway data-plane binary directly,
connecting to a Konnect AI Gateway control plane that you've already
created. It is derived from the mature `kong/kong` chart and shares most of
its configuration surface (`env`, `proxy`, `admin`, `secretVolumes`,
`certificates`, etc.), with Kong Ingress Controller, Postgres/migrations,
Kong Manager, and the Developer Portal removed since they don't apply to a
Konnect-only AI Gateway data plane.

Prerequisites you must satisfy yourself before installing:
- A Konnect AI Gateway control plane already created in Konnect.
- A client mTLS certificate/key pair Konnect is configured to trust for
  that control plane -- either a Secret you create yourself, or issued
  in-cluster via your own cert-manager setup.

### Pros

1. **Minimal and transparent.** One Helm release, familiar `env`/`proxy`
   configuration surface for anyone who already runs Kong Gateway. What
   you template in values is what gets deployed -- no controller
   reinterpreting it afterward.
2. **No cluster-wide dependency.** No operator pod, no CRDs, no
   cluster-scoped controller RBAC. Smaller blast radius: fewer components
   that can fail and block reconciliation of something else.
3. **Clean separation of concerns.** Naturally fits organizations where a
   platform team provisions Konnect control planes centrally (via the
   Konnect UI, API, or their own Terraform/IaC) and application teams only
   need to deploy data planes against them.
4. **Simple upgrade story for the data plane.** A chart/image version bump
   and `helm upgrade`; no CRD/controller version-compatibility matrix to
   track.

### Cons

1. **No Konnect-side automation.** Creating the AI Gateway control plane
   and any model/provider configuration happens entirely outside this
   chart (Konnect UI, API, or separate IaC) -- more tools in the loop, and
   easier for the Kubernetes and Konnect sides to drift out of sync with
   each other.
2. **You own the certificate lifecycle.** No automatic rotation and no
   automatic re-registration with Konnect when a certificate changes.
   Without wiring in your own cert-manager integration, rotation is a
   manual, time-boxed responsibility.
3. **No built-in autoscaling abstraction.** You maintain your own
   `HorizontalPodAutoscaler` object alongside the release; nothing
   corrects it if it's edited or deleted out of band.
4. **No drift correction for the control-plane relationship.** If the
   Konnect control plane this chart points to is deleted or
   misconfigured, the chart has no way to detect or report that --
   failures surface only as data-plane connection errors.

## Decision guidance

**Choose Kong Operator if:**
- You want Kubernetes to be the single declarative source of truth for
  both the Konnect control plane and the data plane.
- You're already investing in Kong Operator for other Kong products.
- Your team is comfortable running and upgrading a cluster-scoped
  controller and its CRDs as part of your platform.
- Automatic certificate rotation and declarative autoscaling are
  meaningful operational wins for you.
- You've evaluated and accepted the current maturity level of AI
  Gateway's rapid-release support in the operator.

**Choose the standalone Helm chart if:**
- Your organization already has an established, separate process for
  provisioning Konnect control planes (UI/API/Terraform) and you don't
  want Kubernetes managing that relationship.
- You want the smallest possible footprint and the fewest moving
  parts/failure domains.
- You need a GA-track, well-established deployment path today rather
  than a rapid-release one.
- Your team already has its own certificate management process (e.g.
  cert-manager) it wants to reuse as-is.

## Notes for migration between the two

- Moving from the standalone chart to Kong Operator means importing (or
  recreating) the existing Konnect control plane as a `KonnectAIGateway`
  resource, and replacing the chart release with an `AIGatewayDataPlane`.
  Certificates should be regenerated under the operator's management
  rather than reused, to get the benefit of automatic rotation.
- Moving from Kong Operator to the standalone chart means you take over
  ownership of the Konnect control plane and certificate lifecycle that
  the operator was managing -- make sure a replacement process is in place
  *before* decommissioning the operator-managed resources, to avoid an
  unmanaged gap.

## References

- Kong Operator AI Gateway deployment guide:
  <https://developer.konghq.com/operator/get-started/ai-gateway/deploy/>
- Kong Operator install guide (AI Gateway):
  <https://developer.konghq.com/operator/get-started/ai-gateway/install/>
- Kong Operator overview:
  <https://developer.konghq.com/operator/>
- Standalone `kong-ai-gateway` chart:
  <https://artifacthub.io/packages/helm/kong/kong-ai-gateway>
- Kubernetes Operators vs. Helm (general background):
  <https://konghq.com/blog/learning-center/kubernetes-operators-vs-helm>
