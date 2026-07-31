---
title: /helm:normalize
parent: Commands
nav_order: 6
---

# /helm:normalize

Rewrites every non-conventional commit message in the repository's history to follow [Conventional Commits](https://www.conventionalcommits.org/) format. Three confirmation gates stand between the user and any destructive action: a plain-language risk warning, a full scan showing exactly how many commits will change and a sample of the rewrites before anything is touched, and a final force-push confirmation before the remote is touched.

## Flow

```mermaid
flowchart TD
  Start([User runs /helm:normalize]) --> Branch{On main<br/>or master?}
  Branch -->|no| BranchStop[/Stop: switch to main first/]
  Branch -->|yes| Warn[Warn: history rewrite,<br/>force push required,<br/>tags orphaned]

  Warn -->|cancel| Cancel[/Exit: no changes/]
  Warn -->|continue| Scan[Collect local branches<br/>Scan all commits<br/>git log --oneline]

  Scan --> Classify[For each commit:<br/>classify compliant vs non-compliant<br/>read diff to infer type + scope]
  Classify --> Count{Non-compliant<br/>count?}

  Count -->|0| NothingToDo[/Exit: all commits already compliant/]
  Count -->|> 0| ShowPlan[Show total, compliant count,<br/>rewrite count, 5 sample rewrites,<br/>local branches affected]

  ShowPlan -->|cancel| Cancel
  ShowPlan -->|confirm| Rewrite[git filter-repo --force --message-callback<br/>rewrite map read from temp file]

  Rewrite --> Tags[Detect orphaned tags<br/>re-create at new SHAs]
  Tags --> PushGate[Ask: force push or skip?]

  PushGate -->|force push| Push[Force push every rewritten<br/>local branch, then tags]
  PushGate -->|skip| Manual[Print manual push commands<br/>per branch, plus tags]

  Push --> Report([Report: rewritten, branches, tags, pushed])
  Manual --> Report
```

## Steps

### Before starting

Only runs from `main` or `master`. Refuses to rewrite history from a feature branch — the rebase would diverge from `main` and create a worse mess.

### 1. Risk warning

Unconditional first gate. Presents a plain-language summary of what history rewriting means:

- Every rewritten commit gets a new SHA — history is permanently altered
- Only local branches are rewritten and force-pushed; anything remote-only (e.g. an environment branch) will diverge unless fetched and checked out first
- Tags pointing at rewritten commits become orphaned (handled in Step 5)
- Anyone else who has cloned the repo will have a broken history

Cancel is equally prominent. The command exits cleanly if the user declines.

### 2. Scan and classify

Runs `git branch --list` to collect every local branch — the only branches this rewrite can affect, since `git filter-repo` rewrites all refs it can see and remote-only branches aren't visible to it. Runs `git log --oneline --no-decorate` to collect every commit from the beginning of the repo. For each commit, checks whether the message already matches the `type(scope): description` format.

For non-compliant commits, reads the diff via `git show {sha} --stat` to infer the correct type and scope:

- **Type** is inferred from what changed: new files → `feat`, broken behavior fixed → `fix`, restructuring with no behavior change → `refactor`, test files only → `test`, docs only → `docs`, dependencies or tooling → `chore`, CI config → `ci`, build config → `build`.
- **Scope** is inferred from the primary module, folder, or domain area affected. Cross-cutting changes use `project` or `core`.
- **Breaking changes** are detected from keywords (`breaking`, `BREAKING`, `!`) in the original message and preserved with a `!` suffix and `BREAKING CHANGE:` footer.

### 3. Show plan and confirm

Second gate, informed by real data. Presents:

- Total commits scanned
- Already-compliant count
- Commits to be rewritten
- Up to 5 sample rewrites showing `'original'` → `'proposed'`
- The local branches that will be rewritten — a reminder to cancel, fetch, and check out any environment or teammate branch that isn't listed but needs the same treatment

The user sees exactly what will change before confirming. Cancel exits cleanly with no changes made.

If the non-compliant count is zero, the command exits here with a "nothing to do" message.

### 4. Rewrite

Uses `git filter-repo --force --message-callback` (preferred) with a JSON rewrite map, falling back to `git filter-branch --msg-filter` if `git filter-repo` is not installed. `--force` is required — filter-repo otherwise refuses to run on anything it doesn't recognize as a fresh clone, and this command runs on the developer's existing working repo. The rewrite map is built during the scan phase and read from a temp file rather than embedded inline, since commit messages can contain quotes, newlines, or unicode that would break a string substituted directly into the callback. Applied in a single pass — no interactive prompts per commit, no partial rewrites. Messages not in the map pass through unchanged.

Verifies the result by sampling the first 20, last 20, and total count of commits before continuing.

### 5. Re-create orphaned tags

Runs regardless of which tool Step 4 used, but does different amounts of work depending on which: `git filter-repo` rewrites all refs it processes, including tags, as part of its normal operation, so this is usually just a confirming pass. The `git filter-branch` fallback has no `--tag-name-filter`, so tags genuinely go orphaned there and this step is load-bearing for that path.

Collects all tags via `git tag`. For each tag, checks whether the original commit SHA still exists in the rewritten history. Orphaned tags (pointing at rewritten SHAs) are deleted and re-created at the corresponding new SHA with the same name and message.

### 6. Force push

Separate third confirmation before touching the remote. Every local branch collected in Step 2 was rewritten, not just the one the command was run from, so each gets its own force push — not only the current branch. If the user skips, prints the exact manual commands to run later, one per branch, plus tags:

```
git push origin {branch} --force   # repeated for every local branch collected in Step 2
git push origin --tags --force
```

### 7. Report

Closes with a structured summary: total commits scanned, rewritten count, which local branches were rewritten, tags re-created, force push status per branch. Any commits Claude could not classify with high confidence are listed separately as uncertain rewrites so the user can review them manually.

## Stop conditions

- **Not on `main` or `master`.** Switch branches and re-run.
- **Risk warning declined.** No changes made.
- **Plan confirmation declined.** No changes made.
- **All commits already compliant.** Nothing to do — exits after the scan.

## Notes

- `git filter-branch` rewrites the full history including merge commits. `git rebase -i` is not used because it requires interactive input per commit and does not handle merge commits cleanly.
- The rewrite map is built from Claude's diff analysis before any git commands run. If the scan is interrupted, nothing has been modified.
- After a successful normalize + force push, `git pull` on any other clone of the repo will fail. Each clone needs a `git fetch --all` followed by `git reset --hard origin/main`.
- Only branches that existed **locally** at rewrite time are covered. An environment branch (staging, production) or any branch that only ever existed on the remote is left pointing at the old history entirely — its ancestry no longer has anything in common with the rewritten main. Fetch and check out any such branch before running normalize if it needs the same treatment; otherwise plan to handle it separately (e.g. delete and recreate it from the new main).

## See also

- [`git.md`](../rules/git.md) — the rule file that enforces Conventional Commits on all future commits
- [`/helm:ship`](ship.md) — the release command that reads commit history to calculate the next version; normalize ensures its version calculation works on repos with messy pre-convention history
- [`/helm:adopt`](adopt.md) — installs the git rules into a project so future commits follow the convention automatically
