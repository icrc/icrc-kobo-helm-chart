# Phase 2: Upstream chart enhancements (PBI #727210) - done steps

Open steps and work items: [ongoing](../ongoing/icrc-helm-phase2-upstream.md).

## Phase 2 upstream-enhancements: Upstream chart enhancements
Definition: the chart gaps blocking the ICRC deployment are fixed in the fork and proposed upstream, so the umbrella chart consumes a tagged fork release until Kobo releases them. Upstream merge is out of scope (`icrc-helm-setup#6.2`).
- [x] 2.0 contribution-prereqs: Contribution prerequisites: one issue per enhancement, check stale `feature/existing-secrets`, ask about the helm-unittest suite (#727250)
  - Definition: maintainers informed before any PR; reusable upstream work identified ([upstream-issues.md](../ongoing/upstream-issues.md))
  - Issues 113 to 117 opened, comment on issue 23; stale branch has nothing to reuse
  - helm-unittest suite welcomed by maintainer 2026-10-08 ([issue 118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118)); [PR 124](https://github.com/kobotoolbox/kobo-helm-chart/pull/124) open, on release branch
- [x] 2.2 post-install-envfrom: `envFrom` on the post-install Job (gap W6, #727219, [issue 113](https://github.com/kobotoolbox/kobo-helm-chart/issues/113), [PR 119](https://github.com/kobotoolbox/kobo-helm-chart/pull/119))
  - Definition: post-install Job loads the kpi Secret and ConfigMap like the migration Job, no new values
  - Merged upstream 2026-10-07 (Kobo 7.0.1), on release branch
- [x] 2.6 image-pull-secrets: Render `imagePullSecrets` in pod specs (gap S3, #727223, [issue 117](https://github.com/kobotoolbox/kobo-helm-chart/issues/117), [PR 121](https://github.com/kobotoolbox/kobo-helm-chart/pull/121))
  - Definition: every pod spec renders `imagePullSecrets`, also with `serviceAccount.create: false`
  - Merged upstream 2026-10-08 (Kobo 7.0.2), on release branch
- Not done (`proxy_pass` already forwards the incoming `X-Forwarded-Proto` and OpenShift edge routes set it; confirmed by maintainer test on PR 120; PR 120 and issue 116 closed 2026-10-08, not on release branch): 2.5 forward-proto (gap S8, #727222)
