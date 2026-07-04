# Preview Environment Lab Results

Last updated: 2026-07-04.

This document records the lab result for self-service preview environments using
`notniknot/web-apps`, `notniknot/web-apps-config`, `notniknot/argocd`, and
`notniknot/preview-controller`.

## Current Direction

- Use normal host-cluster namespaces for preview environments. Do not use
  vCluster in the current implementation path.
- Drive previews entirely from Kubernetes CRs. The old GitHub PR
  label/comment/PR-body-link trigger has been removed; it is not kept as a
  fallback.
- Developers request previews from inside Argo CD:
  - an Argo CD **UI extension** collects the request,
  - a thin Argo CD **proxy-extension backend** creates a
    `PreviewEnvironmentRequest` CR,
  - the **preview controller** reconciles the request into a resolved
    `PreviewEnvironment` CR and generated GitOps state,
  - Argo CD syncs the generated preview `Application` and platform resources,
  - status conditions on both CRs feed the UI.
- GitHub is still used by the controller, but only to resolve requested PRs to
  immutable SHAs, inspect PR metadata, and read/write GitOps files. GitHub is no
  longer an event source.
- Keep platform-owned resources in the host cluster and generated through GitOps.
- Keep tenant-owned preview configuration inside the tenant config repository.

```text
Argo CD UI extension
  -> thin Go proxy-extension backend
    -> PreviewEnvironmentRequest CR (user intent, namespaced)
      -> preview-controller (controller-runtime operator)
        -> PreviewEnvironment CR (resolved, controller-owned)
        -> generated Argo CD Application + platform resources in notniknot/argocd
        -> status, URLs, cleanup
```

## Repository Contract

- Code repository: `notniknot/web-apps`.
- Tenant config repository: `notniknot/web-apps-config`.
- Root/Argo CD config repository: `notniknot/argocd`.
- Controller repository: `notniknot/preview-controller`.
- App-local preview template: `apps/web/preview.yaml` (`kind: PreviewTemplate`).
- App-local preview components: `apps/web/previews/components`.
- App-local preview patches: `apps/web/previews/patches`.
- Stable app structure: `apps/<app>/base`, `apps/<app>/<env>`, and
  `apps/<app>/<app>-<env>.argocd.yaml`.
- Preview identity (`{{.PreviewID}}`), stable and collision-safe:
  - code PR preview: `pr-123`
  - config-only preview: `config-pr-42`
  - code+config preview: `pr-123-config-pr-42`
  - multi-code+config: `pr-123-pr-124-config-pr-42` (code PRs sorted)
  - an optional request `variant` is appended: `pr-123-review2`
- Preview namespace pattern: `<tenant>-preview-<previewID>`.
- Preview Argo CD application pattern: `<app>-apps-preview-<previewID>`.
- Preview hostname pattern: `web-<previewID>.sonia-certs.uk`.

## CRD Model

Three kinds, all group `preview.sonia.so/v1alpha1`.

### PreviewTemplate (tenant-owned, Git document)

`apps/web/preview.yaml` is the app-local template. It is read from the config
repository by the controller; it is not created in the cluster during normal
workflows.

```yaml
apiVersion: preview.sonia.so/v1alpha1
kind: PreviewTemplate
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
          message: preview-{{.PreviewID}}
  values:
    hostname: web-{{.PreviewID}}.sonia-certs.uk
    imageTagPrefix: preview-{{.PreviewID}}-
  secrets:
    import:
      - ghcr-pull-secret
    generated:
      - generated-preview-secret
```

Notes:

- The template no longer carries a `pullRequests` block. PR linkage is now
  explicit request intent, not PR-body parsing.
- `application.path` points to the existing Argo CD `Application` YAML in the
  tenant config repository.
- `sources[*].index` maps to `spec.sources[index]` of that Argo CD Application.
- Template values are rendered with `{{.PreviewID}}`, `{{.PR}}` (owner PR),
  `{{.CodePRs}}`, `{{.ConfigPR}}`, and `{{.Values.*}}`.

### PreviewEnvironmentRequest (user intent)

Namespace-scoped, RBAC-controlled, created by the proxy backend.

```yaml
apiVersion: preview.sonia.so/v1alpha1
kind: PreviewEnvironmentRequest
metadata:
  name: web-pr-123
  namespace: web
spec:
  application:
    configRepository: notniknot/web-apps-config
    path: apps/web
    cluster: playground
  sources:
    config:
      pullRequest: 4
    code:
      - repository: notniknot/web-apps
        pullRequest: 123
      - repository: notniknot/worker
        pullRequest: 77
  ttl: 8h
  reason: review checkout flow
  variant: ""            # optional: a second preview for the same PR combination
```

The preview shape is derived from which sources are set:

- config only: `sources.config`
- code only: one `sources.code[]`
- code + config: one `sources.code[]` and `sources.config`
- multiple code + config: several `sources.code[]` and one `sources.config`

### PreviewEnvironment (resolved, controller-owned)

```yaml
apiVersion: preview.sonia.so/v1alpha1
kind: PreviewEnvironment
metadata:
  name: pr-123-config-pr-4
  namespace: web
spec:
  previewID: pr-123-config-pr-4
  tenant: web
  app: web
  environment: playground
  cluster: playground
  project: web-preview
  configRepository: notniknot/web-apps-config
  argoRepository: notniknot/argocd
  requestRef: web-pr-123
  resolvedSources:
    config: { repository: notniknot/web-apps-config, pullRequest: 4, sha: <sha> }
    code:
      - { repository: notniknot/web-apps, pullRequest: 123, sha: <sha> }
status:
  phase: Ready
  namespace: web-preview-pr-123-config-pr-4
  applicationName: web-apps-preview-pr-123-config-pr-4
  hostnames: [ web-pr-123-config-pr-4.sonia-certs.uk ]
  urls: [ https://web-pr-123-config-pr-4.sonia-certs.uk ]
  generatedFiles: [ ... ]
  conditions:
    - type: Ready
      status: "True"
      reason: Rendered
```

Users normally create `PreviewEnvironmentRequest`, not `PreviewEnvironment`
directly. Direct creation remains an admin/debug path.

## Controller Behavior

For each `PreviewEnvironmentRequest` the controller:

1. Resolves `spec.application` to an allowlisted application binding.
2. Authorizes the request namespace and every requested code repository against
   the allowlist. Config source repository must match the binding.
3. Resolves each config/code PR to an immutable head SHA. A missing or
   non-open PR is denied with an actionable status.
4. Reads and validates the `PreviewTemplate` at the config PR SHA (or the base
   branch for code-only/no-config requests). A missing/invalid template is
   denied with status.
5. Computes the stable `previewID` (with optional `variant`).
6. Creates/updates the resolved `PreviewEnvironment`, guarding the shared
   identity: if the preview ID is already owned by another request, the second
   request is denied and told to set a distinct `variant`.
7. Renders the generated Argo CD `Application` and host-side platform resources
   (namespace and generated `ExternalSecret`s) into `notniknot/argocd`, and adds
   the generated platform file to the preview-platform kustomization.
8. Writes status conditions on both CRs.

For deletion/cancel (finalizer-driven, idempotent) the controller removes the
generated tenant Application file, the generated platform file, the platform
kustomization entry, and the owned `PreviewEnvironment`. Cleanup only touches
files owned by that request's preview identity.

The controller reconciles on CR change and requeues on an interval so it picks
up new commits on a still-open PR. Status is machine-readable (condition
`Ready` with reasons such as `InvalidRepository`, `PullRequestNotFound`,
`TemplateNotFound`, `Unauthorized`, `PreviewIdentityConflict`, `Rendered`) so the
UI can show actionable errors.

## Request Workflows

### Config-only preview

Request sets only `sources.config`. Preview identity `config-pr-<n>`. Tenant
config sources are rewritten to the config PR head SHA.

### Code-only preview

Request sets one `sources.code[]`. Preview identity `pr-<n>`. Tenant config is
read from the base branch; the code PR selects the preview image via the
`imageTagPrefix` value and Argo CD Image Updater.

### Code + config preview

Request sets one `sources.code[]` and `sources.config`. Preview identity
`pr-<code>-config-pr-<config>`. Separate namespace, hostname, and app name from
either the code-only or config-only preview.

### Multiple code repos + one config preview

Request sets several `sources.code[]` and one `sources.config`. This is a single
combined preview `pr-<a>-pr-<b>-config-pr-<c>` (code PRs sorted), with all code
PR SHAs recorded in the resolved environment.

### Multiple previews for the same PR combination

Two requests may reference the same PRs as long as they use different `variant`
values (or names that resolve to different preview IDs). Each owns its own
environment and cleans up independently.

## Self-Service UX

### Argo CD UI extension

A vanilla-JS Argo CD UI extension (`ui-extension/`, following the
`argocd-drift-detector-extension` pattern) registers a system-level "Previews"
view. It lets the user select a preview-capable application, choose the preview
shape, enter PR numbers/repos, set optional TTL/reason/variant, submit the
request, and list existing previews with status, URLs, Argo CD app name, resolved
refs/SHAs, and errors, plus a delete/close action. The UI is client-side only.

### Proxy-extension backend

A thin Go proxy-extension backend (`cmd/preview-backend`) receives the browser
requests forwarded by `argocd-server`. It:

- lists preview-capable applications/templates for the user's app context,
- creates `PreviewEnvironmentRequest` CRs,
- lists requests/environments visible to the user,
- deletes/cancels requests.

It does not render manifests, mutate tenant config repositories, create Argo CD
Applications directly, or duplicate controller reconciliation logic.

### RBAC boundary

- Endpoints are app-scoped. The backend validates the Argo CD identity headers
  (`Argocd-Application-Name`, `Argocd-Project-Name`) that `argocd-server`
  forwards after authenticating the user and enforcing the `extensions` RBAC.
- The backend acts with its own least-privilege ServiceAccount, granted only
  create/list/delete of the request CRs in the tenant request namespace. It is
  not a privilege-escalation path.
- The controller holds the elevated GitHub/GitOps credentials, but every request
  still passes explicit allowlist, namespace, and ownership checks.

## Namespace Cleanup

Namespaces created only by Argo CD `CreateNamespace=true` are not pruned when the
preview app is removed. The controller therefore renders the preview namespace as
a **generated platform resource** in the preview-platform kustomization, which is
synced by a pruning `preview-platform` Application. Removing the generated
platform file on cleanup prunes the namespace.

## Multi-Source Argo CD Applications

The lab app uses `spec.sources`. The controller copies the source list from the
referenced Argo CD Application and mutates sources by index:

- `sourcePolicy: include-all` keeps all sources unless a rule says `action: drop`.
- `sourcePolicy: include-listed` keeps only explicitly listed source indexes.
- Kustomize mutations apply only to selected Kustomize sources; Helm mutations
  only to selected Helm sources.
- The generated preview Application emits a compatibility `spec.source` copied
  from the first source, because the root app uses server-side dry-run and
  rejected child Applications when `spec.source.repoURL` was absent.

## Image Updates

- The code repository builds PR images in GHCR with tags like
  `preview-pr-<n>-<sha>`.
- Argo CD Image Updater runs in `argocd` write-back mode for preview apps.
- Preview patches switch Image Updater away from Git write-back so it updates the
  generated Argo CD Application instead of mutating tenant Git, and constrain
  `allowTags` to the request's `imageTagPrefix`.
- Root ignore rules are required for preview Applications labeled
  `preview.sonia.so/enabled=true` so Image Updater-owned fields do not create
  drift.

Open item:

- Make private GHCR package visibility and pull-secret behavior fully
  production-realistic.

## Secrets

Verified patterns:

- Shared secret path: ESO creates a Secret in the stable tenant namespace, and
  mittwald/kubernetes-replicator copies it into preview namespaces.
- Generated secret path: the controller/platform generates an `ExternalSecret`
  directly in the preview namespace.
- Pull secrets can follow the same replicated/imported secret pattern.

## Gateway API

Verified:

- Preview `HTTPRoute` hostnames can be patched per preview.
- KGateway exposes preview apps through generated hostnames such as
  `web-pr-3.sonia-certs.uk`.

Open item:

- Gateway API policy resources still need a dedicated test.

## Dependencies

Verified in the lab: Redis operator, RabbitMQ operator (during the vCluster
rehearsal), and S3 Mountpoint CSI all worked with host-side platform ownership.

Current recommendation:

- Keep platform dependencies host-side and platform-owned.
- Tenant preview apps consume generated Secrets, Services, PVCs, and normal
  namespaced resources.
- Do not let tenant preview apps create broad operator or cluster-scoped
  platform resources until RBAC, quota, cleanup, and CRD behavior are tested per
  dependency.

Open items:

- CNPG/database preview mode is not installed or tested.
- RabbitMQ should be re-tested in the current host-namespace flow.
- Redis/S3 lifecycle and quota policies still need production design.

## RBAC And Tenancy

Verified:

- Preview apps use a dedicated Argo CD project.
- Generated preview resources are isolated by namespace.
- The controller is allowlisted to the lab repositories only and writes GitOps
  state rather than requiring broad Kubernetes RBAC for workload creation.
- The proxy backend honors Argo CD identity and is namespace-scoped by RBAC.

Open items:

- Production quota and LimitRange policy for preview namespaces.
- NetworkPolicy defaults for preview namespaces.
- AppProject destination and resource allowlist hardening.

## vCluster Findings

vCluster clarified the isolation boundary, but the current implementation moved
back to plain host namespaces. Host-side ownership of platform dependencies was
the cleanest boundary, several convenient vCluster integrations are Pro-only, and
Argo CD had trouble caching some vCluster workload objects. Revisit vCluster only
if namespace-level isolation proves insufficient.

## Verified Use Cases

- CRDs installed: `PreviewTemplate`, `PreviewEnvironmentRequest`,
  `PreviewEnvironment`.
- Controller unit tests (controller-runtime fake client + fake GitHub) cover:
  - config-only request
  - code-only request
  - code + config request
  - multiple code repos + one config repo request (single combined preview)
  - duplicate PR combination with distinct request names/variants
  - duplicate PR combination without a variant is denied with a conflict status
  - request deletion cleanup (files, platform kustomization entry, environment)
  - namespace cleanup via the generated platform resource
  - invalid code repository denied
  - missing PR denied with an actionable status
  - missing preview template denied with an actionable status
  - unauthorized request namespace denied
  - idempotent reconcile and idempotent cleanup
  - source-policy include-listed / drop and value-ref append rendering
- Backend tests cover header-based authorization (403 on missing/unknown
  project), disallowed-repository rejection, and create/list/delete round-trips.
- UI extension registration and helpers pass the bundle smoke test.

## Remaining Todo List

- Real-cluster smoke test of the full request -> environment -> GitOps flow for
  each preview shape after deployment.
- GitHub webhooks/event push (currently CR reconcile + requeue interval).
- Controller status write-back to GitHub PRs.
- Controller Prometheus metrics.
- User-facing events explaining preview creation failures.
- Multi-app requests, where one request creates multiple preview apps.
- Private GHCR package visibility test.
- Gateway API policy resources.
- Metrics scraping, Grafana dashboards, OpenTelemetry instrumentation.
- CNPG/database mode; host-namespace RabbitMQ rehearsal.
- Production RBAC, ResourceQuota, LimitRange, and NetworkPolicy defaults.
- TTL-based automatic preview expiry (spec field is recorded; enforcement TBD).
- Documentation/examples for `PreviewTemplate` source index rules.
