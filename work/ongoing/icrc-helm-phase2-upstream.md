# Phase 2: Upstream chart enhancements (PBI #727210)

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). Gap item IDs refer to the gap analysis (`icrc-kobo-toolbox`: `work/completed/icrc-helm-gap-analysis.md`). Issue drafts and existing upstream items: [upstream-issues.md](./upstream-issues.md). Done steps: [completed](../completed/icrc-helm-phase2-upstream.md).

Runs in parallel with phases 3 to 5 of `icrc-helm-setup`: they use the fork build until the PRs are merged and released. Only `icrc-helm-setup#6.2` waits for upstream merges.

## Procedure per step

Topic branch `feature/<name>` from `kobo/main` with the fix and its helm-unittest cases, `helm lint --strict`, render diff, then a `Chart.yaml` version bump and `CHANGELOG.md` entry (required by upstream CI); upstream PR from that branch. The branch is merged (`--no-ff`) into `release/7.0.2-icrc` (the only integration branch, see `ICRC.md`), again after review changes.

## Phase 2 upstream-enhancements: Upstream chart enhancements
Definition: the chart gaps blocking the ICRC deployment are fixed in the fork and proposed upstream, so the umbrella chart consumes a tagged fork release until Kobo releases them. Upstream merge is out of scope (`icrc-helm-setup#6.2`).
- [ ] 2.1 existing-secret: Existing-secret support for kpi, enketo and the migration/post-install Jobs; `djangoSecret`, `enketoApiKey`, `DATABASE_URL` optional when an existing secret is set (gap S4, #727218, [issue 23 comment](https://github.com/kobotoolbox/kobo-helm-chart/issues/23#issuecomment-5993930688))
  - Definition: one `existingSecret` per component, `checksum/secret` dropped when set, default output unchanged; scope approved by maintainer 2026-10-07
- [ ] 2.3 nginx-port: nginx sidecar configurable port and securityContext, unprivileged image on OpenShift (gap S2, #727220, [issue 114](https://github.com/kobotoolbox/kobo-helm-chart/issues/114), [PR 122](https://github.com/kobotoolbox/kobo-helm-chart/pull/122))
  - Definition: `nginx.port` and `nginx.securityContext` values, defaults keep the current output
  - On release branch; approval dismissed by the rebase on 7.0.2 (2026-10-09), re-review needed
- [ ] 2.4 nginx-protected-media: nginx sidecar extra volume mounts and filesystem `/protected/` location for X-Accel-Redirect (gap D2, #727221, [issue 115](https://github.com/kobotoolbox/kobo-helm-chart/issues/115), [PR 123](https://github.com/kobotoolbox/kobo-helm-chart/pull/123))
  - Definition: attachment downloads work with filesystem storage; `nginx.extraVolumeMounts` and `kpi.nginx.protectedMediaPath`, defaults keep the current output
  - On release branch; approval dismissed by the rebase on 7.0.2 (2026-10-09), re-review needed
- [ ] Done when: steps 2.1 to 2.4 and 2.6 each have an upstream PR from a `feature/<name>` branch on `kobo/main` with helm-unittest cases, `Chart.yaml` bump and `CHANGELOG.md` entry; all their branches are merged into `release/7.0.2-icrc` where `helm lint --strict` and `helm unittest .` pass; a tag `7.0.2-icrc.<n>` on that branch contains all of them and is consumed by the umbrella chart

## Work items

PBI #727210. Tracked per step.

| Plan item | Task |
|---|---|
| 2.0 contribution-prereqs | #727250 |
| 2.1 existing-secret | #727218 |
| 2.2 post-install-envfrom | #727219 |
| 2.3 nginx-port | #727220 |
| 2.4 nginx-protected-media | #727221 |
| 2.5 forward-proto | #727222 |
| 2.6 image-pull-secrets | #727223 |
