# vCluster Preview Lab Results

This document records the manual rehearsal of the future preview controller
workflow for `notniknot/web-apps` and `notniknot/web-apps-config`.

## Repository Contract

- Code repository: `notniknot/web-apps`.
- Config repository: `notniknot/web-apps-config`.
- PR image tag: `ghcr.io/notniknot/web-apps:preview-pr-<number>`.
- Preview metadata file: `preview.yaml`.
- Stable app pattern: `apps/<app>/base`, `apps/<app>/<env>`, and `apps/<app>/<app>-<env>.argocd.yaml`, matching the real tenant config repositories.
- Generated preview overlay pattern: `apps/<app>/previews/<environment>/pr-<number>`.
- For this lab PR, the overlay lives at `apps/web/previews/playground/pr-1`; developers own the stable app base/env overlays and the controller owns generated preview paths.
- Argo CD preview app: `argocd/web-apps-preview-pr-1`.
- Preview vCluster: `web-pr-1` in host namespace `web-preview-pr-1`.
- Preview URL tested internally through KGateway: `https://web-pr-1.sonia-certs.uk/`.

## Use Case Results

- Repository bootstrap: working. `web-apps` and `web-apps-config` are initialized and pushed.
- Private GHCR image and pull secret: partial. GHCR package was created public by default; package API access works after adding package scopes, but changing visibility still needs a follow-up. A real GHCR pull secret is installed, replicated/imported, and referenced by the synced preview Pod.
- vCluster preview app through Argo CD: working. Argo CD deploys `web-apps-preview-pr-1` into the vCluster API, and vCluster syncs the real Pod and Service into `web-preview-pr-1`.
- Preview controller automation: working. `notniknot/preview-controller` polls the allowlisted GitHub PRs, detects `preview/playground`, reads `preview.yaml`, creates/updates the preview config branch, writes generated Argo/platform resources, preserves Image Updater digests, and is deployed in the lab cluster.
- Tenant app-of-apps structure: working. The root repo creates `argocd/web-apps-app-of-apps`, which scans `web-apps-config/apps` and creates the tenant-owned `web/web` Application.
- Stable parent-cluster app: working. `web/web` deploys `apps/web/playground` into the parent-cluster `web` namespace and the Deployment is Ready.
- Argo CD Image Updater digest write-back: working. It matched the preview app, resolved `preview-pr-1`, and pushed `digest: sha256:120...` into `apps/web/previews/playground/pr-1/kustomization.yaml`.
- Shared replicated secret: working at host level. `shared-preview-secret` was copied from `web` to `web-preview-pr-1` by mittwald replicator.
- Per-preview generated ExternalSecret: working with host-side generation. vCluster's built-in `integrations.externalSecrets` setting is a vCluster Pro feature, so OSS vCluster cannot use that integration directly. For OSS, the platform/controller should generate the `ExternalSecret` in the host preview namespace and let vCluster import the resulting Secret.
- Gateway API HTTPRoute through KGateway: working with host-side generation. Declaring `HTTPRoute` inside the vCluster caused Argo CD comparison trouble and did not produce a host-side route in this test. The simpler controller-owned pattern is to generate the host `HTTPRoute` in the preview namespace and point it at the vCluster-synced Service.
- Gateway API policy resources: pending.
- Metrics scraping from vCluster workloads: not installed yet. The synced Service and Pod have stable labels in the host namespace, so the likely pattern is controller-generated host-side `VMServiceScrape` or `VMPodScrape`.
- Grafana dashboard resources: not installed yet. Keep `GrafanaDashboard` host-side/platform-owned unless tenant dashboard CRDs are deliberately allowlisted.
- Redis operator dependency: working with host-side operator ownership. The lab creates a host-side `Redis` CR in `web-preview-pr-1`, imports the generated/password Secret into the vCluster, replicates the host Service into the vCluster, and a vCluster Job successfully ran `redis-cli ping`.
- RabbitMQ operator dependency: working with host-side operator ownership. The lab creates a host-side `RabbitmqCluster`, imports the generated default-user Secret into the vCluster, replicates the host Service into the vCluster, and a vCluster Job successfully reached port `5672`.
- CNPG dependency: not installed yet. Treat as a separate database mode with strict cleanup and credentials policy.
- PersistentVolumeClaim and S3 Mountpoint CSI: working with host-side static PV ownership. The lab uses AWS Mountpoint S3 CSI with a static host `PersistentVolume`, a vCluster PVC, and the vCluster verifier Job successfully listed a public S3 bucket mounted at `/mnt/s3`.
- OpenTelemetry instrumentation: not installed yet. Prefer host-side operator policy or explicit app instrumentation before syncing arbitrary `Instrumentation` CRs.
- Cluster-scoped/RBAC resources: not tested in this run. Preview AppProject should continue to block broad cluster-scoped resources for tenant apps.

## App-Of-Apps Structure

- Infra/root repo owns tenant `AppProject` resources and the tenant app-of-apps `Application`.
- Tenant config repo owns child application manifests under `apps/<app>/<app>-<env>.argocd.yaml`.
- For this lab, `web-apps-app-of-apps` scans `notniknot/web-apps-config/apps` for `*-playground.argocd.yaml`.
- The stable lab app is `web/web`, deployed from `apps/web/playground` to the parent-cluster `web` namespace.
- The preview app stays separate as generated/controller-owned infrastructure: `argocd/web-apps-preview-pr-1` points to the vCluster API and deploys `apps/web/previews/playground/pr-1`.

## Dependency Patterns

- Redis: operate the Redis operator in the host cluster. For previews, the controller should create a tightly-scoped host-side Redis CR or choose a shared Redis mode, then import only the connection Secret into the vCluster. Do not let arbitrary tenant preview apps create Redis operator CRs inside vCluster until RBAC, cleanup, and quota behavior are tested.
- RabbitMQ: same boundary as Redis. Keep the RabbitMQ operator host-side, create per-preview `RabbitmqCluster` resources only through platform automation, and sync the generated connection Secret into the vCluster for the app.
- S3 Mountpoint CSI: operate the CSI driver in the host cluster. The vCluster app can use a PVC only if the host PV/PVC contract is generated and synced correctly. Bucket credentials, bucket naming, access mode, and cleanup should be platform-owned, usually with an imported Secret plus generated static PV/PVC resources.
- Shared dependency Secret flow: ESO or another host mechanism creates the source Secret in the tenant namespace, mittwald replicator copies it into `web-preview-pr-<n>`, and `sync.fromHost.secrets` imports it into the vCluster.
- Per-preview generated Secret flow: the preview controller creates an `ExternalSecret` or plain generated Secret in the host preview namespace, waits for the resulting Secret, and `sync.fromHost.secrets` imports it into the vCluster.
- Per-preview Redis/RabbitMQ flow: the preview controller creates host-side operator CRs in the host preview namespace, waits for Ready conditions and generated connection Secrets, configures vCluster `sync.fromHost.secrets`, and configures vCluster `networking.replicateServices.fromHost` so the app connects to normal in-vCluster Service names.
- Per-preview S3 CSI flow: the preview controller creates or selects host-side S3 credentials, creates the static host `PersistentVolume` with the S3 CSI driver, and lets the tenant preview overlay create the matching PVC inside the vCluster. Keep bucket naming, credentials, lifecycle, and cleanup platform-owned.

## Verified Commands/Signals

- GitHub PR #1 with label `preview/playground` built `ghcr.io/notniknot/web-apps:preview-pr-1`.
- Image Updater wrote commit `7035026` to `web-apps-config`.
- Tenant app-of-apps commit in `argocd`: `a5fb10f`.
- Real-style tenant config commit in `web-apps-config`: `1cdf33e`.
- Lab dependency operator install commit in `argocd`: `06ff35d`.
- Lab dependency platform resource commit in `argocd`: `976b9f4`.
- vCluster dependency verifier commit in `web-apps-config`: `d424bdd`.
- Preview controller image: `ghcr.io/notniknot/preview-controller:latest`, currently also tagged `d23baab`.
- Controller-generated Argo CD revision for preview-platform: `f302137`.
- Controller-generated preview config revision: `99df5f6`.
- Current preview overlay renders `ghcr.io/notniknot/web-apps:preview-pr-1@sha256:120bbbb5702eb2612c6693265abd3334f55fe9aaf6bf7ad7cbfe2eb70781823a`.
- `argocd/web-apps-app-of-apps` is `Synced` and `Healthy`.
- `argocd/preview-platform`, `argocd/web-preview-pr-1-vcluster`, and `argocd/web-apps-preview-pr-1` are `Synced` and `Healthy` after the dependency test.
- `web/web` is `Synced` and `Healthy`.
- Parent-cluster `web` Deployment is `1/1` Ready.
- `argocd/redis-operator`, `argocd/rabbitmq-operator`, and `argocd/mountpoint-s3` are `Synced` and `Healthy`.
- Redis CRDs verified: `redis.redis.redis.opstreelabs.in`, `redisclusters.redis.redis.opstreelabs.in`, `redisreplications.redis.redis.opstreelabs.in`, `redissentinels.redis.redis.opstreelabs.in`.
- RabbitMQ CRD verified: `rabbitmqclusters.rabbitmq.com`.
- Mountpoint S3 verified: `CSIDriver s3.csi.aws.com` and `mountpoints3podattachments.s3.csi.aws.com`.
- Preview dependency test verified from inside the vCluster: Redis ping, RabbitMQ TCP connect, imported Redis/RabbitMQ Secrets, and S3 PVC mount/listing all succeeded.
- Synced host Pod uses the pinned digest and is Ready.
- Host secrets present: `ghcr-pull-secret`, `shared-preview-secret`, `generated-preview-secret`.
- vCluster-imported translated secrets present: `*-x-web-x-web-pr-1`.
- Host `HTTPRoute web-preview-pr-1/web-pr-1` has `Accepted=True` and `ResolvedRefs=True`.
- Internal KGateway curl to `https://web-pr-1.sonia-certs.uk/` returned the app page.
- In-cluster smoke curl to `https://web-pr-1.sonia-certs.uk/` returned the app page with PR `1`, the replicated secret value, and the generated ESO secret value.
- The running controller reconciled multiple times without creating new commits after the generated state stabilized.

## Current Recommended Pattern

- App overlay in `web-apps-config` should own only normal namespaced workload resources inside the vCluster.
- Preview controller/platform owns host namespace, vCluster, replicated/imported secrets, generated host `ExternalSecret`, generated host `HTTPRoute`, ImageUpdater CR, and Argo CD Application.
- Use OSS vCluster with `sync.fromHost.secrets` for pull/application secrets and normal workload sync for Pods/Services.
- Avoid vCluster Pro-only integrations in the OSS setup.
- Avoid Argo CD server-side diff/apply on preview apps that include CRDs inside vCluster unless tested per CRD.

## vCluster Lab Findings

- OSS vCluster is enough for this pattern. We did not need vCluster Pro integrations for Redis, RabbitMQ, or S3; the boundary was host-side platform resources plus explicit sync/replication into the virtual cluster.
- Host-to-vCluster Secret import worked for replicated pull/shared Secrets and generated dependency Secrets.
- Host-to-vCluster Service replication worked for Redis and RabbitMQ, allowing the preview app to use normal in-vCluster Service names.
- Static S3 CSI PV/PVC worked through vCluster when the platform owned the host PV and the app owned only the vCluster PVC.
- Argo CD can manage the preview app against the vCluster API, but this lab exposed a controller panic in Argo CD while caching vCluster Pods: `panic: assignment to entry in nil map` in `kubectl/pkg/util/resource.maxResourceList`. The practical mitigation for now is to require complete CPU and memory requests/limits on all preview Pods and hooks, and to avoid leaving stale Argo hook finalizers during crash recovery.
- Keep operator CRs host-side for now. Letting tenant apps create Redis/RabbitMQ/S3 platform resources inside the vCluster would require extra RBAC, quota, cleanup, and CRD behavior testing.

## Preview Controller Findings

- The first production boundary is GitHub, not Kubernetes. The controller only needs a GitHub token and does not need cluster-wide Kubernetes RBAC because it writes GitOps state and lets Argo CD apply it.
- The repository binding must stay allowlisted. This lab controller is intentionally limited to `notniknot/web-apps`, `notniknot/web-apps-config`, and `notniknot/argocd`.
- The controller must preserve fields written by other automation. In this lab, preserving the Image Updater `digest` in the preview branch was required to avoid controller/Image Updater commit churn.
- Generated preview files should be aggregated under a controller-owned path such as `kubernetes/apps/preview-platform/lab/generated/web-pr-1-platform.yaml`; shared platform resources stay outside the generated file.
- Long-running polling works once writes are byte-stable. The deployed controller reconciles PR #1 every minute and did not create additional commits after the desired state matched.
