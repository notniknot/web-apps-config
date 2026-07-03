# Preview Environment Lab Results

Last updated: 2026-07-03.

This document records the lab result for self-service preview environments using
`notniknot/web-apps`, `notniknot/web-apps-config`, `notniknot/argocd`, and
`notniknot/preview-controller`.

## Current Direction

- Use normal host-cluster namespaces for preview environments first.
- Do not use vCluster in the current implementation path.
- Let Argo CD deploy generated preview `Application` resources into dedicated
  namespaces such as `web-preview-pr-3`.
- Keep platform-owned resources in the host cluster and generated through GitOps.
- Keep tenant-owned preview configuration inside the tenant config repository.
- Use the controller as the automation layer that reacts to GitHub PR labels and
  writes generated GitOps resources.

## Repository Contract

- Code repository: `notniknot/web-apps`.
- Tenant config repository: `notniknot/web-apps-config`.
- Root/Argo CD config repository: `notniknot/argocd`.
- Controller repository: `notniknot/preview-controller`.
- Preview label: `preview/playground`.
- App-local preview contract: `apps/web/preview.yaml`.
- App-local preview components: `apps/web/previews/components`.
- App-local preview patches: `apps/web/previews/patches`.
- Stable app structure: `apps/<app>/base`, `apps/<app>/<env>`, and
  `apps/<app>/<app>-<env>.argocd.yaml`.
- Preview namespace pattern: `<tenant>-preview-pr-<number>`.
- Preview Argo CD application pattern: `<app>-apps-preview-pr-<number>`.

## PreviewEnvironment Contract

`apps/web/preview.yaml` is now a CRD-shaped document:

```yaml
apiVersion: preview.sonia.so/v1alpha1
kind: PreviewEnvironment
metadata:
  name: web-playground
spec:
  configRepository: notniknot/web-apps-config
  application:
    path: web-playground.argocd.yaml
  sourcePolicy: include-all
  sources:
    - index: 0
      action: keep
      kustomize:
        components:
          - ../previews/components
        patches:
          - target:
              group: gateway.networking.k8s.io
              kind: HTTPRoute
              name: web
            path: previews/patches/httproute-hostname.yaml
    - index: 1
      action: keep
      helm:
        valuesObject:
          message: preview-pr-{{.PR}}
```

Notes:

- `configRepository` uses the full GitHub name, for example
  `notniknot/web-apps-config`.
- `app` and `tenant` are intentionally not part of the tenant-owned
  `PreviewEnvironment` spec.
- `app` and `tenant` remain in the controller allowlist because they are identity
  and authorization inputs used for names, namespaces, labels, and template data.
- `application.path` points to the existing Argo CD Application YAML in the
  tenant config repository.
- `sources[*].index` maps to `spec.sources[index]` of that Argo CD Application.
- Kustomize patches can be inline or loaded from files under
  `apps/<app>/previews/patches`.
- Helm sources can receive extra `valueFiles` and inline `valuesObject`.
- `$values/...` Helm value file references can be supported through `valueRefs`.

## Controller Behavior

- The controller polls allowlisted GitHub repositories.
- For an open PR with `preview/playground`, it reads the app-local
  `PreviewEnvironment` contract from the tenant config repository.
- It renders a generated Argo CD `Application` into `notniknot/argocd`.
- It renders host-side platform resources such as namespace and generated
  `ExternalSecret` manifests into `notniknot/argocd`.
- It updates the preview-platform kustomization to include generated platform
  files.
- For a PR without the label, or a closed PR, it removes generated tenant and
  platform files.
- Cleanup is idempotent: later polls do not create additional Git commits after
  the generated state is gone.
- Info logs are emitted only when Git state actually changes. Steady-state
  no-op reconcile/cleanup messages are debug-level.

The controller currently uses polling. GitHub webhooks remain a future
improvement.

## Verified Use Cases

- Repository bootstrap is working.
- Tenant app-of-apps is working.
- Stable `web/web` app in the parent cluster is working.
- `PreviewEnvironment` CRD is installed in the lab cluster.
- Preview controller is built and pushed as
  `ghcr.io/notniknot/preview-controller:latest`.
- Controller deployment in `preview-controller` namespace is healthy.
- PR label detection works for `preview/playground`.
- Generated preview Argo CD apps were created for PRs `1` and `3`.
- Generated preview apps reached `Synced` and `Healthy`.
- Dedicated preview namespaces were created for PRs `1` and `3`.
- Removing the label from PRs `1` and `3` cleaned up:
  - generated tenant Argo CD Application files
  - generated preview-platform files
  - generated Argo CD Application objects
  - preview namespaces `web-preview-pr-1` and `web-preview-pr-3`
- Root, preview-platform, and tenant app-of-apps returned to `Synced` and
  `Healthy`.
- Controller log-noise fix was deployed and verified: with no active preview
  labels, logs stayed quiet across an idle poll interval.

## Multi-Source Argo CD Applications

The lab app uses `spec.sources`.

The preview controller copies the source list from the referenced Argo CD
Application and mutates sources by index:

- `sourcePolicy: include-all` keeps all sources unless a source rule says
  `action: drop`.
- `sourcePolicy: include-listed` keeps only explicitly listed source indexes.
- Kustomize mutations are applied only to selected Kustomize sources.
- Helm mutations are applied only to selected Helm sources.
- The generated preview Application still emits a compatibility `spec.source`
  copied from the first source because the root app uses server-side dry-run and
  rejected child Applications when `spec.source.repoURL` was absent.

## Image Updates

- The code repository builds PR images in GHCR.
- Preview tags use the PR number and short commit suffix pattern.
- Argo CD Image Updater is used in `argocd` write-back mode for preview apps.
- Preview patches switch Image Updater away from Git write-back so it updates
  the generated Argo CD Application instead of mutating tenant Git.
- Root ignore rules are required for preview Applications labeled
  `preview.sonia.so/enabled=true` so Image Updater-owned fields do not create
  Argo CD drift.

Open item:

- Make private GHCR package visibility and pull-secret behavior fully
  production-realistic. The lab has a pull secret path, but GHCR package
  visibility still needs a final private-package rehearsal.

## Secrets

Verified patterns:

- Shared secret path: ESO creates a Secret in the stable tenant namespace, and
  mittwald/kubernetes-replicator copies it into preview namespaces.
- Generated secret path: the controller/platform can generate an `ExternalSecret`
  directly in the preview namespace.
- Pull secrets can follow the same replicated/imported secret pattern.

Current host-namespace implementation does not need vCluster
`sync.fromHost.secrets`.

## Gateway API

Verified:

- Preview `HTTPRoute` hostnames can be patched per PR.
- KGateway exposes preview apps through generated hostnames such as
  `web-pr-3.sonia-certs.uk`.

Open item:

- Gateway API policy resources still need a dedicated test.

## Dependencies

Verified in the lab:

- Redis operator dependency worked with host-side operator ownership.
- RabbitMQ operator dependency worked with host-side operator ownership during
  the vCluster dependency rehearsal.
- S3 Mountpoint CSI worked with host-side static PV ownership.
- S3 PVCs are currently part of the realistic web base/preview flow.

Current recommendation:

- Keep platform dependencies host-side and platform-owned.
- Tenant preview apps should consume generated Secrets, Services, PVCs, and
  normal namespaced resources.
- Do not let tenant preview apps create broad operator or cluster-scoped
  platform resources until RBAC, quota, cleanup, and CRD behavior are tested per
  dependency.

Open items:

- CNPG/database preview mode is not installed or tested.
- RabbitMQ in the current host-namespace flow should be re-tested after the
  vCluster path was retired.
- Redis/S3 are tested enough for this lab, but lifecycle and quota policies still
  need production design.

## Observability

Open items:

- Metrics scraping for preview workloads is not implemented yet.
- Likely pattern: controller-generated `VMServiceScrape` or `VMPodScrape` in
  the preview namespace with strict labels.
- Grafana dashboard resources are not implemented yet.
- Keep `GrafanaDashboard` host-side/platform-owned unless tenant dashboard CRDs
  are explicitly allowlisted.
- OpenTelemetry instrumentation preview mode is not implemented yet.

## RBAC And Tenancy

Verified:

- Preview apps use a dedicated Argo CD project.
- Generated preview resources are isolated by namespace.
- The controller is allowlisted to the lab repositories only.
- The controller writes GitOps state and does not require broad Kubernetes RBAC
  for preview workload creation.

Open items:

- Production quota and LimitRange policy for preview namespaces.
- NetworkPolicy defaults for preview namespaces.
- AppProject destination and resource allowlist hardening.
- Explicit deny/allow behavior for cluster-scoped resources in preview apps.
- User-facing status and failure reporting.

## vCluster Findings

vCluster was useful for understanding the isolation boundary, but the current
implementation intentionally moved back to plain host namespaces first.

Concrete findings from the vCluster experiment:

- Every workload Pod inside a vCluster becomes a real host-cluster Pod.
- vCluster added complexity around secrets, operators, Gateway API, metrics,
  CSI, and Argo CD visibility.
- OSS vCluster can work, but several convenient integrations are Pro-only.
- Host-side ownership of platform dependencies was still the cleanest boundary.
- Argo CD hit trouble caching some vCluster workload objects during the lab.

Current decision:

- Do not use vCluster for the first production-like iteration.
- Revisit vCluster only if namespace-level isolation is not enough.

## Remaining Todo List

- GitHub webhooks instead of polling.
- Controller status reporting back to GitHub PRs.
- Controller Prometheus metrics.
- User-facing events or comments explaining preview creation failures.
- Multi-app PR behavior, where one code PR creates multiple preview apps.
- Private GHCR package visibility test.
- Gateway API policy resources.
- Metrics scraping.
- Grafana dashboards.
- OpenTelemetry instrumentation.
- CNPG/database mode.
- Current host-namespace RabbitMQ rehearsal.
- Production RBAC, ResourceQuota, LimitRange, and NetworkPolicy defaults.
- Documentation/examples for `PreviewEnvironment` source index rules.
