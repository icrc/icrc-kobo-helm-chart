# Phase 2: Upstream chart enhancements (PBI #727210)

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). Gap item IDs refer to the gap analysis (`icrc-kobo-toolbox`: `work/completed/icrc-helm-gap-analysis.md`). Issue drafts and existing upstream items: [upstream-issues.md](./upstream-issues.md).

Runs in parallel with phases 3 to 5: they use the fork build until the PRs are merged and released. Only phase 6 step 2 waits for upstream merges.

## Completion target

- Each of steps 1 to 6 has an open upstream PR, from a `feature/<name>` branch on `kobo/main`, including helm-unittest cases, `Chart.yaml` bump and `CHANGELOG.md` entry.
- All six feature branches are merged into `release/7.0.0-icrc`, and `helm lint --strict` plus `helm unittest .` pass there.
- A tag `7.0.0-icrc.<n>` on `release/7.0.0-icrc` contains all six fixes and is the version consumed by the umbrella chart.

Upstream merge is not part of this target (tracked in phase 6 step 2).

## Procedure per step

Topic branch `feature/<name>` from `kobo/main` with the fix and its helm-unittest cases, `helm lint --strict`, render diff, then a `Chart.yaml` version bump and `CHANGELOG.md` entry (required by upstream CI); upstream PR from that branch. The branch is merged (`--no-ff`) into `release/7.0.0-icrc` (the only integration branch, see `ICRC.md`), again after review changes.

## Steps

| # | Step | Gap | ADO | GitHub | Status |
|---|---|---|---|---|---|
| 0 | Contribution prerequisites: one issue per enhancement, check stale `feature/existing-secrets`, ask about the helm-unittest suite | | #727250 | unittest suite: [issue 118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118) | done |
| 1 | Existing-secret support for kpi, enketo and the migration/post-install Jobs; `djangoSecret`, `enketoApiKey`, `DATABASE_URL` optional when an existing secret is set | S4 | #727218 | [issue 23 comment](https://github.com/kobotoolbox/kobo-helm-chart/issues/23#issuecomment-5993930688) | todo |
| 2 | `envFrom` on the post-install Job | W6 | #727219 | [issue 113](https://github.com/kobotoolbox/kobo-helm-chart/issues/113), [PR 119](https://github.com/kobotoolbox/kobo-helm-chart/pull/119) | PR open, on release branch |
| 3 | nginx sidecar: configurable port and securityContext (unprivileged image on OpenShift) | S2 | #727220 | [issue 114](https://github.com/kobotoolbox/kobo-helm-chart/issues/114), [PR 122](https://github.com/kobotoolbox/kobo-helm-chart/pull/122) | PR open, on release branch |
| 4 | nginx sidecar: extra volume mounts and filesystem `/protected/` location for X-Accel-Redirect | D2 | #727221 | [issue 115](https://github.com/kobotoolbox/kobo-helm-chart/issues/115), [PR 123](https://github.com/kobotoolbox/kobo-helm-chart/pull/123) | PR open, on release branch |
| 5 | nginx sidecar: forward `X-Forwarded-Proto` | S8 | #727222 | [issue 116](https://github.com/kobotoolbox/kobo-helm-chart/issues/116), [PR 120](https://github.com/kobotoolbox/kobo-helm-chart/pull/120) | PR open, on release branch |
| 6 | Render `imagePullSecrets` in pod specs | S3 | #727223 | [issue 117](https://github.com/kobotoolbox/kobo-helm-chart/issues/117), [PR 121](https://github.com/kobotoolbox/kobo-helm-chart/pull/121) | PR open, on release branch |
