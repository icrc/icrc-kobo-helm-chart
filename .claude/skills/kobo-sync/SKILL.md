---
name: kobo-sync
description: Plan a sync of the ICRC Helm chart fork with kobotoolbox/kobo-helm-chart - detect what changed upstream (kobo/main commits, new Kobo release, PR reviews and merges, issue replies, stale branches), write work/ongoing/kobo-sync-<date>.md and create the AzDO PBI and Tasks. Does not run the sync itself. Trigger: /kobo-sync, "sync the fork with Kobo", "plan the Kobo sync".
---

# Plan a sync of the fork with Kobo

Procedure and commands: `ICRC.md` sections "Syncing with Kobo", "Applying features". A sync never tags: tagging is the release procedure (`ICRC.md` "Tagging a release"), run separately. This skill only detects what to do, writes the plan and creates the work items. The sync is then run step by step with the `work-plan` skill ("continue the plan").

## Requirements

- Skills `work-plan` (plan format, running the plan) and `azdo-plan-sync` (AzDO Tasks), installed at user level (`~/.claude/skills/`). If one is missing, stop and tell the user.
- Azure DevOps on-prem MCP tools (`icrc-connectors` plugin), to create the PBI.
- `gh` authenticated with access to `kobotoolbox/kobo-helm-chart` and `icrc/icrc-kobo-helm-chart`; `helm` with the `helm-unittest` plugin.

Git network calls are wrapped in `timeout 60`. If an SSH push or fetch hangs, use HTTPS with `git -c credential.helper='!gh auth git-credential'`.

## 1. Preconditions

- Clean working tree. The plan is written on `icrc-base` (`work/` is fork-only).
- No open `work/ongoing/kobo-sync-*.md`: if one exists, offer to continue it instead.
- `timeout 60 git fetch kobo --tags --prune && timeout 60 git fetch icrc --prune`.
- Local branches equal `icrc`: `git rev-list --left-right --count icrc/<b>...<b>` is `0 0` for `main`, `icrc-base`, `feature/*`, `release/*`. Otherwise stop and report (upstream tracking of these branches points to `kobo/main`, so `git status` is not a reliable check). Exception: a local `feature/*` missing on `icrc` whose PR is merged (GitHub deleted it) is not a mismatch, plan its deletion. If `icrc` has a merge of `main` into a feature branch (GitHub "Update branch"), propose rebasing the local branch instead (`ICRC.md` "Contributing an enhancement").

## 2. Detect

Run read-only, in one batch:

```bash
R=kobotoolbox/kobo-helm-chart
merged() { gh pr list -R $R --state merged --limit 200 --json headRefName,headRepositoryOwner --jq '.[] | select(.headRepositoryOwner.login=="icrc") | .headRefName'; }
features() { git for-each-ref --format='%(refname:short)' refs/heads/feature/ | grep -vxF -f <(merged); }
release() { git for-each-ref --format='%(refname:short)' refs/remotes/icrc/release/ | sed 's|^icrc/||' | sort -V | tail -1; }
kobo_tags() { git tag --list | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$' | sort -V; }

git rev-list --count icrc/main..kobo/main                                                    # A: commits to mirror
kobo_tags | tail -1; release                                                                   # B: new Kobo release without release/<v>-icrc
git log --oneline <latest-tag>..kobo/main                                                      # B: commits past the tag (CI only?)
gh pr list -R $R --state all --limit 200 \
  --json number,title,state,headRefName,headRepositoryOwner,author,comments,reviews,mergeCommit \
  --jq '.[] | select(.headRepositoryOwner.login=="icrc") | .author.login as $a | [.number, .state, .headRefName, ([.comments[] | select(.author.login != $a) | .createdAt] + [.reviews[] | select(.author.login != $a) | .submittedAt] | sort | last // "-"), (.mergeCommit.oid // "-")] | @tsv'   # C
git tag --contains <merge-commit> | grep -E '^[0-9]+\.[0-9]+\.[0-9]+$'                           # C: merged PR released?
me=$(gh api user --jq .login)
gh issue list -R $R --search 'involves:@me' --state all --limit 200 --json number,title,state,comments \
  --jq ".[] | [.number, .state, ([.comments[] | select(.author.login != \"$me\") | .createdAt] | sort | last // \"-\"), .title] | @tsv"   # F: issues opened or commented by us
for b in $(features) icrc-base; do git merge-base --is-ancestor kobo/main "$b" || echo "$b"; done            # D: to rebase
for b in $(features); do git merge-base --is-ancestor "$b" "icrc/$(release)" || echo "$b"; done                  # E: missing from release
git log --oneline "icrc/$(release)..icrc-base" -- . ':!ICRC.md' ':!work' ':!.claude'                       # E: bootstrap chart changes missing from release (docs-only commits ignored)
```

- **C, new activity**: last comment or review by someone other than the PR author, after the date of the previous `work/completed/kobo-sync-*.md` (all activity on the first run). Read it with `gh pr view <n> -R $R --comments` and `gh api repos/$R/pulls/<n>/comments` and summarize each request in one line.
- **C, merged**: delete the branch if it still exists locally or on `icrc`. Released when C returns a Kobo tag; if not released and the current release branch is rebuilt, it merges the PR head commit (`ICRC.md` "Syncing with Kobo").
- **F, new activity**: last comment by someone else after the date of the previous sync, same rule as C. Read it with `gh issue view <n> -R $R --comments`; each reply calling for work (new PR, change of scope) becomes a step. Also update the matching step of the phase plan (`work/ongoing/icrc-helm-phase2-upstream.md`).
- If A to E are all empty and no PR or issue has new activity: report "fork in sync" and stop, no plan.

## 3. Write the plan

Show the sections and steps to the user and confirm before writing `work/ongoing/kobo-sync-<YYYY-MM-DD>.md`. Follow the `work-plan` format (steps `- [ ]`, one `Done when` per section, no em-dash). Omit sections with nothing to do.

```markdown
# Kobo sync <YYYY-MM-DD>

Sync of the fork with `kobo/main` at `<short-sha>` (procedure: `ICRC.md` "Syncing with Kobo"). Previous sync: <link or "none">.

## Mirror Kobo
- [ ] Fast-forward `main` to `kobo/main` (<A> commits) and push to `icrc`
- [ ] Done when: `icrc/main` equals `kobo/main`

## Upstream PRs
- [ ] PR <n> (`feature/<name>`): <one line per review request>; merge the branch again into the release branch
- [ ] PR <n> merged in Kobo (<tag or "not released yet">): delete `feature/<name>` locally and on `icrc`
- [ ] Done when: every review request answered on its PR, merged branches deleted

## Upstream issues
- [ ] Issue <n> (<title>): <reply in one line>; <action>
- [ ] Done when: every reply acted on or recorded in the phase plan

## Rebase on kobo/main
- [ ] Rebase and force-push: `feature/<a>`, ..., `icrc-base` (re-bump `Chart.yaml` above Kobo <version> on conflict)
- [ ] Done when: every branch listed contains `kobo/main` and equals `icrc`

## Release <kobo-version>
- [ ] Create `release/<kobo-version>-icrc` from `icrc-base` on tag `<kobo-version>` with one `--no-ff` merge per feature branch not released upstream: <list>
- [ ] `helm lint --strict -f tests/values/required.yaml .` and `helm unittest .` pass
- [ ] Delete `release/<previous-version>-icrc` locally and on `icrc`
- [ ] Done when: `release/<kobo-version>-icrc` pushed with lint and unit tests passing, previous release branch deleted

## Work items

PBI #<pbi>.

| Plan section | Task |
|---|---|
```

- **Release section**: "Release <kobo-version>" when B finds a new Kobo release (new branch); else "Rebuild release/<v>-icrc" when D or E is non-empty (rebuild, compare with `icrc`, force-push).
- If `kobo/main` has non-CI commits past the release tag, add a step to rebuild from copies rebased onto the tag (`ICRC.md` "Applying features").
- Rebuilding the current release branch: list merged-but-unreleased PRs with their head commit to merge instead of the deleted branch.

## 4. Create the work items

1. Sync PBIs have no parent: syncs are periodic over the project lifetime, not part of a Feature.
2. Create the PBI: title `Sync fork with Kobo <latest-kobo-tag>` (the date is in the PBI metadata), iteration and area taken from the previous sync PBI (`## Work items` line of the latest `work/completed/kobo-sync-*.md`), assigned to the current user (`uniqueName` from `get_me`, never the email), as are its Tasks. Tag the PBI and its Tasks `KOBO`, `KOBO_HELM` and `GH_SYNC` (GitHub-related work); tags are uppercase snake case. Description: `<div>Plan: <code>work/ongoing/kobo-sync-<date>.md</code> in icrc/icrc-kobo-helm-chart (branch <code>icrc-base</code>).</div>`. Confirm before creating (outward-facing).
3. Write the PBI ID in the plan's `## Work items` line, then run the `azdo-plan-sync` skill to create one Task per section and fill the table.

## 5. Commit

Commit the plan on `icrc-base` as `docs(work): plan Kobo sync <YYYY-MM-DD>` and push with `--force-with-lease` (ask first). The release branch picks the plan up at its next rebuild, which is part of the plan.
