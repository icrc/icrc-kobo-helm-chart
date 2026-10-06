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

### Signing

Every commit and tag made on the fork is signed with the ICRC SSH key. Add the public key on GitHub a second time as a **Signing key** (Settings > SSH and GPG keys) so commits show as Verified.

```bash
git config gpg.format ssh
git config user.signingkey ~/.ssh/<icrc-key>.pub
git config commit.gpgsign true
git config tag.gpgsign true
# Local verification: git log --show-signature, git tag -v <tag>
echo "$(git config user.email) $(cat ~/.ssh/<icrc-key>.pub)" > .git/allowed_signers
git config gpg.ssh.allowedSignersFile "$PWD/.git/allowed_signers"
```

The ruleset `immutable-icrc-tags` also requires the tagged commit to be signed. Upstream Kobo commits are not all signed, so signatures are not enforced on branches.

## Branches and tags

| Ref | Base | Role |
|---|---|---|
| `kobo/main` | - | Kobo reference (remote-tracking, read only). |
| `main` | `kobo/main` | Mirror of Kobo on the fork, never committed to. Pushing it triggers `publish-chart` if Actions are enabled on the fork. |
| `icrc-bootstrap` | `kobo/main` | `kobo/main` + fork-only changes (this file, `work/`, `.helmignore`), never proposed upstream. Seeds new release branches. |
| `feature/<name>` | `kobo/main` | One per enhancement (upstream PR pending or rejected). Must bump `Chart.yaml` `version` and add a `CHANGELOG.md` entry (upstream CI). Not based on `icrc-bootstrap`, so the PR carries no fork-only change. |
| `release/<kobo-version>-icrc` | `icrc-bootstrap` on Kobo tag `<kobo-version>` | Integration branch: `icrc-bootstrap` + one merge per feature branch. Tagged and deployed through the umbrella chart. One branch per Kobo release. |
| `<kobo-version>-icrc.<n>` tag | release branch | Build of the release branch, e.g. `7.0.0-icrc.1`. Also the chart version. |

There is no separate integration branch: features are tried on the release branch, and only a tag is deployed. Branches are rewritten freely and force-pushed: `icrc-bootstrap` and `feature/*` are rebased when `kobo/main` advances, release branches are rebuilt when `icrc-bootstrap` changes. Tags are immutable:

- Tags are never moved nor deleted, enforced by the GitHub ruleset `immutable-icrc-tags` on `*-icrc.*`. A tag keeps its commits even once a rebuild drops them from the release branch.
- `7.0.0-icrc.1` is a SemVer prerelease of `7.0.0`: the umbrella chart must pin it exactly, ranges like `~7.0.0` skip it.

## Work tracking

`work/` holds the plan of the chart phases of the ICRC Helm setup (overview in `icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). One file per phase, each with a completion target: `work/next/` before it starts, `work/ongoing/` while running, `work/completed/` once the target is met.

Like this file, `work/` is fork-only: edit it on `icrc-bootstrap`, then rebuild the release branch (see [Applying features](#applying-features)). Commits made directly on a release branch are lost at the next rebuild.

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
3. Merge the branch into the current release branch (see [Applying features](#applying-features)). Merge it again after review changes. Tag a patch to ship it.
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

Release branches stay on their Kobo release: they are rebuilt, not rebased on `kobo/main`.

## Applying features

Release branches are `icrc-bootstrap` plus one `--no-ff` merge per feature branch, so `git log --merges` and `git branch --merged` show which features are integrated.

```bash
# Add a feature, or review changes of a merged feature, to the release branch
git switch release/<kobo-version>-icrc && git merge --no-ff -m "Merge branch 'feature/<name>' into release/<kobo-version>-icrc" feature/<name>

# Rebuild the release branch (icrc-bootstrap changed, or a feature branch was rebased), or create it for a new Kobo release
git switch -C release/<kobo-version>-icrc icrc-bootstrap
git merge --no-ff -m "Merge branch 'feature/<a>' into release/<kobo-version>-icrc" feature/<a>   # one per feature branch
git push --force-with-lease origin release/<kobo-version>-icrc
```

Each merge after the first conflicts on the version bump commit (`chore: release ...`, needed by upstream CI): keep any `Chart.yaml` `version` (it is overwritten when tagging), and keep every `CHANGELOG.md` entry. Compare the rebuilt tree with the previous one (`git diff origin/release/<kobo-version>-icrc`) before pushing.

A merge also brings the `kobo/main` commits its branch is based on. The release branch must only contain the Kobo release, so this works while `kobo/main` has no commits past the release tag other than CI. Otherwise, rebuild from a copy of `icrc-bootstrap` and of each feature branch rebased onto the tag (`git rebase --onto <kobo-version> kobo/main <copy>`).

## Tagging a release

```bash
git switch release/<kobo-version>-icrc
sed -i 's/^version: .*/version: <kobo-version>-icrc.<n>/' Chart.yaml
git commit -am "chore: release <kobo-version>-icrc.<n>"
git tag -s <kobo-version>-icrc.<n> -m "<kobo-version>-icrc.<n>" && git tag -v <kobo-version>-icrc.<n>
git push origin release/<kobo-version>-icrc <kobo-version>-icrc.<n>
```

Then update the tag in the umbrella chart, re-render it and review the baseline diff in `icrc-kobo-toolbox`. Each environment (test, uat, PROD) pins its own tag there.

## Fork build

Defined in phase 3 (umbrella scaffold): how the umbrella references a tag (local path or published package).
