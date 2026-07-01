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

## Use Case Results

- Repository bootstrap: working. `web-apps` and `web-apps-config` are initialized and pushed.
- Private GHCR image and pull secret: partial. GHCR package was created public by default; package API access works after adding package scopes, but changing visibility still needs a follow-up. A real GHCR pull secret is installed for the lab.
- vCluster preview app through Argo CD: in progress.
- Argo CD Image Updater digest write-back: in progress. The v1 `ImageUpdater` CRD requires `commonUpdateSettings.platforms` as a list, not a scalar string.
- Shared replicated secret: working at host level. `shared-preview-secret` was copied from `web` to `web-preview-pr-1` by mittwald replicator.
- Per-preview generated ExternalSecret: adjusted. vCluster's built-in `integrations.externalSecrets` setting is a vCluster Pro feature, so OSS vCluster cannot use that integration directly. For OSS, the platform/controller should generate the `ExternalSecret` in the host preview namespace and let vCluster import the resulting Secret.
- Gateway API HTTPRoute through KGateway: adjusted to host-side generation. Declaring `HTTPRoute` inside the vCluster caused Argo CD comparison trouble and did not produce a host-side route in this test. The simpler controller-owned pattern is to generate the host `HTTPRoute` in the preview namespace and point it at the vCluster-synced Service.
- Gateway API policy resources: pending.
- Metrics scraping from vCluster workloads: pending.
- Grafana dashboard resources: pending.
- Redis operator dependency: pending.
- RabbitMQ operator dependency: pending.
- CNPG dependency: pending.
- PersistentVolumeClaim and S3 Mountpoint CSI: pending.
- OpenTelemetry instrumentation: pending.
- Cluster-scoped/RBAC resources: pending.
