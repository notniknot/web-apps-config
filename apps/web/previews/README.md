# Preview overlays

A preview environment is a normal kustomize build owned by the app team:
**the rendered env overlay + the shared preview component**, instantiated N
times by the preview controller.

```
base/                     shared across all clusters (incl. image-updater.yaml)
  └─ playground/          env overlay: make-it-work patches + preview.yaml
       └─ previews/playground/   = ../../playground + previews/component
  └─ staging/             (illustrative) env overlay + preview.yaml
       └─ previews/staging/      = ../../staging   + previews/component
previews/component/       minimal shared preview deltas:
                            kustomization.yaml  (static patches, replicas)
                            preview-params.yaml (the platform contract)
previews/helm/            helm addon + per-env value files (extra source)
```

Previews layer on the **env overlay**, not raw base, so they inherit every
env-specific fix (reduced compute, hostnames, tracked tags, …) instead of
re-inventing them.

**Templates travel with their env**: each env overlay carries its own
`preview.yaml` (`PreviewTemplate`, named `web` like the app), so every cluster
receives exactly its env's template through plain GitOps. Since previews wrap
the env overlay, the template would render into the preview too — the
**controller strips it** by appending a `$patch: delete` for it (it knows the
template's name from parsing it) to the injected patch set. In a local
`kustomize build` of a preview overlay the template is therefore still
visible; that is expected.

The **replacements live in the per-env roots** (`previews/<env>/
kustomization.yaml`), not in the component. This is load-bearing: component
transformers run when the component is accumulated, *before* the root's own
patches — and the platform's preview-params patch is folded into the root at
render time. Within one kustomization, ordering is patches → replacements, so
only root-level replacements see the injected values (verified by simulating
the injection with `kustomize edit add patch` + build).

## Contract with the platform

For each preview `<id>`, the controller generates an Argo CD `Application`
whose envelope (name, project, destination, sync policy) is platform-owned and
whose **sources are inherited** from the env's Application manifest
(`application.path` in the template, read at the config ref — so a PR that
edits sources, e.g. a chart version bump, is reflected in the preview). The
controller then applies value-shaped rewrites only:

| Rewrite | Mechanism | Structure-coupled? |
|---|---|---|
| env overlay source → `apps/web/previews/<env>` | **path-matched** swap (never index-matched; no/ambiguous match = deny) | no |
| pin config-repo sources to the PR head SHA | `targetRevision` rewrite by repoURL match | no |
| target namespace `web-preview-<id>` | `spec.destination` + `kustomize.namespace` | no |
| preview labels | `kustomize.commonLabels` | no |
| per-instance values (hostname, configPR, imageAllowTags, …) | one kustomize patch on the **`preview-params` ConfigMap `data`** | no |
| which values exist + which are user-overridable | template `values` (per-key `default`/`expose`/`description`) | no |
| how values become params keys | template `preview.params` mapping (value-shaped Go templates) | no |
| drop the env's `PreviewTemplate` from the render | `$patch: delete` appended by the controller | no (platform-owned kind) |
| per-source preview adjustments (e.g. preview helm values) | template `preview.sourceOverrides`, selector-matched by path/chart | no |

Sources not present in the env's Application don't exist in the preview;
sources it declares pass through (bounded by the AppProject source allowlist).

Everything the platform touches is value-shaped or targets platform-owned
kinds. The `preview-params.data` content is not controller-hardcoded: the
template's `preview.params` mapping declares which keys exist and how they
derive from `values` (e.g. `imageAllowTags: "regexp:^{{.Values.imageTagPrefix}}.*"`),
on top of the context params the controller always provides (`previewID`,
`configPR`, `hostNamespace`, `applicationName`, `persistentVolumeName`).
`hostname` is additionally a well-known key: the controller mirrors it into
the PreviewEnvironment status as the UI's preview URL. The replacements
(owned here, versioned with the branch) fan the params out to the app's
actual resources. If a feature branch renames or restructures
resources, it updates base/env overlays and these replacements **in the same
commit** — the platform has nothing left to invalidate.

## How params are injected — Git is never written

The controller does not commit, push, or edit files anywhere. It only creates
the Application CR in-cluster, roughly:

```yaml
spec:
  sources:
    # Inherited from web-playground.argocd.yaml @ ref; path swapped
    # playground -> previews/playground (path-matched), injections added:
    - repoURL: git@github.com:notniknot/web-apps-config.git
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
                value: {previewID: pr42, configPR: "17",
                        hostname: web-pr42.sonia-certs.uk,
                        imageAllowTags: "regexp:^pr42-.*", ...}
          - target: {group: preview.sonia.so, kind: PreviewTemplate, name: web}
            patch: |-
              $patch: delete
              apiVersion: preview.sonia.so/v1alpha1
              kind: PreviewTemplate
              metadata:
                name: web
    # Inherited unchanged except the ref pin and the selector-matched
    # sourceOverride (preview helm values):
    - repoURL: git@github.com:notniknot/web-apps-config.git
      targetRevision: <main | PR head SHA>
      path: apps/web/previews/helm/addon
      helm:
        valueFiles: [values.yaml, ../values/preview-values.yaml]
        valuesObject: {message: preview-pr42}
```

At sync time the Argo CD repo-server checks out the ref into a scratch dir,
folds these options into that checkout's kustomization (`kustomize edit …`),
and builds. Kustomize applies patches **before** replacements, so the injected
`preview-params` data — not the placeholders committed here — is what the
replacements fan out. The checkout is discarded after render; deleting a
preview deletes the Application and leaves no trace in Git. The placeholder
values in `component/preview-params.yaml` exist only so local
`kustomize build` works.

## Controller migration path

This layout can be reached in two phases (idea credit: the alternative
`feat/branch-declared-preview-profile` design):

1. **Path rewrite** — today's controller gains only `sources[].path` support;
   templates keep authoring the params patch via `patchTemplate`. Previews
   gain branch fidelity and this overlay structure immediately.
2. **Built-in contract with source inheritance** — the controller owns the
   params patch, the template delete, and the Application envelope; sources
   are inherited from the env's Application manifest with the overlay source
   swapped by **path match** (`preview.overlayPath`) and per-source tweaks
   via selector-matched `preview.sourceOverrides`. Index-based source rules
   and per-app `patchTemplate` boilerplate are deleted (kept only as a
   documented escape hatch). Phase 2 must be scheduled explicitly — the two
   mechanisms coexisting indefinitely is how stale-path bugs happen.

## Local check

```sh
kustomize build apps/web/previews/playground
kustomize build apps/web/previews/staging
```
