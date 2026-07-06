# Preview overlays

A preview environment is a normal kustomize build owned by the app team:
**the rendered env overlay + the shared preview component**, instantiated N
times by the preview controller.

```
base/                     shared across all clusters (incl. image-updater.yaml)
  └─ playground/          env overlay: all its make-it-work patches
       └─ previews/playground/   = ../../playground + previews/component
  └─ staging/             (dummy here) env overlay
       └─ previews/staging/      = ../../staging   + previews/component
previews/component/       minimal shared preview deltas:
                            kustomization.yaml  (static patches, replicas)
                            preview-params.yaml (the platform contract)
previews/helm/            helm addon + per-env value files (extra source)
```

The **replacements live in the per-env roots** (`previews/<env>/
kustomization.yaml`), not in the component. This is load-bearing: component
transformers run when the component is accumulated, *before* the root's own
patches — and the platform's preview-params patch is folded into the root at
render time. Within one kustomization, ordering is patches → replacements, so
only root-level replacements see the injected values (verified by simulating
the injection with `kustomize edit add patch` + build).

Previews layer on the **env overlay**, not raw base, so they inherit every
env-specific fix (reduced compute, hostnames, tracked tags, …) instead of
re-inventing them. One `PreviewTemplate` per preview-capable env
(`apps/web/preview-playground.yaml`, `apps/web/preview-staging.yaml`) points
at the matching overlay. Apps that need preview-only resources (seed jobs,
generated secrets, …) add them to the component or a per-env preview overlay.

## Contract with the platform

For each preview `<id>`, the controller generates an Argo CD `Application`
(name, project, destination, sync policy are platform-owned) with:

| Injection | Mechanism | Structure-coupled? |
|---|---|---|
| source → `apps/web/previews/<env>` @ config ref | generated Application source | no |
| target namespace `web-preview-<id>` | `spec.destination` + `kustomize.namespace` | no |
| preview labels | `kustomize.commonLabels` | no |
| per-instance values (hostname, pr, allowTags, pvName, …) | one kustomize patch on the **`preview-params` ConfigMap `data`** | no |
| per-preview helm values (addon source) | `helm.valuesObject` from the template | no |

Everything the platform touches is value-shaped. The **replacements in
`component/kustomization.yaml`** (owned here, versioned with the branch) fan
the params out to the app's actual resources. If a feature branch renames or
restructures resources, it updates base/env overlays and these replacements
**in the same commit** — the platform has nothing left to invalidate.

## How params are injected — Git is never written

The controller does not commit, push, or edit files anywhere. It only creates
the Application CR in-cluster, roughly:

```yaml
spec:
  source:
    repoURL: git@github.com:notniknot/web-apps-config.git
    targetRevision: <main | PR head SHA>
    path: apps/web/previews/playground
    kustomize:
      namespace: web-preview-pr42
      commonLabels: {preview.sonia.so/id: pr42, ...}
      patches:
        - target: {kind: ConfigMap, name: preview-params}
          patch: |-
            - op: replace
              path: /data
              value: {previewId: pr42, pr: "17", hostname: web-pr42.sonia-certs.uk,
                      allowTags: "regexp:^pr42-.*", ...}
```

At sync time the Argo CD repo-server checks out the ref into a scratch dir,
folds these options into that checkout's kustomization (`kustomize edit …`),
and builds. Kustomize applies patches **before** replacements, so the injected
`preview-params` data — not the placeholders committed here — is what the
replacements fan out. The checkout is discarded after render; deleting a
preview deletes the Application and leaves no trace in Git. The placeholder
values in `component/preview-params.yaml` exist only so local
`kustomize build` works.

## Local check

```sh
kustomize build apps/web/previews/playground
kustomize build apps/web/previews/staging
```
