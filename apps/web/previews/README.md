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
helm/addon/               in-repo helm chart (extra Application source)
*/addon-values.yaml       helm values live NEXT TO the kustomizations
                          (base/, playground/, staging/, previews/<env>/),
                          referenced via $values — same convention as
                          infra-config apps (e.g. keda). The preview values
                          sit beside the preview kustomization, so a config
                          PR edits them like any other preview file.
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
| target namespace `web-preview-<id>` | `spec.destination.namespace` only — no `kustomize.namespace`, so explicit pins (ImageUpdater → `argocd`) survive (E2E-verified) | no |
| preview labels | `kustomize.commonLabels` | no |
| per-instance values (hostname, configPR, imageAllowTags, …) | one kustomize patch on the **`preview-params` ConfigMap `data`** | no |
| which values exist + which are user-overridable | template `values` (per-key `default`/`expose`/`description`) | no |
| how values become params keys | template `preview.params` mapping (value-shaped Go templates) | no |
| drop the env's `PreviewTemplate` from the render | `$patch: delete` appended by the controller | no (platform-owned kind) |
| per-source preview adjustments (helm values, chart pins, drops) | template `preview.sourceOverrides`, selector-matched | no |

Sources not present in the env's Application don't exist in the preview;
sources it declares pass through (bounded by the AppProject source allowlist).

### `sourceOverrides` semantics

- **Selectors** match on stable source identity: `path`, `chart`, `repoURL`,
  `ref`, `releaseName` (`helm.releaseName` — the discriminator when one chart
  is installed as several releases). Never by position/index.
- **Set semantics**: an override applies to *every* matching source. One
  `match: {chart: vector}` bumps all releases of that chart in lockstep; add
  `releaseName` to the match to target one. (Contrast: the `overlayPath` swap
  requires *exactly one* match and denies loudly on zero or several.)
- **Empty means inherit**: an override field that renders to the empty string
  (e.g. `targetRevision: "{{.Values.chartVersion}}"` with the value unset) is
  a no-op — the inherited field stays. Templates therefore never duplicate the
  Application's pins; they only declare what *may* change.
- **`action: drop`** removes all matching sources from the preview — for
  sources that must not run twice or cannot be namespace-confined (e.g. a
  log-shipper DaemonSet release). A drop override carries no other fields.

### Source ordering: keep Kustomize before Helm (Image Updater)

Previews inherit `spec.sources` **in the order the env Application manifest
declares them** — and that order is load-bearing for the image loop. Until
[argocd-image-updater#1725](https://github.com/argoproj-labs/argocd-image-updater/issues/1725)
is fixed, `method: argocd` write-back picks its target source by *type* in
`status.sourceTypes` list order, checking Helm before Kustomize. Rules for the
env Application manifest (e.g. `web-playground.argocd.yaml`):

- The env overlay (kustomize) source comes **first** in `spec.sources`, before
  any helm source, and is marked explicit with `kustomize: {}`.
- This holds **per branch**: a config PR that reorders sources (or a preview
  of an old branch predating this rule) silently breaks that preview's image
  updates — the updater writes junk `helm.parameters` onto the chart source,
  never `kustomize.images`, and logs success anyway.

This is deliberately a config-repo convention, not a controller rewrite: the
controller preserves inherited order faithfully, and the constraint disappears
once the upstream bug is fixed.

Everything the platform touches is value-shaped or targets platform-owned
kinds. The `preview-params.data` content is not controller-hardcoded: the
template's `preview.params` mapping declares which keys exist and how they
derive from `values` (e.g. `imageAllowTags: "regexp:^{{.Values.imageTagPrefix}}.*"`),
on top of the context params the controller always provides (`previewID`,
`configPR`, `hostNamespace`, `applicationName`). WEB-specific structure such
as the per-preview S3 `persistentVolumeName` is declared in this app's
`preview.params`, not by the controller.
`visible: true` on WEB's hostname value asks the controller to copy the
rendered hostname into PreviewEnvironment status for plain-text tile display. The replacements
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
        commonLabels: {preview.sonia.so/id: pr42, ...}
        patches:
          - target: {kind: ConfigMap, name: preview-params}
            patch: |-
              - op: replace
                path: /data
                value: {previewID: pr42, configPR: "17",
                        hostname: web-pr42.sonia-certs.uk,
                        imageAllowTags: "regexp:^pr42-.*",
                        persistentVolumeName: web-pr42-s3-public, ...}
          - target: {group: preview.sonia.so, kind: PreviewTemplate, name: web}
            patch: |-
              $patch: delete
              apiVersion: preview.sonia.so/v1alpha1
              kind: PreviewTemplate
              metadata:
                name: web
    # Inherited unchanged except the ref pin and the selector-matched
    # sourceOverride (preview values file instead of the env's):
    - repoURL: git@github.com:notniknot/web-apps-config.git
      targetRevision: <main | PR head SHA>
      path: apps/web/helm/addon
      helm:
        valueFiles:
          - $values/apps/web/base/addon-values.yaml
          - $values/apps/web/previews/playground/addon-values.yaml
        valuesObject: {message: preview-pr42}
    # Inherited ref source; SHA-pinned so $values files resolve at the PR ref:
    - repoURL: git@github.com:notniknot/web-apps-config.git
      targetRevision: <main | PR head SHA>
      ref: values
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
