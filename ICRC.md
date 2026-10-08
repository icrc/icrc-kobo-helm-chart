# ICRC fork

ICRC fork of [kobotoolbox/kobo-helm-chart](https://github.com/kobotoolbox/kobo-helm-chart), consumed by the ICRC umbrella chart in `icrc-kobo-toolbox`. The fork only carries generic enhancements proposed upstream, kept here while their PR is pending or if it is rejected. ICRC-only resources belong to the umbrella chart; a rejected enhancement moves there when it can be done without the fork.

## Remotes

| Remote | URL | Push |
|---|---|---|
| `icrc` | git@github.com:icrc/icrc-kobo-helm-chart.git | yes |
| `kobo` | https://github.com/kobotoolbox/kobo-helm-chart.git | disabled |

The `icrc` organization enforces SAML SSO: the SSH key (or token) must be authorized for it in GitHub settings. Pin the ICRC key to this repository when the machine also holds a personal GitHub key.

```bash
git clone -o icrc git@github.com:icrc/icrc-kobo-helm-chart.git && cd icrc-kobo-helm-chart   # existing clone: git remote rename origin icrc
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
| `icrc-base` | `kobo/main` | `kobo/main` + fork-only changes (this file, `work/`, `.helmignore`), never proposed upstream. Seeds new release branches. |
| `feature/<name>` | `kobo/main` | One per enhancement (upstream PR pending or rejected). Must bump `Chart.yaml` `version` and add a `CHANGELOG.md` entry (upstream CI). Not based on `icrc-base`, so the PR carries no fork-only change. |
| `release/<kobo-version>-icrc` | `icrc-base` on Kobo tag `<kobo-version>` | Integration branch: `icrc-base` + one merge per feature branch. Tagged and deployed through the umbrella chart. Only the latest Kobo release has one: the previous branch is deleted once the next is created. |
| `<kobo-version>-icrc.<n>` tag | release branch | Build of the release branch, e.g. `7.0.0-icrc.1`. Also the chart version. |

There is no separate integration branch: features are tried on the release branch, and only a tag is deployed. Branches are rewritten freely and force-pushed: `icrc-base` and `feature/*` are rebased when `kobo/main` advances, release branches are rebuilt when a feature branch or the chart content of `icrc-base` changes, and before tagging. Docs-only commits on `icrc-base` (`ICRC.md`, `work/`, `.claude/`) do not trigger a rebuild: they are not packaged, and the next rebuild picks them up. Tags are immutable:

- Tags are never moved nor deleted, enforced by the GitHub ruleset `immutable-icrc-tags` on `*-icrc.*`. A tag keeps its commits even once a rebuild drops them from the release branch, or the release branch is deleted.
- `7.0.0-icrc.1` is a SemVer prerelease of `7.0.0`: the umbrella chart must pin it exactly, ranges like `~7.0.0` skip it.

## Work tracking

`work/` holds the plan of the chart phases of the ICRC Helm setup (overview in `icrc-kobo-toolbox`: `work/ongoing/icrc-helm-setup.md`). One file per phase, each with a completion target: `work/next/` before it starts, `work/ongoing/` while running, `work/completed/` once the target is met.

Like this file, `work/` is fork-only: edit it on `icrc-base` and push, no release branch rebuild needed. Its copy on the release branch may lag behind; `icrc-base` is the reference. Commits made directly on a release branch are lost at the next rebuild.

## Tests

[helm-unittest](https://github.com/helm-unittest/helm-unittest) suites live in `tests/`, with fixture values in `tests/values/required.yaml`. They pin the current behaviour of the templates changed by ICRC enhancements; each enhancement adds its own cases.

```bash
helm plugin install https://github.com/helm-unittest/helm-unittest.git --verify=false
helm unittest .
helm lint --strict -f tests/values/required.yaml .
```

## Contributing an enhancement

1. Create `feature/<name>` from `kobo/main`. Commit the fix with its tests (`tests/<suite>_test.yaml`, plus `tests/values/required.yaml` if missing), then a separate commit with the version bump and changelog entry.
2. Push to `icrc`, open the PR against `kobotoolbox/kobo-helm-chart:main`. Apply review changes on the feature branch. To bring the PR up to date with `kobo/main`, rebase and force-push (see [Syncing with Kobo](#syncing-with-kobo)); never use the GitHub "Update branch" button, it merges `kobo/main` into the feature branch and the release branch then gets `kobo/main` commits past its Kobo tag.
3. Merge the branch into the current release branch (see [Applying features](#applying-features)). Merge it again after review changes. Tag a patch to ship it (see [Tagging a release](#tagging-a-release)).
4. Merged upstream: delete the branch. Rejected: keep the branch, it is carried to every release branch.

## Syncing with Kobo

Run whenever `kobo/main` advances. The Claude Code skill `/kobo-sync` (`.claude/skills/kobo-sync/`) detects what to do below, writes the plan `work/ongoing/kobo-sync-<date>.md` and creates its AzDO PBI and Tasks.

```bash
git fetch kobo --tags --prune
git switch main && git merge --ff-only kobo/main && git push icrc main
```

Then check the state of the upstream PRs opened from the fork:

```bash
gh pr list -R kobotoolbox/kobo-helm-chart --state all --limit 200 \
  --json number,state,headRefName,headRepositoryOwner,reviewDecision,comments,reviews,mergeCommit \
  --jq '.[] | select(.headRepositoryOwner.login=="icrc") | [.number,.state,.headRefName,.reviewDecision,(.comments|length),(.reviews|length),.mergeCommit.oid] | @tsv'
```

- `OPEN`: read new comments and reviews, apply requested changes on the feature branch and merge it again into the release branch (see [Contributing an enhancement](#contributing-an-enhancement)).

  ```bash
  gh pr view <n> -R kobotoolbox/kobo-helm-chart --comments
  gh api repos/kobotoolbox/kobo-helm-chart/pulls/<n>/comments --jq '.[] | "\(.path):\(.line) \(.user.login): \(.body)"'   # inline review comments
  ```

- `MERGED`: delete the branch locally, and on `icrc` if GitHub did not. Until a Kobo release tag contains the merge commit, rebuilds of the current release branch merge the PR head commit instead of the branch.

  ```bash
  git branch -D feature/<name> && git push icrc --delete feature/<name>
  git tag --contains <merge-commit> | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$'   # empty: not released yet
  gh pr view <n> -R kobotoolbox/kobo-helm-chart --json headRefOid --jq .headRefOid   # commit to merge while not released
  ```

- `CLOSED` (rejected): keep the branch, it is carried to every release branch. Move the enhancement to the umbrella chart if it can be done without the fork.

Then rebase every feature branch not merged upstream, and `icrc-base`.

```bash
# Works in bash and zsh
merged() { gh pr list -R kobotoolbox/kobo-helm-chart --state merged --limit 200 --json headRefName,headRepositoryOwner --jq '.[] | select(.headRepositoryOwner.login=="icrc") | .headRefName'; }
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/ | grep -vxF -f <(merged); }
for b in $(features) icrc-base; do git rebase kobo/main "$b" || break; done
git push --force-with-lease icrc $(features) icrc-base
```

On conflict a rebase stops: resolve, `git rebase --continue`, then re-run the commands (already rebased branches are no-ops). A Kobo release changes `Chart.yaml` `version` and `CHANGELOG.md`, so each version bump commit conflicts: re-bump above the new Kobo version.

Release branches stay on their Kobo release: they are rebuilt, not rebased on `kobo/main`. Once `release/<new-version>-icrc` is created, delete the previous release branch locally and on `icrc`; its tags remain.

## Applying features

Release branches are `icrc-base` plus one `--no-ff` merge per feature branch, so `git log --merges` and `git branch --merged` show which features are integrated.

```bash
# Add a feature, or review changes of a merged feature, to the release branch
git switch release/<kobo-version>-icrc && git merge --no-ff -m "Merge branch 'feature/<name>' into release/<kobo-version>-icrc" feature/<name>

# Rebuild the release branch (chart content of icrc-base changed, a feature branch was rebased, or before tagging), or create it for a new Kobo release
git switch -C release/<kobo-version>-icrc icrc-base
git merge --no-ff -m "Merge branch 'feature/<a>' into release/<kobo-version>-icrc" feature/<a>   # one per feature branch
git push --force-with-lease icrc release/<kobo-version>-icrc
```

Each merge after the first conflicts on the version bump commit (`chore: release ...`, needed by upstream CI): keep any `Chart.yaml` `version` (it is overwritten when tagging), and keep every `CHANGELOG.md` entry. Compare the rebuilt tree with the previous one (`git diff icrc/release/<kobo-version>-icrc`) before pushing.

A merge also brings the `kobo/main` commits its branch is based on. The release branch must only contain the Kobo release, so this works while `kobo/main` has no commits past the release tag other than CI. Otherwise, rebuild from a copy of `icrc-base` and of each feature branch rebased onto the tag (`git rebase --onto <kobo-version> kobo/main <copy>`).

## Tagging a release

Tagging is the release procedure, separate from [Syncing with Kobo](#syncing-with-kobo): a sync only rebuilds release branches. No tag until the umbrella chart is released; until then, tests use the release branch as reference.

If `git log --oneline release/<kobo-version>-icrc..icrc-base` is not empty, rebuild the release branch first (see [Applying features](#applying-features)).

```bash
git switch release/<kobo-version>-icrc
sed -i 's/^version: .*/version: <kobo-version>-icrc.<n>/' Chart.yaml
git commit -am "chore: release <kobo-version>-icrc.<n>"
git tag -s <kobo-version>-icrc.<n> -m "<kobo-version>-icrc.<n>" && git tag -v <kobo-version>-icrc.<n>
git push icrc release/<kobo-version>-icrc <kobo-version>-icrc.<n>
```

Then update the tag in the umbrella chart, re-render it and review the baseline diff in `icrc-kobo-toolbox`. Each environment (test, uat, PROD) pins its own tag there.

## Fork build

Defined in phase 3 (umbrella scaffold): how the umbrella references a tag (local path or published package).
