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
  Start([User runs /helm:normalize]) --> Warn[Warn: history rewrite,<br/>force push required]

  Warn -->|cancel| CancelEarly[/Exit: no changes,<br/>ledger untouched/]
  Warn -->|continue| Ledger[Load .claude/normalize-log.json<br/>diff branch list against it:<br/>new · dropped · carried forward]

  Ledger --> Scan[git log {branches} --not {last_checked_commit}s<br/>incremental — full history on first run<br/>or if a stored commit no longer exists]

  Scan --> Classify[For each commit:<br/>classify compliant vs non-compliant<br/>read diff to infer type + scope]
  Classify --> Count{Non-compliant<br/>count?}

  Count -->|0| NothingToDo[Nothing to do this run]
  Count -->|> 0| ShowPlan[Show total, compliant count,<br/>rewrite count, 5 sample rewrites,<br/>local branches affected]

  ShowPlan -->|cancel| CancelLate[/Exit: no changes,<br/>ledger untouched/]
  ShowPlan -->|confirm| Rewrite[git filter-repo --force --message-callback<br/>rewrite map read from temp file]

  Rewrite --> Tags[Verify tags moved<br/>git merge-base --is-ancestor]
  Tags --> UpdateLedger[Capture new tip SHA per branch<br/>write + commit the ledger]
  NothingToDo --> UpdateLedger

  UpdateLedger --> PushGate[Ask: force push or skip?]

  PushGate -->|force push| Push[Force push every rewritten<br/>local branch, then tags]
  PushGate -->|skip| Manual[Print manual push commands<br/>per branch, plus tags]

  Push --> Report([Report: rewritten, branches, tags, pushed, ledger])
  Manual --> Report
```

## Steps

### Before starting

No branch requirement — runs from any branch. `git filter-repo` (and the `git filter-branch` fallback, invoked with `--all`) rewrite every local branch together in one consistent pass regardless of which one is checked out, so the starting branch has no effect on the result — see Step 2 for the actual scope.

### 1. Risk warning

Unconditional first gate. Presents a plain-language summary of what history rewriting means:

- Every rewritten commit gets a new SHA — history is permanently altered
- Only local branches are rewritten and force-pushed; anything remote-only (e.g. an environment branch) will diverge unless fetched and checked out first
- Tags move to their rewritten commits automatically — verified, not manually reconstructed (Step 5)
- Anyone else who has cloned the repo will have a broken history

Cancel is equally prominent. The command exits cleanly if the user declines.

### 2. Load ledger, scan, and classify

Reads `.claude/normalize-log.json` if it exists — the command's memory across runs, so each run only has to scan commits added since the last check instead of the full history every time. First run (no file): informs the user and scans everything, same as before this ledger existed.

The ledger stores one entry per local branch: `last_checked_commit` (that branch's tip SHA as of the end of the last run that updated the ledger) and `last_checked_date`. Because `git filter-repo`/`filter-branch` change every commit's SHA from the first rewritten commit onward — not just the ones whose message actually changed — `last_checked_commit` is always the branch's **post-rewrite** tip whenever the run that recorded it performed a rewrite.

Runs `git branch --list` to collect every local branch — the only branches this rewrite can affect, since `git filter-repo` rewrites all refs it can see and remote-only branches aren't visible to it. Diffs this list against the ledger's branches: a branch in both scans incrementally; a branch with no ledger entry is new and gets its full history scanned (shared history with an already-tracked branch is still excluded automatically); a ledger entry with no matching local branch is dropped and not carried forward.

Before excluding a branch's `last_checked_commit`, validates it still exists in the repo (`git cat-file -e {sha}^{commit}`) — a reset or otherwise-stale ledger entry is dropped and that branch is scanned in full instead, reported at the end as re-scanned. The actual scan is one combined query: `git log --oneline --no-decorate {every local branch} --not {every valid last_checked_commit}` — commits reachable from any current branch, minus everything already covered by a previous run. This is exactly `git log --oneline --no-decorate --all` (today's full-history behavior) when the ledger is empty, and automatically deduplicates a commit reachable from more than one branch. For each commit returned, checks whether the message already matches the `type(scope): description` format.

For non-compliant commits, reads the diff via `git show {sha} --stat` to infer the correct type and scope:

- **Type** is inferred from what changed: new files → `feat`, broken behavior fixed → `fix`, restructuring with no behavior change → `refactor`, test files only → `test`, docs only → `docs`, dependencies or tooling → `chore`, CI config → `ci`, build config → `build`.
- **Scope** is inferred from the primary module, folder, or domain area affected. Cross-cutting changes use `project` or `core`.
- **Breaking changes** are detected from keywords (`breaking`, `BREAKING`, `!`) in the original message and preserved with a `!` suffix and `BREAKING CHANGE:` footer.

### 3. Show plan and confirm

If the non-compliant count is zero, informs the user "nothing to do" — no prompt shown, since there'd be nothing to confirm — and still proceeds to Step 6 to advance the ledger: the scan found the accurate current state, so it's safe to record that this range is checked.

Otherwise, this second gate, informed by real data, presents:

- Commits scanned this run (incremental, not the full repo — see the ledger note in Step 2)
- Already-compliant count
- Commits to be rewritten
- Up to 5 sample rewrites showing `'original'` → `'proposed'`
- The local branches that will be rewritten — a reminder to cancel, fetch, and check out any environment or teammate branch that isn't listed but needs the same treatment

The user sees exactly what will change before confirming. Cancel exits cleanly with no changes made — including no ledger update, so these same non-compliant commits are found again next run rather than silently skipped by an advanced watermark.

### 4. Rewrite

Uses `git filter-repo --force --message-callback` (preferred) with a JSON rewrite map, falling back to `git filter-branch --msg-filter --tag-name-filter cat -- --all` if `git filter-repo` is not installed. `--force` is required — filter-repo otherwise refuses to run on anything it doesn't recognize as a fresh clone, and this command runs on the developer's existing working repo. `--tag-name-filter cat` is required on the fallback path — unlike filter-repo, filter-branch doesn't touch tags by default, so without it every tag is left pointing at its old, pre-rewrite commit. The rewrite map is built during the scan phase and read from a temp file rather than embedded inline, since commit messages can contain quotes, newlines, or unicode that would break a string substituted directly into the callback. Applied in a single pass — no interactive prompts per commit, no partial rewrites. Messages not in the map pass through unchanged.

Both callbacks strip the incoming message before looking it up, so the map's keys are built with the same `.strip()`-equivalent normalization when captured in Step 2 — otherwise a mismatched key silently misses the lookup and that commit passes through unrewritten with no error.

Captures the full-history commit count (`git log --all | wc -l`) before touching anything — separate from Step 2's incrementally-scanned `{total}`, since the rewrite itself still operates on the entire history via `--all` regardless of how much of it this run actually scanned. Verifies the result across every local branch afterward (`git log --all`, matching the rewrite's actual scope) by sampling the first 20, last 20, and comparing the count against that captured baseline before continuing.

### 5. Verify tags

Both `git filter-repo` (by default) and the `git filter-branch` fallback (via `--tag-name-filter cat` in Step 4) move tags to their rewritten commits automatically as part of the rewrite itself, so this step confirms that worked rather than manually detecting and reconstructing orphaned tags.

Collects all tags via `git tag`. For each tag, confirms it resolves to a commit reachable from at least one local branch collected in Step 2, via `git merge-base --is-ancestor {tag} {branch}` checked against each branch until one succeeds. A tag that doesn't resolve against any branch means the rewrite didn't correctly move it — reported to the user as needing manual attention rather than guessed at.

### 6. Update ledger

Reached whenever the ledger should actually advance: Step 3's "nothing to do" exit routes straight here, and a completed rewrite (Steps 4–5) also routes here. Cancelling in Step 1 or Step 3 skips this step entirely — see those steps for why.

Captures each current local branch's tip SHA (`git rev-parse {branch}`) — the post-rewrite tip if Step 4 ran, or the branch's unchanged tip if Step 3 exited on "all compliant." Writes `.claude/normalize-log.json` with `branches` set to exactly the current local branch list — new branches get an entry, branches removed since the last run are dropped, matching Step 2's diff. Commits the ledger (`chore(normalize): update ledger after {scan / rewrite}`) on whichever branch is currently checked out, so it travels with that branch's push in Step 7 like any other commit made this run.

### 7. Force push

Separate third confirmation before touching the remote. Every local branch collected in Step 2 was rewritten, not just the one the command was run from, so each gets its own force push — not only the current branch. If the user skips, prints the exact manual commands to run later, one per branch, plus tags, then still proceeds to Step 8 for the report:

```
git push origin {branch} --force   # repeated for every local branch collected in Step 2
git push origin --tags --force
```

### 8. Report

Closes with a structured summary: commits scanned this run, rewritten count, which local branches were rewritten, tags verified (plus any needing manual attention), force push status per branch, and the ledger's branch counts (tracked, added, dropped, re-scanned in full). Runs regardless of whether the push in Step 7 was automatic or manual/skipped.

Always includes the commands needed to sync any other clone of the repo — `git pull` will fail or silently diverge after a force-pushed rewrite, so every other clone needs a `git fetch --all && git reset --hard origin/{branch}` per rewritten branch.

Any commits Claude could not classify with high confidence are listed separately as uncertain rewrites so the user can review them manually.

## Stop conditions

- **Risk warning declined.** No changes made, ledger untouched.
- **Plan confirmation declined.** No changes made, ledger untouched — these non-compliant commits are found again next run.
- **All commits already compliant.** Nothing to rewrite, but the ledger still advances (see Step 6) so this range isn't rescanned next time.

## Notes

- `git filter-branch` rewrites the full history including merge commits. `git rebase -i` is not used because it requires interactive input per commit and does not handle merge commits cleanly.
- The rewrite map is built from Claude's diff analysis before any git commands run. If the scan is interrupted, nothing has been modified.
- `.claude/normalize-log.json` tracks each local branch's last-checked commit so re-runs only scan new commits, not the entire history every time. Because a rewrite changes every commit's SHA from the first rewritten one onward, this value is always the branch's tip *after* the most recent rewrite, not before.
- After a successful normalize + force push, `git pull` on any other clone of the repo will fail. Each clone needs a `git fetch --all` followed by `git reset --hard origin/main`.
- Only branches that existed **locally** at rewrite time are covered. An environment branch (staging, production) or any branch that only ever existed on the remote is left pointing at the old history entirely — its ancestry no longer has anything in common with the rewritten main. Fetch and check out any such branch before running normalize if it needs the same treatment; otherwise plan to handle it separately (e.g. delete and recreate it from the new main).

## See also

- [`git.md`](../rules/git.html) — the rule file that enforces Conventional Commits on all future commits
- [`/helm:ship`](ship.html) — the release command that reads commit history to calculate the next version; normalize ensures its version calculation works on repos with messy pre-convention history
- [`/helm:adopt`](adopt.html) — installs the git rules into a project so future commits follow the convention automatically
