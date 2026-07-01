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

- Repository bootstrap: pending.
- Private GHCR image and pull secret: pending.
- vCluster preview app through Argo CD: pending.
- Argo CD Image Updater digest write-back: pending.
- Shared replicated secret: pending.
- Per-preview generated ExternalSecret: pending.
- Gateway API HTTPRoute through KGateway: pending.
- Gateway API policy resources: pending.
- Metrics scraping from vCluster workloads: pending.
- Grafana dashboard resources: pending.
- Redis operator dependency: pending.
- RabbitMQ operator dependency: pending.
- CNPG dependency: pending.
- PersistentVolumeClaim and S3 Mountpoint CSI: pending.
- OpenTelemetry instrumentation: pending.
- Cluster-scoped/RBAC resources: pending.

