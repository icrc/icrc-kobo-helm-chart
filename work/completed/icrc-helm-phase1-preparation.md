# Phase 1: Preparation (PBI #727209) - completed

Part of the ICRC Helm setup plan (`icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`).

## Completion target

- `kobo-helm-chart` has a `kobo` remote and a `release/7.0.0-icrc` branch pushed to `origin`, with the fork procedure in `ICRC.md`.
- `helm unittest .` passes on `release/7.0.0-icrc` and covers the templates touched in phase 2: kpi deployment, nginx sidecar config, secrets, migration and post-install Jobs.

Status: met. `ICRC.md` on `release/7.0.0-icrc`; suites under `tests/` (kpi deployment, service, nginx conf, secrets, jobs, post-install job, enketo, service account). Upstream interest asked in [issue 118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118).

## Steps

1. Set up the fork workflow in `kobo-helm-chart` (fork `icrc/icrc-kobo-helm-chart`): `kobo` remote (`kobo/main` reference), `release/<kobo-version>-icrc` branch with patch tags `<kobo-version>-icrc.<n>`, `icrc/main` for PROD, procedure in `ICRC.md`. (#727216, done)
2. Add helm-unittest tests covering the templates touched in phase 2 (kpi deployment, sidecar, secrets, jobs), required before any template change. (#727217, done)
