---
description: Rewrite non-conventional commit messages across full repo history
---

# normalize

Rewrite all commit messages in this repository to follow Conventional Commits format.

---

## Step 1 — Branch check

Only proceed if on main or master.
If on any other branch, stop and inform user:

"normalize must be run on main or master.
Current branch is {branch}. Please switch and re-run."

---

## Step 2 — Risk warning

Before doing anything else, present this warning using AskUserQuestion:

  AskUserQuestion:
    question: "This command rewrites git history. Understand the consequences before continuing:\n\n- Every rewritten commit gets a new SHA — history is permanently altered\n- Any remote copies (GitHub, GitLab, etc.) require a force push to sync\n- Tags pointing at rewritten commits become orphaned — they will be re-created\n- Anyone else who has cloned this repo will have a broken history\n\nThis is safe for solo developers on private repos with no active collaborators."
    header:   "Risk"
    multiSelect: false
    options:
      - label: "I understand — continue"
        description: "Proceed to scan commit history"
      - label: "Cancel"
        description: "Exit without making any changes"

If Cancel → exit silently.

---

## Step 3 — Scan history

Run: git log --oneline --no-decorate
Collect every commit SHA and message from the beginning of the repo to HEAD.

For each commit, classify as:
- **Compliant** — message already matches `type(scope): description` format
- **Non-compliant** — message does not match

To classify each non-compliant commit:
1. Run: git show {sha} --stat --format="%B"
2. Read the commit message and the file diff summary
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

## Step 4 — Show plan and confirm

Present the plan using AskUserQuestion:

  AskUserQuestion:
    question: "Scan complete.\n\nTotal commits: {total}\nAlready compliant: {compliant_count}\nTo be rewritten: {non_compliant_count}\n\nSample rewrites:\n{show up to 5 examples in format: '{original}' → '{proposed}'}\n\nProceeding will rewrite {non_compliant_count} commits and force-push to remote."
    header:   "Confirm rewrite"
    multiSelect: false
    options:
      - label: "Rewrite {non_compliant_count} commits (Recommended)"
        description: "Apply all rewrites and re-create orphaned tags"
      - label: "Cancel"
        description: "Exit without making any changes"

If Cancel → exit silently.
If non_compliant_count is 0 → inform user "All commits already follow Conventional Commits. Nothing to do." and exit.

---

## Step 5 — Rewrite commit messages

Use git filter-branch to rewrite messages without touching the working tree or file contents:

```
git filter-branch -f --msg-filter '
python3 -c "
import sys, json

rewrite_map = {json_map_here}

msg = sys.stdin.read().strip()
print(rewrite_map.get(msg, msg))
"' -- --all
```

Build the rewrite_map as a JSON object: { "original_message": "conventional_message", ... }
Include only non-compliant commits in the map.
Messages not in the map pass through unchanged.

After filter-branch completes, run: git log --oneline -10
Verify the rewrite applied correctly.

---

## Step 6 — Re-create orphaned tags

Run: git tag
Collect all tags.

For each tag:
1. Run: git rev-list -n 1 {tag} to get the original commit SHA
2. Check if that SHA exists in the new history: git cat-file -t {sha}
3. If the SHA no longer exists (orphaned): find the corresponding new SHA via the rewrite mapping and re-create the tag

Re-create each orphaned tag:
```
git tag -d {tag}
git tag -a {tag} {new_sha} -m "Release {tag}"
```

If no tags are orphaned: skip silently.

---

## Step 7 — Force push

  AskUserQuestion:
    question: "Rewrite complete. Ready to force push to remote?\n\nThis will overwrite the remote history at origin/{branch}. This cannot be undone on the remote."
    header:   "Force push"
    multiSelect: false
    options:
      - label: "Force push to origin (Recommended)"
        description: "Sync the rewritten history to remote"
      - label: "Skip — push manually"
        description: "Leave remote as-is. Run: git push origin {branch} --force --tags"

If force push selected:
```
git push origin {branch} --force
git push origin --tags --force
```

If skip selected: print the manual commands and exit.

---

## Step 8 — Report

─────────────────────────────────
NORMALIZE COMPLETE
─────────────────────────────────
Total commits scanned:   {total}
Already compliant:       {compliant_count}
Rewritten:               {non_compliant_count}
Tags re-created:         {tag_count} (or "none")
Force pushed:            yes / no (manual)
─────────────────────────────────

Sample of rewrites applied:
- '{original}' → '{proposed}'
- '{original}' → '{proposed}'
- ... (up to 5 examples)

If any commits could not be confidently classified, list them:
Uncertain rewrites (review manually):
- {sha} '{original}' → '{proposed}' (reason: {why uncertain})
