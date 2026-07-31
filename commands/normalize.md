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

## Step 2 — Scan history

Run: git branch --list
Collect every local branch — these are the only branches this rewrite will affect (git filter-repo rewrites all refs it can see, and only local ones are visible to it). List them in the Step 3 plan so the developer can fetch and check out any remote-only branch (e.g. an environment branch) that also needs the same treatment, before confirming.

Run: git log --oneline --no-decorate --all
Collect every commit SHA and message reachable from any local branch — `--all` matches the rewrite's actual scope (every local branch, not just the current one); a plain `git log` would only see the current branch's history and miss commits unique to other local branches, even though those commits still get new SHAs from the rewrite regardless. Commits reachable from more than one branch are naturally deduplicated by SHA, not double-counted.

For each commit, classify as:
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

If non_compliant_count is 0 → inform user "All commits already follow Conventional Commits. Nothing to do." and exit. Do not present any prompt — there is nothing to confirm.

Otherwise, present the plan using AskUserQuestion:

  AskUserQuestion:
    question: "Scan complete.\n\nTotal commits: {total}\nAlready compliant: {compliant_count}\nTo be rewritten: {non_compliant_count}\n\nSample rewrites:\n{show up to 5 examples in format: '{original}' → '{proposed}'}\n\nLocal branches that will be rewritten: {list from Step 2}\nIf an environment or teammate branch you need rewritten too isn't listed, cancel, fetch and check it out, then re-run.\n\nProceeding will rewrite {non_compliant_count} commits and force-push every listed branch to remote."
    header:   "Confirm rewrite"
    multiSelect: false
    options:
      - label: "Rewrite {non_compliant_count} commits (Recommended)"
        description: "Apply all rewrites; tags move to their rewritten commits automatically"
      - label: "Cancel"
        description: "Exit without making any changes"

If Cancel → exit silently.

---

## Step 4 — Rewrite commit messages

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

After rewrite completes, verify a representative sample across the full history — use `--all` here too, matching the rewrite's actual scope (every local branch), not just whichever branch is currently checked out:
```
git log --oneline --all | head -20
git log --oneline --all | tail -20
git log --oneline --all | wc -l  # compare against {total} from Step 2 — must match exactly
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

## Step 6 — Force push

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

If skip selected: print the manual commands for every branch (matching the loop above) plus the tags push, and exit.

---

## Step 7 — Report

─────────────────────────────────
NORMALIZE COMPLETE
─────────────────────────────────
Total commits scanned:   {total}
Already compliant:       {compliant_count}
Rewritten:               {non_compliant_count}
Local branches rewritten: {branch_list from Step 2}
Tags verified:           {tag_count} correctly moved (or "none" if no tags exist); {N} needing manual attention, if any
Force pushed:            yes / no (manual) — {branch_list} + tags
─────────────────────────────────

Sample of rewrites applied:
- '{original}' → '{proposed}'
- '{original}' → '{proposed}'
- ... (up to 5 examples)

If any commits could not be confidently classified, list them:
Uncertain rewrites (review manually):
- {sha} '{original}' → '{proposed}' (reason: {why uncertain})
