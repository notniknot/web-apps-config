# Preview overlay

This directory is the **complete definition of a preview environment** for the
`web` app — a normal kustomize overlay, sibling to `playground/`, owned by the
app team. The preview controller does **not** patch app resources anymore; it
instantiates this overlay.

## Contract with the platform

For each preview `<id>`, the controller generates an Argo CD `Application`
(name, project, destination, sync policy are platform-owned) with:

| Injection                              | Mechanism                              | Structure-coupled? |
|----------------------------------------|----------------------------------------|--------------------|
| source → `apps/web/previews` @ config ref | generated Application source        | no |
| target namespace `web-preview-<id>`    | `spec.destination` + `kustomize.namespace` | no |
| preview labels                          | `kustomize.commonLabels`               | no |
| per-instance values (hostname, pr, allowTags, pvName, …) | one kustomize patch on the **`preview-params` ConfigMap `data`** | no |
| per-preview helm values (addon source)  | `helm.valuesObject` from the template  | no |

Everything the platform touches is value-shaped. The **replacements in
`kustomization.yaml`** (owned here, versioned with the branch) fan the params
out to the app's actual resources. If a feature branch renames or restructures
resources, it updates base and these replacements **in the same commit** — the
platform has nothing left to invalidate.

## What lives where

- `preview-params.yaml` — the contract ConfigMap. Placeholders keep
  `kustomize build` working locally; the controller overwrites `data`.
- `kustomization.yaml` — includes `../base`, preview-only resources, static
  preview-shape patches (e.g. ImageUpdater → `argocd` writeback +
  `newest-build`; the ImageUpdater itself stays in `base/` since it is shared
  across clusters), and the replacements.
- `preview-demo-configmap.yaml`, `generated-preview-secret.yaml`,
  `dependency-verifier-job.yaml` — resources that only exist in previews
  (formerly `previews/components/`).
- `../playground/preview.yaml` — the `PreviewTemplate`: identity, default
  values, UI-exposed overrides, `preview.overlayPath`, extra sources. Applied
  live by the playground app (so the UI can read it) and read from Git at the
  requested ref by the controller (so a PR can change it).

## Local check

```sh
kustomize build apps/web/previews
```
