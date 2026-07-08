# Preview Controller E2E Findings

Date: 2026-07-08

Target: https://argocd.sonia-certs.uk/argocd

Dedicated app: `apps/test-case`

## Test 1a - Baseline preview scaffold, first run

- Started: 2026-07-08 05:00 UTC
- Config commit: `9a0cb7b`
- Action: created `tc-baseline-0708` from the public Argo CD Previews UI with overrides `appName=custom-test-case`, `imageTagPrefix=preview-pr-4-`, `message=hello-e2e`, `ttl=1h`.
- Result: controller created `PreviewEnvironmentRequest`, `PreviewEnvironment`, namespace `web-preview-tc-baseline-0708`, generated Application `test-case-apps-preview-tc-baseline-0708`, Deployment, Service, HTTPRoute, and ImageUpdater.
- Verified: Deployment was `1/1` Ready; env fan-out worked (`APP_NAME=custom-test-case`, `GENERATED_SECRET_MESSAGE=hello-e2e`); ImageUpdater allowTags rendered as `regexp:^preview-pr-4-.*`.
- Finding: generated preview URL returned Envoy `404` because the dedicated test scaffold referenced non-existent Gateway `kgateway-system/public`; existing app routes use `kgateway-system/north-south`.
- Cleanup: deleted from public UI; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`.

## Test 1b - Baseline preview after route fix

- Started: 2026-07-08 05:07 UTC
- Config commit: `2308485`
- Action: created `tc-baseline-0708b` from the public Argo CD Previews UI with overrides `appName=custom-test-case`, `imageTagPrefix=preview-pr-4-`, `message=hello-e2e`, `ttl=1h`.
- Result: UI showed `web-preview-tc-baseline-0708b`, transitioned from `Pending` to `Synced` / `Healthy`, and exposed `Open`, `Argo CD`, `Edit`, and `Delete` actions.
- Verified: `PreviewEnvironmentRequest` and `PreviewEnvironment` reached `Ready`; generated Application `argocd/test-case-apps-preview-tc-baseline-0708b` was `Synced` / `Healthy`; Deployment was `1/1` Ready; HTTPRoute was `Accepted=True` and `ResolvedRefs=True`; ImageUpdater rendered `allowTags=regexp:^preview-pr-4-.*` and reported `Ready=True`.
- URL check: `https://test-case-tc-baseline-0708b.sonia-certs.uk` returned HTTP `200` and rendered `custom-test-case`, `preview-tc-baseline-0708b`, `baseline-shared`, and `hello-e2e`.
- Finding: baseline public UI create/open/read/delete flow works after the scaffold route fix.
- Cleanup: deleted from public UI; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`.

## Test 2 - Preview without rendered HTTPRoute

- Started: 2026-07-08 05:12 UTC
- Config commit: `a9868fa`
- Setup: temporarily removed `httproute.yaml` and the hostname replacement from the dedicated `test-case` app; verified the live base app no longer had `HTTPRoute/test-case`.
- Action: created `tc-no-route-0708` from the public Argo CD Previews UI with overrides `appName=no-route-app`, `imageTagPrefix=preview-no-route-`, `message=no-route-message`, `ttl=1h`.
- Result: generated preview became `Synced` / `Healthy` with Deployment `1/1` Ready and no rendered HTTPRoute in namespace `web-preview-tc-no-route-0708`.
- UI finding: while Pending, `Open` rendered as a disabled button; after Healthy, `Open` became an enabled link because `PreviewEnvironment.status.urls` still contained `https://test-case-tc-no-route-0708.sonia-certs.uk`.
- URL check: the enabled `Open` target returned Envoy HTTP `404`, because no HTTPRoute served the hostname.
- Finding: the controller/UI currently treat a hostname-derived URL as openable even when the rendered preview has no route; the Open button can lead to a dead URL for route-less apps.
- Cleanup: deleted from public UI; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`.

## Test 3 - Slow cleanup blocked by namespaced finalizer

- Started: 2026-07-08 05:16 UTC
- Config commit: `d332132`
- Setup: temporarily added preview-only ConfigMap `slow-cleanup-blocker` with finalizer `preview.sonia.so/e2e-slow-cleanup`.
- Action: created `tc-slow-clean-0708` from the public Argo CD Previews UI with overrides `appName=slow-clean-app`, `imageTagPrefix=preview-slow-clean-`, `message=slow-clean-message`, `ttl=1h`, then deleted it from the UI after it became `Synced` / `Healthy`.
- Result: UI showed `Deleting`; `PreviewEnvironmentRequest` had a deletion timestamp and status `Deleting` with message `waiting for Argo CD to prune application test-case-apps-preview-tc-slow-clean-0708`; generated Application was `OutOfSync` / `Progressing`; blocker ConfigMap had a deletion timestamp but retained the test finalizer.
- Finding: the controller did not orphan resources or prematurely remove the request while Argo CD prune was blocked; it kept the request in `Deleting` until the blocking finalizer was cleared.
- Cleanup: manually removed the test finalizer from `slow-cleanup-blocker`; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`.

## Test 4 - Config repository PR adds a preview resource

- Started: 2026-07-08 05:20 UTC
- Config PR: `notniknot/web-apps-config#8`, commit `e7231a8`, branch `codex/e2e-config-pr-add-resource`.
- Setup: PR added preview-only ConfigMap `config-pr-added-resource` under `apps/test-case/previews/component`.
- Action: enabled `Preview a config repository PR` in the public UI, entered config PR `8`, and created `tc-config-pr-add-0708` with overrides `appName=config-pr-add-app`, `imageTagPrefix=preview-config-pr-add-`, `message=config-pr-add-message`, `ttl=1h`.
- Result: UI showed chip `config PR #8`, transitioned to `Synced` / `Healthy`, and exposed `Open` and `Argo CD`; `PreviewEnvironment.spec.resolvedSources.config` recorded PR `8` and SHA `e7231a84dbd719d081dd37e6f936f9f9feed9f31`.
- Verified: generated namespace contained `ConfigMap/config-pr-added-resource` with `scenario=add-resource`; Deployment rendered `PREVIEW_PR=8`; ImageUpdater rendered `allowTags=regexp:^preview-config-pr-add-.*`; public URL returned HTTP `200` and rendered `config-pr-add-app`, PR `8`, and `config-pr-add-message`.
- Finding: config PR source override works for an added resource and for exposed values/params.
- Cleanup: deleted preview from public UI; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`; closed PR `#8` and deleted its branch.

## Test 5 - Config repository PR patches, removes, and renames resources

- Started: 2026-07-08 08:08 UTC
- Config PR: `notniknot/web-apps-config#9`, commit `09f8bfc`, branch `codex/e2e-config-pr-patch-remove-rename`.
- Setup: PR patched Deployment annotation/env/resources, renamed Service `test-case` to `test-case-pr-renamed`, updated HTTPRoute backendRef to the renamed Service, and removed ImageUpdater plus its preview replacements.
- Action: enabled `Preview a config repository PR` in the public UI, entered config PR `9`, and created `tc-pr9-patch-0708` with overrides `appName=pr9-patched-app`, `imageTagPrefix=preview-pr9-`, `message=pr9-message`, `ttl=1h`.
- Result: UI showed chip `config PR #9`, transitioned to `Synced` / `Healthy`, and exposed `Open` and `Argo CD`; `PreviewEnvironment.spec.resolvedSources.config` recorded PR `9` and SHA `09f8bfc57673f378fa4af1d928653710f007ca99`.
- Verified: generated namespace contained Service `test-case-pr-renamed`; HTTPRoute backendRef pointed to `test-case-pr-renamed` with `Accepted=True` and `ResolvedRefs=True`; Deployment had annotation `e2e.preview.sonia.so/config-pr-patched=true`, env `CONFIG_PR_PATCHED=true`, `PREVIEW_PR=9`, request CPU `30m`, and limit memory `160Mi`; ImageUpdater `argocd/test-case-apps-preview-tc-pr9-patch-0708` was absent; public URL returned HTTP `200` and rendered `pr9-patched-app` and `pr9-message`.
- Finding: config PR source override works for patched fields, removed resources/replacements, and resource rename/restructure when the PR keeps the app internally consistent.
- Cleanup: deleted preview from public UI; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespace all returned `NotFound`; closed PR `#9` and deleted its branch.

## Test 6 - Invalid config repository PR inputs

- Started: 2026-07-08 08:12 UTC
- Action: created invalid requests for closed config PR `#9`, missing config PR `#99999`, and non-numeric config PR value `nope`.
- Result: closed PR `#9` caused events `PreviewPullRequestClosed` and `PreviewDeleted`; the controller deleted `PreviewEnvironmentRequest/tc-invalid-closed-pr-0708` instead of leaving an Error CR. Missing PR `#99999` left `PreviewEnvironmentRequest/tc-invalid-missing-pr-0708` in `phase=Error` with reason `PullRequestNotFound` and message `config PR #99999 not found in notniknot/web-apps-config`. Non-numeric `configPullRequest: nope` was rejected by the API server because the field must be an integer.
- Verified: closed and missing PR cases did not create preview namespaces; missing-PR request was manually deleted after status capture.
- Finding: invalid PR handling is split by failure type: missing PR is a retained Error status, closed PR is treated as a delete signal and removes the request, and non-numeric input is schema-invalid.
- Cleanup: deleted the retained missing-PR request; no `tc-invalid-*` preview namespaces remained.

## Test 7 - PreviewTemplate values and params matrix

- Started: 2026-07-08 08:15 UTC
- Action: first tried creating `tc-values-params-0708` from the public UI with spaces and `&` in exposed values; then retried with allowed special characters `values/params:0708.alpha_ok-1`, `preview-values-0708-`, and `msg/with:allowed.value_ok-1`; separately created API request `tc-hidden-override-0708` attempting to override hidden value `hostname`.
- Result: public UI create returned HTTP `400` for unsupported characters with message `override appName contains unsupported characters (allowed: letters, digits, . _ : / -)`; the allowed-character preview became `Synced` / `Healthy`; hidden `hostname` override was rejected with `phase=Error`, reason `InvalidRequest`, and message `value "hostname" is not overridable for this application`.
- Verified: `preview-params` included controller-owned params `previewID`, `configPR`, `hostNamespace`, `applicationName` plus exposed value fan-out; Deployment env rendered `APP_NAME`, `PREVIEW_PR=0`, and `GENERATED_SECRET_MESSAGE`; HTTPRoute hostname rendered `test-case-tc-values-params-0708.sonia-certs.uk`; ImageUpdater rendered `allowTags=regexp:^preview-values-0708-.*`; public URL returned HTTP `200` and rendered the allowed values.
- Finding: exposed values are shape-validated consistently, hidden values cannot be overridden, and params fan out correctly into ConfigMap, Deployment, HTTPRoute, ImageUpdater, and the public app.
- Cleanup: deleted both requests; verified `PreviewEnvironmentRequest`, `PreviewEnvironment`, generated Application, and namespaces all returned `NotFound`.

## Test 8 - sourceOverrides drop with multi-source Application

- Started: 2026-07-08 08:21 UTC
- Config PR: `notniknot/web-apps-config#10`, commit `0e4bf22`, branch `codex/e2e-source-overrides-drop`.
- Setup: PR added a second config-repo Application source at `apps/test-case/extra` containing ConfigMap `source-override-should-be-dropped`, and added `PreviewTemplate.spec.preview.sourceOverrides` with `match.path=extra` and `action=drop`.
- Action: enabled `Preview a config repository PR` in the public UI, entered config PR `10`, and created `tc-src-drop-0708` with overrides `appName=source-drop-app`, `imageTagPrefix=preview-src-drop-`, `message=source-drop-message`, `ttl=1h`.
- Result: preview became `Synced` / `Healthy`; `PreviewEnvironment.spec.resolvedSources.config` recorded PR `10` and SHA `0e4bf221a385288b88532a42431948be8f86308f`.
- Verified: generated Application had only one source, `apps/test-case/previews/playground` pinned to the PR SHA; ConfigMap `source-override-should-be-dropped` was absent from the preview namespace; public URL returned HTTP `200` and rendered `source-drop-app` and `source-drop-message`.
- Finding: sourceOverrides `action=drop` works with selector matching by app-relative path, and a multi-source app can be reduced to the swapped preview source when the template intentionally drops the extra source.
- Cleanup: deleted preview from public UI; verified request and namespace returned `NotFound`; closed PR `#10` and deleted its branch.

## Test 9 - sourceOverrides no-match denial

- Started: 2026-07-08 08:25 UTC
- Config PR: `notniknot/web-apps-config#11`, commit `b93f630`, branch `codex/e2e-source-overrides-invalid`.
- Setup: PR added an effectful `sourceOverrides` entry with `match.path=does-not-exist` and `targetRevision=main`.
- Action: created request `tc-src-invalid-0708` against config PR `11`.
- Result: request stayed in `phase=Error` with reason `InvalidTemplate`; message reported `preview.sourceOverrides[0] ... matches no inherited source`.
- Verified: no preview namespace was created.
- Finding: sourceOverrides no-match denial is enforced for effectful overrides and surfaces a clear `InvalidTemplate` status.
- Cleanup: deleted the error request; closed PR `#11` and deleted its branch.
