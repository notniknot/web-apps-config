# vCluster Preview Lab Results

This document records the manual rehearsal of the future preview controller
workflow for `notniknot/web-apps` and `notniknot/web-apps-config`.

## Repository Contract

- Code repository: `notniknot/web-apps`.
- Config repository: `notniknot/web-apps-config`.
- PR image tag: `ghcr.io/notniknot/web-apps:preview-pr-<number>`.
- Preview metadata file: `preview.yaml`.
- Generated overlay pattern: `previews/pr-<number>`.
- Kustomize structure: stable app base plus small preview components.
- Argo CD preview app: `argocd/web-apps-preview-pr-1`.
- Preview vCluster: `web-pr-1` in host namespace `web-preview-pr-1`.
- Preview URL tested internally through KGateway: `https://web-pr-1.sonia-certs.uk/`.

## Use Case Results

- Repository bootstrap: working. `web-apps` and `web-apps-config` are initialized and pushed.
- Private GHCR image and pull secret: partial. GHCR package was created public by default; package API access works after adding package scopes, but changing visibility still needs a follow-up. A real GHCR pull secret is installed, replicated/imported, and referenced by the synced preview Pod.
- vCluster preview app through Argo CD: working. Argo CD deploys `web-apps-preview-pr-1` into the vCluster API, and vCluster syncs the real Pod and Service into `web-preview-pr-1`.
- Argo CD Image Updater digest write-back: working. It matched the preview app, resolved `preview-pr-1`, and pushed `digest: sha256:07dc...` into `previews/pr-1/kustomization.yaml`.
- Shared replicated secret: working at host level. `shared-preview-secret` was copied from `web` to `web-preview-pr-1` by mittwald replicator.
- Per-preview generated ExternalSecret: working with host-side generation. vCluster's built-in `integrations.externalSecrets` setting is a vCluster Pro feature, so OSS vCluster cannot use that integration directly. For OSS, the platform/controller should generate the `ExternalSecret` in the host preview namespace and let vCluster import the resulting Secret.
- Gateway API HTTPRoute through KGateway: working with host-side generation. Declaring `HTTPRoute` inside the vCluster caused Argo CD comparison trouble and did not produce a host-side route in this test. The simpler controller-owned pattern is to generate the host `HTTPRoute` in the preview namespace and point it at the vCluster-synced Service.
- Gateway API policy resources: pending.
- Metrics scraping from vCluster workloads: not installed yet. The synced Service and Pod have stable labels in the host namespace, so the likely pattern is controller-generated host-side `VMServiceScrape` or `VMPodScrape`.
- Grafana dashboard resources: not installed yet. Keep `GrafanaDashboard` host-side/platform-owned unless tenant dashboard CRDs are deliberately allowlisted.
- Redis operator dependency: not installed yet. Recommended first test is host-side generated Redis CR plus imported connection Secret, not arbitrary tenant CR sync.
- RabbitMQ operator dependency: not installed yet. Same pattern as Redis: host-side generated `RabbitmqCluster`, imported Secret.
- CNPG dependency: not installed yet. Treat as a separate database mode with strict cleanup and credentials policy.
- PersistentVolumeClaim and S3 Mountpoint CSI: not tested yet. Basic PVC sync is enabled by default; S3 CSI needs the host storage class/driver present and should be tested with a small read-only bucket first.
- OpenTelemetry instrumentation: not installed yet. Prefer host-side operator policy or explicit app instrumentation before syncing arbitrary `Instrumentation` CRs.
- Cluster-scoped/RBAC resources: not tested in this run. Preview AppProject should continue to block broad cluster-scoped resources for tenant apps.

## Verified Commands/Signals

- GitHub PR #1 with label `preview/playground` built `ghcr.io/notniknot/web-apps:preview-pr-1`.
- Image Updater wrote commit `7035026` to `web-apps-config`.
- Current preview overlay renders `ghcr.io/notniknot/web-apps:preview-pr-1@sha256:07dc148eef9f847e4fa8b7acf875c127cd343fb02eca1ebdf87f96d9681ac172`.
- Synced host Pod uses the pinned digest and is Ready.
- Host secrets present: `ghcr-pull-secret`, `shared-preview-secret`, `generated-preview-secret`.
- vCluster-imported translated secrets present: `*-x-web-x-web-pr-1`.
- Host `HTTPRoute web-preview-pr-1/web-pr-1` has `Accepted=True` and `ResolvedRefs=True`.
- Internal KGateway curl to `https://web-pr-1.sonia-certs.uk/` returned the app page.

## Current Recommended Pattern

- App overlay in `web-apps-config` should own only normal namespaced workload resources inside the vCluster.
- Preview controller/platform owns host namespace, vCluster, replicated/imported secrets, generated host `ExternalSecret`, generated host `HTTPRoute`, ImageUpdater CR, and Argo CD Application.
- Use OSS vCluster with `sync.fromHost.secrets` for pull/application secrets and normal workload sync for Pods/Services.
- Avoid vCluster Pro-only integrations in the OSS setup.
- Avoid Argo CD server-side diff/apply on preview apps that include CRDs inside vCluster unless tested per CRD.
