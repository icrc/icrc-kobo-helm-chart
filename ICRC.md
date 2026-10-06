# ICRC fork

ICRC fork of [kobotoolbox/kobo-helm-chart](https://github.com/kobotoolbox/kobo-helm-chart), consumed by the ICRC umbrella chart in `icrc-kobo-toolbox`. The fork only carries generic enhancements proposed upstream, kept here while their PR is pending or if it is rejected. ICRC-only resources belong to the umbrella chart; a rejected enhancement moves there when it can be done without the fork.

## Remotes

| Remote | URL | Push |
|---|---|---|
| `origin` | git@github.com:icrc/icrc-kobo-helm-chart.git | yes |
| `kobo` | https://github.com/kobotoolbox/kobo-helm-chart.git | disabled |

The `icrc` organization enforces SAML SSO: the SSH key (or token) must be authorized for it in GitHub settings. Pin the ICRC key to this repository when the machine also holds a personal GitHub key.

```bash
git clone git@github.com:icrc/icrc-kobo-helm-chart.git && cd icrc-kobo-helm-chart
git remote add kobo https://github.com/kobotoolbox/kobo-helm-chart.git
git remote set-url --push kobo no_push
git config core.sshCommand "ssh -i ~/.ssh/<icrc-key> -o IdentitiesOnly=yes"
```

## Branches and tags

| Ref | Base | Role |
|---|---|---|
| `kobo/main` | - | Kobo reference (remote-tracking, read only). |
| `main` | `kobo/main` | Mirror of Kobo on the fork, never committed to. Pushing it triggers `publish-chart` if Actions are enabled on the fork. |
| `icrc-bootstrap` | `kobo/main` | `kobo/main` + fork-only changes (this file, `work/`), never proposed upstream. Always kept rebased on `kobo/main`. |
| `feature/<name>` | `kobo/main` | One per enhancement (upstream PR pending or rejected). Must bump `Chart.yaml` `version` and add a `CHANGELOG.md` entry (upstream CI). Not based on `icrc-bootstrap`, so the PR carries no fork-only change. |
| `develop` | `icrc-bootstrap` | `icrc-bootstrap` + all feature branches, to try them together before a release. Never tagged. |
| `release/<kobo-version>-icrc` | Kobo tag `<kobo-version>` | Kobo release + `icrc-bootstrap` changes + the feature branches validated for it. Deployed through the umbrella chart. One branch per Kobo release. |
| `<kobo-version>-icrc.<n>` tag | release branch | Build of the release branch, e.g. `7.0.0-icrc.1`. Also the chart version. |

`icrc-bootstrap`, `feature/*` and `develop` carry no tags and are rewritten freely: rebased when `kobo/main` advances, force-pushed. `develop` has no commits of its own beyond `icrc-bootstrap` and feature commits. Release branches and tags are frozen:

- A release branch only grows: new commits on top, never rebased nor force-pushed. A new Kobo release gets a new release branch.
- Tags are never moved nor deleted.
- Enforced by GitHub rulesets on `icrc/icrc-kobo-helm-chart`: no force-push or deletion on `release/*`, no update or deletion of `*-icrc.*` tags.
- `7.0.0-icrc.1` is a SemVer prerelease of `7.0.0`: the umbrella chart must pin it exactly, ranges like `~7.0.0` skip it.

## Work tracking

`work/` holds the plan of the chart phases of the ICRC Helm setup (overview in `icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). One file per phase, each with a completion target: `work/next/` before it starts, `work/ongoing/` while running, `work/completed/` once the target is met.

Like this file, `work/` is fork-only: edit it on `icrc-bootstrap` only. Copies on `develop` and release branches are snapshots, refreshed when those branches are rebuilt.

## Tests

[helm-unittest](https://github.com/helm-unittest/helm-unittest) suites live in `tests/`, with fixture values in `tests/values/required.yaml`. They pin the current behaviour of the templates changed by ICRC enhancements; each enhancement adds its own cases.

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest.git --verify=false
helm unittest .
helm lint --strict -f tests/values/required.yaml .
```

## Contributing an enhancement

1. Create `feature/<name>` from `kobo/main`. Commit the fix with its tests (`tests/<suite>_test.yaml`, plus `tests/values/required.yaml` if missing), then a separate commit with the version bump and changelog entry.
2. Push to `origin`, open the PR against `kobotoolbox/kobo-helm-chart:main`. Apply review changes on the feature branch.
3. Cherry-pick its commits onto `develop` (see [Applying features](#applying-features)) to try it with the other features. Same for later review changes.
4. To ship it, cherry-pick its commits onto the current release branch and tag a patch.
5. Merged upstream: delete the branch and recreate `develop`. Rejected: keep the branch, it is carried to every release.

## Syncing with Kobo

Run whenever `kobo/main` advances.

```bash
git fetch kobo --tags --prune
git switch main && git merge --ff-only kobo/main && git push origin main
```

**No PR merged upstream**: rebase every feature branch, then `develop` together with `icrc-bootstrap` (`--update-refs` moves `icrc-bootstrap` along with the stack).

```bash
# Works in bash and zsh
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/; }
for b in $(features); do git rebase kobo/main "$b" || break; done
git rebase --update-refs kobo/main develop
git push --force-with-lease origin $(features) icrc-bootstrap develop
```

On conflict a rebase stops: resolve, `git rebase --continue`, then re-run the commands (already rebased branches are no-ops). A Kobo release changes `Chart.yaml` `version` and `CHANGELOG.md`, so each version bump commit conflicts: re-bump above the new Kobo version.

**A PR was merged upstream**: delete its feature branch, rebase the remaining feature branches and `icrc-bootstrap` (`git rebase kobo/main icrc-bootstrap`), then recreate `develop` instead of rebasing it. Its copy of the feature commits would conflict with, or duplicate, the upstream merge (often squashed, so Git does not recognize them).

## Applying features

`develop` and release branches are built the same way: a base plus the feature commits, without their version bump commits (`chore: release ...`), which only serve upstream CI.

```bash
# apply <branch>...: cherry-pick the branch commits (kobo/main..<branch>) onto the current branch
apply() { for b; do git cherry-pick $(git rev-list --reverse --invert-grep --grep='^chore: release' kobo/main.."$b") || return; done; }
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/; }

# Recreate develop
git switch -C develop icrc-bootstrap && apply $(features) && git push --force-with-lease origin develop

# New release branch from a Kobo release, with the fork-only changes and the validated features
git switch -c release/<kobo-version>-icrc <kobo-version> && apply icrc-bootstrap feature/<a> feature/<b>

# Add new commits of a feature to develop or to an existing release branch
git cherry-pick <commit>...
```

## Tagging a release

```bash
git switch release/<kobo-version>-icrc
sed -i 's/^version: .*/version: <kobo-version>-icrc.<n>/' Chart.yaml
git commit -am "chore: release <kobo-version>-icrc.<n>"
git tag <kobo-version>-icrc.<n> && git push origin release/<kobo-version>-icrc <kobo-version>-icrc.<n>
```

Then update the tag in the umbrella chart, re-render it and review the baseline diff in `icrc-kobo-toolbox`. Each environment (test, uat, PROD) pins its own tag there.

## Fork build

Defined in phase 3 (umbrella scaffold): how the umbrella references a tag (local path or published package).
