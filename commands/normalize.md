---
description: Rewrite non-conventional commit messages across full repo history
---

# normalize

Rewrite all commit messages in this repository to follow Conventional Commits format.

---

## Before starting

No branch requirement — run from any branch. `git filter-repo` (and the `git filter-branch` fallback, invoked with `--all`) rewrite every local branch together in one consistent pass regardless of which one is checked out, so the starting branch has no effect on the result. See Step 2 and Step 3 for the actual scope (every local branch) and Step 1's risk warning for what that means.

---

## Step 1 — Risk warning

Before doing anything else, present this warning using AskUserQuestion:

  AskUserQuestion:
    question: "This command rewrites git history. Understand the consequences before continuing:\n\n- Every rewritten commit gets a new SHA — history is permanently altered\n- Only local branches are rewritten and force-pushed; anything remote-only (e.g. an environment branch) will diverge unless fetched and checked out first\n- Tags move to their rewritten commits automatically — verified, not manually reconstructed\n- Anyone else who has cloned this repo will have a broken history\n\nThis is safe for solo developers on private repos with no active collaborators."
    header:   "Risk"
    multiSelect: false
    options:
      - label: "I understand — continue"
        description: "Proceed to scan commit history"
      - label: "Cancel"
        description: "Exit without making any changes"

If Cancel → exit silently.

---

## Step 2 — Load ledger and scan history

### 2.1 Load the ledger

Look for `.claude/helm/normalize-log.json`. This is the command's memory across runs — it lets each run scan only commits added since the last check, instead of the full history every time. This plugin's ledgers live under `.claude/helm/` rather than flat in `.claude/`, to avoid colliding with another plugin's own files of the same generic name.

Schema:
```json
{
  "schema_version": 1,
  "branches": {
    "main": {
      "last_checked_commit": "abc1234",
      "last_checked_date": "2026-08-01"
    }
  }
}
```

`schema_version` — bumped only if this ledger's structure changes in a future release. Lets a future version of this command detect an older-shaped file and handle it explicitly instead of misreading it. Treat a missing `schema_version` as `1` (every ledger written before this field existed). Always write it going forward.

`branches` — one entry per local branch, keyed by branch name. `last_checked_commit` is that branch's tip SHA as of the end of the most recent run that updated the ledger (see Step 6 for when that is) — note this is the branch's **post-rewrite** SHA whenever that run performed a rewrite, since `git filter-repo`/`filter-branch` change every commit's SHA from the first rewritten commit onward, not just the ones whose message actually changed.

If the file does not exist, this is the first run. Inform the user: "No normalize history found — scanning full history." Proceed with an empty `branches` map — equivalent to no exclusions in 2.3 below.

### 2.2 Collect branches

Run: git branch --list
Collect every local branch — these are the only branches this rewrite will affect (git filter-repo rewrites all refs it can see, and only local ones are visible to it). List them in the Step 3 plan so the developer can fetch and check out any remote-only branch (e.g. an environment branch) that also needs the same treatment, before confirming.

Diff the current branch list against the ledger's `branches` keys:
- **In both**: scan incrementally in 2.3.
- **In the current list only** (no ledger entry): a new branch — scan its full reachable history. Shared history with an already-tracked branch is still excluded naturally, since that branch's `last_checked_commit` is one of the exclusions in 2.3.
- **In the ledger only** (no longer a local branch): dropped. Do not carry its entry forward when the ledger is next written (Step 6).

### 2.3 Scan history

For each `branches` entry carried forward from 2.2, validate its `last_checked_commit` still exists in the repo: `git cat-file -e {sha}^{commit}`. If it doesn't (e.g. the repo was reset, or the ledger is stale in some other way), drop it from the exclusion list below and scan that branch's full history instead, same as a new branch — note it in the Step 8 report as re-scanned in full, with the reason.

Run one combined query — the positive refs are every current local branch, the negative refs are every still-valid `last_checked_commit` surviving the checks above:
```
git log --oneline --no-decorate {every local branch} --not {every valid last_checked_commit}
```
This returns exactly the commits reachable from any current branch but not already covered by a previous run — automatically deduplicated (a commit reachable from two branches only appears once), and correctly scoped for both new branches (nothing excludes their history unless it overlaps an already-tracked branch) and existing branches (only new commits since their own last check). If the ledger is empty (first run, or every entry was dropped above), this is equivalent to `git log --oneline --no-decorate --all` — the full history, matching today's behavior.

For each commit returned, classify as:
- **Compliant** — message already matches `type(scope): description` format
- **Non-compliant** — message does not match

To classify each non-compliant commit:
1. Run: git show {sha} --stat --format="%B"
2. Read the commit message and the file diff summary. Record the raw message text exactly as `%B` printed it, before any trimming — the exact normalization applied here must match Step 4's lookup normalization (`.strip()`), see below.
3. Infer the correct Conventional Commits type and scope from the diff

Use this inference logic:

**Type inference (from diff):**
- New files added, new functionality → feat
- Existing files changed to fix broken behavior → fix
- Files restructured or renamed with no behavior change → refactor
- Test files added or modified → test
- Documentation files only (.md, comments) → docs
- package.json, composer.json, lock files, dependencies → chore
- CI/CD config files (.github/, .gitlab-ci.yml, etc.) → ci
- Build config (webpack, vite, makefile, etc.) → build
- Whitespace, formatting, style only → style
- Performance optimization with benchmarks → perf

**Scope inference (from diff):**
- Use the primary module, folder, or domain area affected
- If changes span multiple areas, use the dominant one
- If truly cross-cutting, use "project" or "core"
- Keep it lowercase, one word or hyphenated

**Breaking change detection:**
- If the original message contains "breaking", "BREAKING", or "!" → flag as breaking change
- Add `!` to the type and append `BREAKING CHANGE: {description}` footer

Build a rewrite plan: a list of {sha, original_message, proposed_message} for every non-compliant commit.

---

## Step 3 — Show plan and confirm

If non_compliant_count is 0 → inform user "All commits already follow Conventional Commits. Nothing to do." Do not present any prompt — there is nothing to confirm. The scan itself still found the current state accurately, so proceed to Step 6 to update the ledger (safe to advance the watermark — nothing was left pending), then Step 8 to report.

Otherwise, present the plan using AskUserQuestion:

  AskUserQuestion:
    question: "Scan complete.\n\nCommits scanned this run: {total}\nAlready compliant: {compliant_count}\nTo be rewritten: {non_compliant_count}\n\nSample rewrites:\n{show up to 5 examples in format: '{original}' → '{proposed}'}\n\nLocal branches that will be rewritten: {list from Step 2}\nIf an environment or teammate branch you need rewritten too isn't listed, cancel, fetch and check it out, then re-run.\n\nProceeding will rewrite {non_compliant_count} commits and force-push every listed branch to remote."
    header:   "Confirm rewrite"
    multiSelect: false
    options:
      - label: "Rewrite {non_compliant_count} commits (Recommended)"
        description: "Apply all rewrites; tags move to their rewritten commits automatically"
      - label: "Cancel"
        description: "Exit without making any changes"

If Cancel → exit silently. The ledger is not updated — these non-compliant commits are found again on the next run rather than silently skipped by an advanced watermark.

---

## Step 4 — Rewrite commit messages

Before doing anything else, capture the full history's current commit count for the post-rewrite integrity check below — this is independent of `{total}` from Step 2, which only reflects this run's incrementally-scanned commits, not the whole repo:
```
git log --oneline --all | wc -l
```

Also capture the `origin` remote's URL, if one exists — `git filter-repo` removes it by default after a rewrite (a safety measure that assumes it's operating on a disposable clone), and Step 7's force push needs it back:
```
git remote get-url origin
```

Build the rewrite map as a JSON object: `{ "original_message": "conventional_message", ... }`
Include only non-compliant commits. Messages not in the map pass through unchanged.
**Every key must be `.strip()`-equivalent** (leading/trailing whitespace and newlines removed) — the callbacks below strip the incoming message the same way before looking it up. If the map's keys aren't normalized identically, lookups silently miss and the commit passes through unrewritten with no error.

Write the rewrite map to an actual temp file (e.g. `/tmp/normalize-rewrite-map.json`) — do not embed the JSON inline in a shell-quoted script. Commit messages can contain quotes, newlines, or unicode that would break a string substituted directly into the callback; reading from a file avoids that entirely. Delete the temp file once the rewrite below completes.

**Preferred: git filter-repo** (immutable, no backup refs left behind)

First check if available:
```
git filter-repo --version
```

If available, use `--force` — filter-repo refuses to run on anything it doesn't recognize as a fresh clone, and this command runs on the developer's existing working repo, not a throwaway clone:
```
git filter-repo --force --message-callback '
import json
with open("/tmp/normalize-rewrite-map.json") as f:
    rewrite_map = json.load(f)
decoded = message.decode("utf-8").strip()
return rewrite_map.get(decoded, decoded).encode("utf-8")
'
```

**Fallback: git filter-branch** (deprecated since Git 2.36, but still functional)

Only use if `git filter-repo` is not installed. Includes `--tag-name-filter cat` — without it, filter-branch leaves every tag pointing at its old, pre-rewrite commit instead of moving it to the rewritten one, since (unlike filter-repo) it doesn't touch tags by default:
```
git filter-branch -f --msg-filter '
python3 -c "
import json
with open(\"/tmp/normalize-rewrite-map.json\") as f:
    rewrite_map = json.load(f)
import sys
msg = sys.stdin.read().strip()
print(rewrite_map.get(msg, msg))
"' --tag-name-filter cat -- --all
```

After rewrite completes, restore `origin` if `git filter-repo` removed it — idempotent, safe to run regardless of which rewrite path was used or whether a remote existed in the first place:
```
git remote get-url origin >/dev/null 2>&1 || { [ -n "{origin_url}" ] && git remote add origin {origin_url}; }
```

Then verify a representative sample across the full history — use `--all` here too, matching the rewrite's actual scope (every local branch), not just whichever branch is currently checked out:
```
git log --oneline --all | head -20
git log --oneline --all | tail -20
git log --oneline --all | wc -l  # compare against the full-history count captured at the start of this step — must match exactly
```

---

## Step 5 — Verify tags

Both `git filter-repo` (by default) and the `git filter-branch` fallback (via `--tag-name-filter cat` in Step 4) move tags to their rewritten commits automatically as part of the rewrite itself — this step confirms that worked, rather than manually detecting and reconstructing orphaned tags.

Run: git tag
Collect all tags.

For each tag, confirm it resolves to a commit reachable from at least one local branch collected in Step 2:
```
git merge-base --is-ancestor {tag} {branch}
```
Check against each local branch until one succeeds (exit code 0). If none succeed for a given tag, the rewrite did not correctly move it — report it to the user as needing manual attention rather than guessing at a fix.

If every tag resolves correctly: skip silently.

---

## Step 6 — Update ledger

Only reached when the ledger should actually advance: Step 3's "nothing to do" exit routes here directly (the scan found nothing pending, so it's safe to record), and completing Step 4's rewrite through Step 5 also routes here. Cancelling in Step 1 or Step 3 exits before this step — the ledger is left untouched in both cases (see those steps for why).

For every current local branch (from Step 2.2, including new ones — excluding any dropped in 2.2): capture its current tip SHA:
```
git rev-parse {branch}
```
If Step 4 ran, this is the post-rewrite tip; if Step 3 exited on "all compliant" without a rewrite, it's simply the branch's unchanged tip.

Set `schema_version` to `1` if not already present, and set `branches` to exactly the current local branch list from 2.2 — this naturally drops entries for branches removed since the last run and adds entries for new ones, each with `last_checked_commit` set to the SHA just captured and `last_checked_date` to today.

Write it to `.claude/helm/normalize-log.json` (creating `.claude/helm/` if it doesn't exist yet), then commit:
```
git add .claude/helm/normalize-log.json
git commit -m "chore(normalize): update ledger after {scan / rewrite}"
```
This commit lands on whichever branch is currently checked out — it travels with that branch's force push in Step 7 like any other commit made during this run.

---

## Step 7 — Force push

Every local branch collected in Step 2 was rewritten, not just the one this command was run from — each needs pushing to stay in sync with its remote copy.

  AskUserQuestion:
    question: "Rewrite complete. Ready to force push {branch_count} rewritten branches ({branch_list}) and tags to remote?\n\nThis will overwrite their remote history. This cannot be undone on the remote."
    header:   "Force push"
    multiSelect: false
    options:
      - label: "Force push to origin (Recommended)"
        description: "Sync every rewritten branch and tag to remote"
      - label: "Skip — push manually"
        description: "Leave remote as-is. Run the commands below yourself"

If force push selected, for each local branch collected in Step 2:
```
git push origin {branch} --force
```
Then:
```
git push origin --tags --force
```

If skip selected: print the manual commands for every branch (matching the loop above) plus the tags push, then proceed to Step 8 for the report.

---

## Step 8 — Report

─────────────────────────────────
NORMALIZE COMPLETE
─────────────────────────────────
Commits scanned this run: {total} (incremental — see ledger below; "full history" if this was the first run)
Already compliant:       {compliant_count}
Rewritten:               {non_compliant_count}
Local branches rewritten: {branch_list from Step 2}
Tags verified:           {tag_count} correctly moved (or "none" if no tags exist); {N} needing manual attention, if any
Force pushed:            yes / no (manual) — {branch_list} + tags
Ledger:                  {N} branches tracked, {N} added, {N} dropped, {N} re-scanned in full (stale or missing last_checked_commit)
─────────────────────────────────

Other clones must run, per rewritten branch, before their next pull:
  git fetch --all && git reset --hard origin/{branch}
`git pull` will fail or silently diverge otherwise.

Sample of rewrites applied:
- '{original}' → '{proposed}'
- '{original}' → '{proposed}'
- ... (up to 5 examples)

If any commits could not be confidently classified, list them:
Uncertain rewrites (review manually):
- {sha} '{original}' → '{proposed}' (reason: {why uncertain})
