# Phase 2: Upstream chart enhancements (PBI #727210)

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). Gap item IDs refer to the gap analysis (`icrc-kobo-toolbox`: `work/completed/icrc-helm-gap-analysis.md`). Issue drafts and existing upstream items: [upstream-issues.md](./upstream-issues.md).

Upstream PR follow-up runs in the kobo-sync workflow; tagging a fork release is part of the release procedure (`ICRC.md`). Only `icrc-helm-setup#6.2` waits for upstream merges.

## Procedure per step

Topic branch `feature/<name>` from `kobo/main` with the fix and its helm-unittest cases, `helm lint --strict`, render diff, then a `Chart.yaml` version bump and `CHANGELOG.md` entry (required by upstream CI); upstream PR from that branch. The branch is merged (`--no-ff`) into `release/7.0.2-icrc` (the only integration branch, see `ICRC.md`); the step is done once merged there.

## Phase 2 upstream-enhancements: Upstream chart enhancements
Definition: the chart gaps blocking the ICRC deployment are fixed in the fork and proposed upstream, so an ICRC release of the fork can ship them until Kobo releases them. Upstream merge is out of scope (`icrc-helm-setup#6.2`); tagging is part of the release procedure (`ICRC.md`).
- [x] 2.0 contribution-prereqs: Contribution prerequisites: one issue per enhancement, check stale `feature/existing-secrets`, ask about the helm-unittest suite (#727250)
  - Definition: maintainers informed before any PR; reusable upstream work identified ([upstream-issues.md](./upstream-issues.md))
  - Issues 113 to 117 opened, comment on issue 23; stale branch has nothing to reuse
  - helm-unittest suite welcomed by maintainer 2026-10-08 ([issue 118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118)); [PR 124](https://github.com/kobotoolbox/kobo-helm-chart/pull/124) open, on release branch
- [x] 2.1 existing-secret: Existing-secret support for kpi, enketo and the migration/post-install Jobs; `djangoSecret`, `enketoApiKey`, `DATABASE_URL` optional when an existing secret is set (gap S4, #727218, [issue 23 comment](https://github.com/kobotoolbox/kobo-helm-chart/issues/23#issuecomment-5993930688), [PR 125](https://github.com/kobotoolbox/kobo-helm-chart/pull/125))
  - Definition: one `existingSecret` per component, `checksum/secret` dropped when set, default output unchanged; scope approved by maintainer 2026-10-07
  - `kpi.existingSecret`, `enketo.existingSecret`; Jobs also stop inlining `KC_DATABASE_URL`; chart 7.3.0
  - Default render byte-identical to `kobo/main` for 3 value sets; 10 new tests, 8 fail on the old templates
  - On release branch 2026-10-09: `helm lint --strict` passes, 61 tests pass; PR 125 opened
- [x] 2.2 post-install-envfrom: `envFrom` on the post-install Job (gap W6, #727219, [issue 113](https://github.com/kobotoolbox/kobo-helm-chart/issues/113), [PR 119](https://github.com/kobotoolbox/kobo-helm-chart/pull/119))
  - Definition: post-install Job loads the kpi Secret and ConfigMap like the migration Job, no new values
  - Merged upstream 2026-10-07 (Kobo 7.0.1), on release branch
- [x] 2.3 nginx-port: nginx sidecar configurable port and securityContext, unprivileged image on OpenShift (gap S2, #727220, [issue 114](https://github.com/kobotoolbox/kobo-helm-chart/issues/114), [PR 122](https://github.com/kobotoolbox/kobo-helm-chart/pull/122))
  - Definition: `nginx.port` and `nginx.securityContext` values, defaults keep the current output
  - On release branch 2026-10-09 (chart 7.1.0); PR 122 re-review after the rebase on 7.0.2 is kobo-sync follow-up
- [x] 2.4 nginx-protected-media: nginx sidecar extra volume mounts and filesystem `/protected/` location for X-Accel-Redirect (gap D2, #727221, [issue 115](https://github.com/kobotoolbox/kobo-helm-chart/issues/115), [PR 123](https://github.com/kobotoolbox/kobo-helm-chart/pull/123))
  - Definition: attachment downloads work with filesystem storage; `nginx.extraVolumeMounts` and `kpi.nginx.protectedMediaPath`, defaults keep the current output
  - On release branch 2026-10-09 (chart 7.2.0); PR 123 re-review after the rebase on 7.0.2 is kobo-sync follow-up
- [x] 2.6 image-pull-secrets: Render `imagePullSecrets` in pod specs (gap S3, #727223, [issue 117](https://github.com/kobotoolbox/kobo-helm-chart/issues/117), [PR 121](https://github.com/kobotoolbox/kobo-helm-chart/pull/121))
  - Definition: every pod spec renders `imagePullSecrets`, also with `serviceAccount.create: false`
  - Merged upstream 2026-10-08 (Kobo 7.0.2), on release branch
- Not done (`proxy_pass` already forwards the incoming `X-Forwarded-Proto` and OpenShift edge routes set it; confirmed by maintainer test on PR 120; PR 120 and issue 116 closed 2026-10-08, not on release branch): 2.5 forward-proto (gap S8, #727222)
- [x] Done: steps 2.1 to 2.4 and 2.6 each have an upstream PR (119, 121, 122, 123, 125) from a `feature/<name>` branch on `kobo/main` with helm-unittest cases, `Chart.yaml` bump and `CHANGELOG.md` entry; all merged into `release/7.0.2-icrc` (2026-10-09, `bb6f4a9`), where `helm lint --strict` passes and `helm unittest .` runs 61 tests, all passing

## Work items

PBI #727210 (Done). Tracked per step.

| Plan item | Task |
|---|---|
| 2.0 contribution-prereqs | #727250 |
| 2.1 existing-secret | #727218 |
| 2.2 post-install-envfrom | #727219 |
| 2.3 nginx-port | #727220 |
| 2.4 nginx-protected-media | #727221 |
| 2.5 forward-proto | #727222 |
| 2.6 image-pull-secrets | #727223 |
