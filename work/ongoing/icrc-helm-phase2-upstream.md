# Phase 2: Upstream chart enhancements (PBI #727210)

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). Gap item IDs refer to the gap analysis (`icrc-kobo-toolbox`: `work/completed/icrc-helm-gap-analysis.md`). Issue drafts and existing upstream items: [upstream-issues.md](./upstream-issues.md). Done steps: [completed](../completed/icrc-helm-phase2-upstream.md).

Runs in parallel with phases 3 to 5 of `icrc-helm-setup`: they use the fork build until the PRs are merged and released. Only `icrc-helm-setup#6.2` waits for upstream merges.

## Procedure per step

Topic branch `feature/<name>` from `kobo/main` with the fix and its helm-unittest cases, `helm lint --strict`, render diff, then a `Chart.yaml` version bump and `CHANGELOG.md` entry (required by upstream CI); upstream PR from that branch. The branch is merged (`--no-ff`) into `release/7.0.2-icrc` (the only integration branch, see `ICRC.md`); the step is done once merged there. Upstream review follow-up (changes, rebases, merge) is handled by the kobo-sync workflow.

Phase 2 is done ([completed](../completed/icrc-helm-phase2-upstream.md)). Upstream PR follow-up runs in the kobo-sync workflow; tagging a fork release is part of the release procedure (`ICRC.md`).

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
