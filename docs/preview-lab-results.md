# vCluster Preview Lab Results

This document records the manual rehearsal of the future preview controller
workflow for `notniknot/web-apps` and `notniknot/web-apps-config`.

## Repository Contract

- Code repository: `notniknot/web-apps`.
- Config repository: `notniknot/web-apps-config`.
- PR image tag: `ghcr.io/notniknot/web-apps:preview-pr-<number>`.
- Preview metadata file: `preview.yaml`.
- Stable app pattern: `apps/<app>/base`, `apps/<app>/<env>`, and `apps/<app>/<app>-<env>.argocd.yaml`, matching the real tenant config repositories.
- Generated preview overlay pattern for this lab run: `previews/pr-<number>`.
- Future generated preview overlay pattern should either keep `previews/pr-<number>` or move under `apps/<app>/previews/pr-<number>`; the important contract is that the controller owns generated preview paths and developers own the stable app base/env overlays.
- Argo CD preview app: `argocd/web-apps-preview-pr-1`.
- Preview vCluster: `web-pr-1` in host namespace `web-preview-pr-1`.
- Preview URL tested internally through KGateway: `https://web-pr-1.sonia-certs.uk/`.

## Use Case Results

- Repository bootstrap: working. `web-apps` and `web-apps-config` are initialized and pushed.
- Private GHCR image and pull secret: partial. GHCR package was created public by default; package API access works after adding package scopes, but changing visibility still needs a follow-up. A real GHCR pull secret is installed, replicated/imported, and referenced by the synced preview Pod.
- vCluster preview app through Argo CD: working. Argo CD deploys `web-apps-preview-pr-1` into the vCluster API, and vCluster syncs the real Pod and Service into `web-preview-pr-1`.
- Tenant app-of-apps structure: working. The root repo creates `argocd/web-apps-app-of-apps`, which scans `web-apps-config/apps` and creates the tenant-owned `web/web` Application.
- Stable parent-cluster app: working. `web/web` deploys `apps/web/playground` into the parent-cluster `web` namespace and the Deployment is Ready.
- Argo CD Image Updater digest write-back: working. It matched the preview app, resolved `preview-pr-1`, and pushed `digest: sha256:07dc...` into `previews/pr-1/kustomization.yaml`.
- Shared replicated secret: working at host level. `shared-preview-secret` was copied from `web` to `web-preview-pr-1` by mittwald replicator.
- Per-preview generated ExternalSecret: working with host-side generation. vCluster's built-in `integrations.externalSecrets` setting is a vCluster Pro feature, so OSS vCluster cannot use that integration directly. For OSS, the platform/controller should generate the `ExternalSecret` in the host preview namespace and let vCluster import the resulting Secret.
- Gateway API HTTPRoute through KGateway: working with host-side generation. Declaring `HTTPRoute` inside the vCluster caused Argo CD comparison trouble and did not produce a host-side route in this test. The simpler controller-owned pattern is to generate the host `HTTPRoute` in the preview namespace and point it at the vCluster-synced Service.
- Gateway API policy resources: pending.
- Metrics scraping from vCluster workloads: not installed yet. The synced Service and Pod have stable labels in the host namespace, so the likely pattern is controller-generated host-side `VMServiceScrape` or `VMPodScrape`.
- Grafana dashboard resources: not installed yet. Keep `GrafanaDashboard` host-side/platform-owned unless tenant dashboard CRDs are deliberately allowlisted.
- Redis operator dependency: platform capability installed. The lab now has the OT Container Kit Redis operator and the Redis CRDs.
- RabbitMQ operator dependency: platform capability installed. The lab now has the RabbitMQ Cluster Operator and the `RabbitmqCluster` CRD.
- CNPG dependency: not installed yet. Treat as a separate database mode with strict cleanup and credentials policy.
- PersistentVolumeClaim and S3 Mountpoint CSI: driver installed. The lab now has the AWS Mountpoint S3 CSI driver, its `CSIDriver`, and `MountpointS3PodAttachment` CRD. The driver does not create a dynamic StorageClass; the real cluster pattern uses static PV/PVC resources with `driver: s3.csi.aws.com`.
- OpenTelemetry instrumentation: not installed yet. Prefer host-side operator policy or explicit app instrumentation before syncing arbitrary `Instrumentation` CRs.
- Cluster-scoped/RBAC resources: not tested in this run. Preview AppProject should continue to block broad cluster-scoped resources for tenant apps.

## App-Of-Apps Structure

- Infra/root repo owns tenant `AppProject` resources and the tenant app-of-apps `Application`.
- Tenant config repo owns child application manifests under `apps/<app>/<app>-<env>.argocd.yaml`.
- For this lab, `web-apps-app-of-apps` scans `notniknot/web-apps-config/apps` for `*-playground.argocd.yaml`.
- The stable lab app is `web/web`, deployed from `apps/web/playground` to the parent-cluster `web` namespace.
- The preview app stays separate as generated/controller-owned infrastructure: `argocd/web-apps-preview-pr-1` points to the vCluster API and deploys `previews/pr-1`.

## Dependency Patterns

- Redis: operate the Redis operator in the host cluster. For previews, the controller should create a tightly-scoped host-side Redis CR or choose a shared Redis mode, then import only the connection Secret into the vCluster. Do not let arbitrary tenant preview apps create Redis operator CRs inside vCluster until RBAC, cleanup, and quota behavior are tested.
- RabbitMQ: same boundary as Redis. Keep the RabbitMQ operator host-side, create per-preview `RabbitmqCluster` resources only through platform automation, and sync the generated connection Secret into the vCluster for the app.
- S3 Mountpoint CSI: operate the CSI driver in the host cluster. The vCluster app can use a PVC only if the host PV/PVC contract is generated and synced correctly. Bucket credentials, bucket naming, access mode, and cleanup should be platform-owned, usually with an imported Secret plus generated static PV/PVC resources.
- Shared dependency Secret flow: ESO or another host mechanism creates the source Secret in the tenant namespace, mittwald replicator copies it into `web-preview-pr-<n>`, and `sync.fromHost.secrets` imports it into the vCluster.
- Per-preview generated Secret flow: the preview controller creates an `ExternalSecret` or plain generated Secret in the host preview namespace, waits for the resulting Secret, and `sync.fromHost.secrets` imports it into the vCluster.

## Verified Commands/Signals

- GitHub PR #1 with label `preview/playground` built `ghcr.io/notniknot/web-apps:preview-pr-1`.
- Image Updater wrote commit `7035026` to `web-apps-config`.
- Tenant app-of-apps commit in `argocd`: `a5fb10f`.
- Real-style tenant config commit in `web-apps-config`: `1cdf33e`.
- Lab dependency operator install commit in `argocd`: `06ff35d`.
- Current preview overlay renders `ghcr.io/notniknot/web-apps:preview-pr-1@sha256:07dc148eef9f847e4fa8b7acf875c127cd343fb02eca1ebdf87f96d9681ac172`.
- `argocd/web-apps-app-of-apps` is `Synced` and `Healthy`.
- `web/web` is `Synced` and `Healthy`.
- Parent-cluster `web` Deployment is `1/1` Ready.
- `argocd/redis-operator`, `argocd/rabbitmq-operator`, and `argocd/mountpoint-s3` are `Synced` and `Healthy`.
- Redis CRDs verified: `redis.redis.redis.opstreelabs.in`, `redisclusters.redis.redis.opstreelabs.in`, `redisreplications.redis.redis.opstreelabs.in`, `redissentinels.redis.redis.opstreelabs.in`.
- RabbitMQ CRD verified: `rabbitmqclusters.rabbitmq.com`.
- Mountpoint S3 verified: `CSIDriver s3.csi.aws.com` and `mountpoints3podattachments.s3.csi.aws.com`.
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
