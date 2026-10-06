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
| `icrc-bootstrap` | `kobo/main` | `kobo/main` + fork-only changes (this file, `work/`, `.helmignore`), never proposed upstream. Seeds new release branches. |
| `feature/<name>` | `kobo/main` | One per enhancement (upstream PR pending or rejected). Must bump `Chart.yaml` `version` and add a `CHANGELOG.md` entry (upstream CI). Not based on `icrc-bootstrap`, so the PR carries no fork-only change. |
| `release/<kobo-version>-icrc` | Kobo tag `<kobo-version>` | Integration branch: Kobo release + `icrc-bootstrap` changes + every feature branch. Tagged and deployed through the umbrella chart. One branch per Kobo release. |
| `<kobo-version>-icrc.<n>` tag | release branch | Build of the release branch, e.g. `7.0.0-icrc.1`. Also the chart version. |

There is no separate integration branch: features are tried on the release branch, and only a tag is deployed. `icrc-bootstrap` and `feature/*` carry no tags and are rewritten freely: rebased when `kobo/main` advances, force-pushed. Release branches and tags are frozen:

- A release branch only grows: new commits on top, never rebased nor force-pushed. A new Kobo release gets a new release branch.
- Tags are never moved nor deleted.
- Enforced by GitHub rulesets on `icrc/icrc-kobo-helm-chart`: no force-push or deletion on `release/*`, no update or deletion of `*-icrc.*` tags.
- `7.0.0-icrc.1` is a SemVer prerelease of `7.0.0`: the umbrella chart must pin it exactly, ranges like `~7.0.0` skip it.

## Work tracking

`work/` holds the plan of the chart phases of the ICRC Helm setup (overview in `icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). One file per phase, each with a completion target: `work/next/` before it starts, `work/ongoing/` while running, `work/completed/` once the target is met.

Like this file, `work/` is fork-only: edit it on the current release branch. `icrc-bootstrap` is refreshed from it before a new release branch is created (see [Applying features](#applying-features)).

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
3. Cherry-pick the fix commit onto the current release branch (see [Applying features](#applying-features)), without the version bump. Same for later review changes. Tag a patch to ship it.
4. Merged upstream: delete the branch once a Kobo release includes it. Rejected: keep the branch, it is carried to every release branch.

## Syncing with Kobo

Run whenever `kobo/main` advances.

```bash
git fetch kobo --tags --prune
git switch main && git merge --ff-only kobo/main && git push origin main
```

Then rebase every feature branch that is not merged upstream, and `icrc-bootstrap`. Merged feature branches are left as they are until a Kobo release includes them.

```bash
# Works in bash and zsh
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/; }
for b in $(features) icrc-bootstrap; do git rebase kobo/main "$b" || break; done
git push --force-with-lease origin $(features) icrc-bootstrap
```

On conflict a rebase stops: resolve, `git rebase --continue`, then re-run the commands (already rebased branches are no-ops). A Kobo release changes `Chart.yaml` `version` and `CHANGELOG.md`, so each version bump commit conflicts: re-bump above the new Kobo version.

The release branch is not rebased: it stays on its Kobo release.

## Applying features

Release branches are a Kobo release plus the feature commits, without their version bump commits (`chore: release ...`), which only serve upstream CI.

```bash
# apply <branch>...: cherry-pick the branch commits (kobo/main..<branch>) onto the current branch
apply() { for b; do git cherry-pick $(git rev-list --reverse --invert-grep --grep='^chore: release' kobo/main.."$b") || return; done; }
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/; }

# Add a feature, or new commits of a feature, to the current release branch
git switch release/<kobo-version>-icrc && git cherry-pick <commit>...

# New release branch for a new Kobo release: refresh icrc-bootstrap from the current release branch,
# then apply it and the features not included in that Kobo release
git switch icrc-bootstrap && git checkout release/<current>-icrc -- ICRC.md work/ .helmignore && git commit -m "docs: refresh fork docs"
git switch -c release/<kobo-version>-icrc <kobo-version> && apply icrc-bootstrap feature/<a> feature/<b>
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
