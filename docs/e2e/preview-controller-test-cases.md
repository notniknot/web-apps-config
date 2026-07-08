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
