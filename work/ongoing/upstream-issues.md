# Upstream issues for kobotoolbox/kobo-helm-chart (draft)

Drafts for [phase 2](./icrc-helm-phase2-upstream.md) step 0 (#727250). Post from the ICRC GitHub account, then record the issue numbers in the table.

## Existing upstream items

| Item | State | Relevance |
|---|---|---|---|
| [#23](https://github.com/kobotoolbox/kobo-helm-chart/issues/23) Support existing secrets/config mapping | open (2023) | Same need as S4. Maintainer asked to follow the bitnami `existingSecret` convention. Comment there instead of opening a new issue. |
| [#95](https://github.com/kobotoolbox/kobo-helm-chart/pull/95) existingSecret pattern | open PR (2026-01), no review | Based on pre-valkey `main`; removes `DJANGO_SECRET_KEY` / `ENKETO_API_KEY` from the generated secret for all users; one secret shared by kpi and enketo; migration Job still inlines `DJANGO_SECRET_KEY` and requires `DATABASE_URL`; post-install Job not covered; escaped quotes (`\"`) in beat and worker templates break rendering. Not reusable as is. |
| [#33](https://github.com/kobotoolbox/kobo-helm-chart/pull/33) Implement Image Pull Secrets | closed unmerged (2023) | Same need as S3. Reference it. |
| `feature/existing-secrets` branch | stale (2023) | Only adds empty `existingSecret` keys for subcharts. Nothing to reuse. |

No existing item for OpenShift / unprivileged nginx, `X-Forwarded-Proto`, post-install `envFrom`, filesystem protected media or tests. Several external PRs (#75 to #79) are unreviewed: expect slow merges.

## Tracking

| ADO | Topic | GitHub | PR |
|---|---|---|---|
| #727218 | Existing secrets (S4) | [#23 comment](https://github.com/kobotoolbox/kobo-helm-chart/issues/23#issuecomment-5993930688) | |
| #727219 | Post-install Job env (W6) | [#113](https://github.com/kobotoolbox/kobo-helm-chart/issues/113) | [#119](https://github.com/kobotoolbox/kobo-helm-chart/pull/119) |
| #727220 | nginx sidecar port and securityContext (S2) | [#114](https://github.com/kobotoolbox/kobo-helm-chart/issues/114) | |
| #727221 | nginx sidecar filesystem protected media (D2) | [#115](https://github.com/kobotoolbox/kobo-helm-chart/issues/115) | |
| #727222 | nginx sidecar `X-Forwarded-Proto` (S8) | [#116](https://github.com/kobotoolbox/kobo-helm-chart/issues/116) | [#120](https://github.com/kobotoolbox/kobo-helm-chart/pull/120) |
| #727223 | `imagePullSecrets` (S3) | [#117](https://github.com/kobotoolbox/kobo-helm-chart/issues/117) | [#121](https://github.com/kobotoolbox/kobo-helm-chart/pull/121) |
| #727217 | helm-unittest suite | [#118](https://github.com/kobotoolbox/kobo-helm-chart/issues/118) | |

---

## 1. Comment on #23: existing secrets for kpi and enketo

We would like to work on this and open a PR. Proposed scope, before writing code:

- New values `kpi.existingSecret` and `enketo.existingSecret` (bitnami-style: name of a Secret managed outside the chart).
- When set, the chart does not render the corresponding Secret and every consumer uses the existing one: kpi, beat, all workers, migration Job, post-install Job (kpi); enketo Deployment (enketo).
- `kobotoolbox.djangoSecret`, `kobotoolbox.enketoApiKey` and `kpi.env.secret.DATABASE_URL` are no longer `required` when the matching existing secret is set. The migration and post-install Jobs read `DJANGO_SECRET_KEY` and `DATABASE_URL` from the secret instead of inlining them.
- The existing secret must hold the same keys the chart would generate (documented in `values.yaml`).
- Default behaviour is unchanged: without `existingSecret`, the rendered output is identical.

Open questions: one value per component as above, or a single `kobotoolbox.existingSecret`? Should the `checksum/secret` annotation be dropped when an existing secret is used (it no longer reflects the content)?

PR #95 covers part of this but removes keys from the generated secret for all users; we would start from current `main` instead. Happy to coordinate with its author.

## 2. Post-install Job does not load the kpi secret and configmap

**Problem.** `templates/kpi/post-install-job.yaml` sets a few env vars inline but has no `envFrom`, unlike the migration Job. Commands that need the full kpi environment fail or behave differently. Example: `postInstall.command: ./manage.py create_kobo_superuser` cannot see `KOBO_SUPERUSER_USERNAME` / `KOBO_SUPERUSER_PASSWORD` set in `kpi.env.secret`.

**Proposal.** Add the same `envFrom` as the migration Job (kpi Secret and ConfigMap). Since the Job runs as a `post-install` hook, both resources already exist. No new values.

## 3. nginx sidecar: configurable port and securityContext

**Problem.** The kpi nginx sidecar listens on port 80, hardcoded in the container port, the Service `targetPort` and `nginx.conf`. It has no `securityContext`. Clusters enforcing restricted Pod Security (or OpenShift `restricted-v2`) reject or break it: non-root containers cannot bind port 80. Unprivileged images such as `nginxinc/nginx-unprivileged` listen on 8080.

**Proposal.**
- `nginx.port` (default `80`), used by the container port, the Service `targetPort` and `listen` in `nginx.conf`.
- `nginx.securityContext` (default `{}`), applied to the sidecar container.
- Defaults keep the current output.

## 4. nginx sidecar: serve protected media from the filesystem

**Problem.** With filesystem storage (no S3/GCS), kpi answers attachment downloads with `X-Accel-Redirect` to `/protected/...`, expecting nginx to serve the file from the media directory. The sidecar only defines `/protected-s3/`, and cannot mount the media volume (`kpi.extraVolumeMounts` only applies to the backend container). Attachment downloads fail for filesystem-storage deployments.

**Proposal.**
- `nginx.extraVolumeMounts` (default `[]`) for the sidecar, using volumes from `kpi.extraVolumes`.
- `kpi.nginx.protectedMediaPath` (default empty): when set, renders an `internal` `location /protected/` aliasing that path.

To confirm with maintainers: the exact internal path kpi uses for filesystem storage in current releases.

## 5. nginx sidecar: forward X-Forwarded-Proto

**Problem.** The sidecar sets `Host`, `X-Real-IP` and `X-Forwarded-For` but not `X-Forwarded-Proto`. Behind a TLS-terminating ingress, Django's `SECURE_PROXY_SSL_HEADER` (`HTTP_X_FORWARDED_PROTO`) then sees `http`, which can cause redirect loops or `http://` absolute URLs depending on settings.

**Proposal.** Pass through the incoming header when present, otherwise use the request scheme:

```nginx
map $http_x_forwarded_proto $forwarded_proto {
    default $http_x_forwarded_proto;
    ''      $scheme;
}
...
proxy_set_header X-Forwarded-Proto $forwarded_proto;
```

## 6. imagePullSecrets is declared but never rendered

**Problem.** `values.yaml` declares `imagePullSecrets: []`, but no template uses it. Pulling kpi/enketo/nginx from a private registry requires patching the ServiceAccount outside the chart. PR #33 (2023) addressed this on the ServiceAccount but was closed unmerged.

**Proposal.** Render `imagePullSecrets` in every pod spec (kpi, enketo, beat, workers, flower, Jobs), so it also works with `serviceAccount.create: false`. Format: list of `{name: ...}` objects, as in the default `helm create` scaffold.

## 7. Unit tests for templates

**Question.** We added a [helm-unittest](https://github.com/helm-unittest/helm-unittest) suite (`tests/`, 35 tests covering kpi, enketo, jobs, nginx config, secrets, service account) to secure our changes. Would you welcome it upstream, with a CI step in `pr.yml`? If so, we would open it as a separate PR first, then add tests to each enhancement PR.
