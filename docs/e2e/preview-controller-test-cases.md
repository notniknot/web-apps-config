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
