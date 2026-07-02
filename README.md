# web-apps-config

Tenant GitOps configuration for `notniknot/web-apps`.

Layout:

- `apps/web/base`: normal reusable app manifests.
- `apps/web/previews/components/*`: opt-in preview features and experiments.
- `previews/pr-<number>`: generated preview overlay owned by preview automation.
- `docs/preview-lab-results.md`: running notes from the vCluster lab.

The intended controller contract is:

```text
PR label -> create preview namespace/vCluster -> create preview overlay -> create Argo CD Application -> Image Updater pins image digest
```

