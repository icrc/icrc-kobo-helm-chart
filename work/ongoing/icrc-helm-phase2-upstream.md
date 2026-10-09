# Phase 2: Upstream chart enhancements (PBI #727210)

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). Gap item IDs refer to the gap analysis (`icrc-kobo-toolbox`: `work/completed/icrc-helm-gap-analysis.md`). Issue drafts and existing upstream items: [upstream-issues.md](./upstream-issues.md).

Runs in parallel with phases 3 to 5: they use the fork build until the PRs are merged and released. Only phase 6 step 2 waits for upstream merges.

## Completion target

- Each of steps 1 to 4 and 6 has an upstream PR, from a `feature/<name>` branch on `kobo/main`, including helm-unittest cases, `Chart.yaml` bump and `CHANGELOG.md` entry.
- All five feature branches are merged into `release/7.0.1-icrc`, and `helm lint --strict` plus `helm unittest .` pass there.
- A tag `7.0.1-icrc.<n>` on `release/7.0.1-icrc` contains all five fixes and is the version consumed by the umbrella chart.

Upstream merge is not part of this target (tracked in phase 6 step 2).

## Procedure per step

Topic branch `feature/<name>` from `kobo/main` with the fix and its helm-unittest cases, `helm lint --strict`, render diff, then a `Chart.yaml` version bump and `CHANGELOG.md` entry (required by upstream CI); upstream PR from that branch. The branch is merged (`--no-ff`) into `release/7.0.1-icrc` (the only integration branch, see `ICRC.md`), again after review changes.

## Steps

| # | Step | Gap | ADO | GitHub | Status |
|---|---|---|---|---|---|
| 0 | Contribution prerequisites: one issue per enhancement, check stale `feature/existing-secrets`, ask about the helm-unittest suite | | #727250 | unittest suite: [issue 118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118) | done |
| 1 | Existing-secret support for kpi, enketo and the migration/post-install Jobs; `djangoSecret`, `enketoApiKey`, `DATABASE_URL` optional when an existing secret is set | S4 | #727218 | [issue 23 comment](https://github.com/kobotoolbox/kobo-helm-chart/issues/23#issuecomment-5993930688) | todo; scope approved by maintainer 2026-10-07: one `existingSecret` per component, drop `checksum/secret` when set |
| 2 | `envFrom` on the post-install Job | W6 | #727219 | [issue 113](https://github.com/kobotoolbox/kobo-helm-chart/issues/113), [PR 119](https://github.com/kobotoolbox/kobo-helm-chart/pull/119) | merged upstream 2026-10-07, on release branch |
| 3 | nginx sidecar: configurable port and securityContext (unprivileged image on OpenShift) | S2 | #727220 | [issue 114](https://github.com/kobotoolbox/kobo-helm-chart/issues/114), [PR 122](https://github.com/kobotoolbox/kobo-helm-chart/pull/122) | approval dismissed by rebase on 7.0.2 (2026-10-09), re-review needed; on release branch |
| 4 | nginx sidecar: extra volume mounts and filesystem `/protected/` location for X-Accel-Redirect | D2 | #727221 | [issue 115](https://github.com/kobotoolbox/kobo-helm-chart/issues/115), [PR 123](https://github.com/kobotoolbox/kobo-helm-chart/pull/123) | approval dismissed by rebase on 7.0.2 (2026-10-09), re-review needed; on release branch |
| 5 | nginx sidecar: forward `X-Forwarded-Proto` | S8 | #727222 | [issue 116](https://github.com/kobotoolbox/kobo-helm-chart/issues/116), [PR 120](https://github.com/kobotoolbox/kobo-helm-chart/pull/120) | dropped: PR and issue closed 2026-10-08, not on release branch |
| 6 | Render `imagePullSecrets` in pod specs | S3 | #727223 | [issue 117](https://github.com/kobotoolbox/kobo-helm-chart/issues/117), [PR 121](https://github.com/kobotoolbox/kobo-helm-chart/pull/121) | merged upstream 2026-10-08 (Kobo 7.0.2), on release branch |

- Not done (`proxy_pass` already forwards the incoming `X-Forwarded-Proto` and OpenShift edge routes set it; confirmed by maintainer test on PR 120): step 5.
