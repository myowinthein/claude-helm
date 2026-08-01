---
title: /helm:env
parent: Commands
nav_order: 10
---

# /helm:env

Audit `.env` files, `.gitignore`, and source code to find and fix environment configuration issues. Covers sync gaps, missing vars, hardcoded values, secret/placeholder correctness, and formatting — all in one pass.

## Flow

```mermaid
flowchart TD
  Start([User runs /helm:env]) --> Branch[Solo Mode: require main/master<br/>GitHub Flow: branch off main]
  Branch --> Scan[Scan all env files,<br/>source code, and .gitignore<br/>11 checks — read-only]

  Scan --> Report[Present findings — ENV AUDIT REPORT]
  Report --> Ask[Ask: which targets to work through?<br/>Env files and/or .gitignore]

  Ask -->|Env files| EnvFlows[Env-file sub-flows, each with a Skip:<br/>Secret cleanup · Placeholder cleanup ·<br/>Env sync · Missing from env ·<br/>Hardcoded values · Never referenced ·<br/>Env formatting]
  Ask -->|.gitignore| GitFlows[.gitignore sub-flows, each with a Skip:<br/>Missing entries · Tracked files ·<br/>Entries to remove · Formatting]
  Ask -->|neither| SkipExit[/Exit: no changes,<br/>clean up branch if one was created/]

  EnvFlows --> Commit[Commit — silent under<br/>git-auto-commit, or confirm]
  GitFlows --> Commit
  Commit --> Push[Ask: push main now?]
  Push --> Promote[Ask: promote to<br/>environment branches?]
  Promote --> Cleanup[GitHub Flow: merge to main,<br/>delete temp branch]
  Cleanup --> Done([Report: ENV COMPLETE])
```

## Steps

### Before starting

Behavior depends on `git-strategy` in CLAUDE.md's Project Config (absence defaults to GitHub Flow, per git.html):

- **Solo Mode**: runs only on `main`/`master`. Halts on any other branch. No branch is created — every commit lands directly on main.
- **GitHub Flow**: records the current branch, then checks out a fresh branch from main's current tip (`chore/env-{date}`). Returns to the original branch at the end (see Step 4).

If the command exits at any point without writing anything, this cleanup still runs: delete the temporary branch and return to the original branch.

### 1. Scan

Eleven read-only checks across all env files, source code, and `.gitignore`. Never stops early — a partial scan produces missed findings that silently persist.

| Check | What it finds |
|---|---|
| 1.1 Detect env files | All `.env*` files and their inferred purpose |
| 1.2 Env sync | Keys present in some files but missing from others |
| 1.3 Source code refs | Every env var accessed in source, per stack |
| 1.4 Cross-reference | Keys missing from all env files; keys never referenced in code |
| 1.5 Secret/placeholder | Real secrets in `.env.example`; unfilled placeholders in other env files |
| 1.6 Hardcoded values | String literals in source that should be env vars |
| 1.7 Env formatting | Duplicate keys, spacing, missing category groupings |
| 1.8 Gitignore missing entries | Stack-appropriate entries absent from `.gitignore` |
| 1.9 Gitignore tracked files | Tracked files that match a `.gitignore` pattern |
| 1.10 Gitignore entries to remove | Overly broad or harmful `.gitignore` entries |
| 1.11 Gitignore formatting | Duplicate entries, missing category groupings |

### 2. Report

Findings presented as a single `ENV AUDIT REPORT` grouped by category. Includes a total issue count and a multi-select to choose which **targets** to work through — **Env files** and/or **.gitignore**. Selecting a target runs every one of its sub-flows in Step 3 (each with its own Skip); selecting neither exits without changes. The report still lists every finding regardless of what's selected.

### 3. Apply fixes

Run the selected target's sub-flows, one at a time, each completing fully before the next. Each sub-flow runs only if it has findings and offers a Skip.

**Env files sub-flows** (if "Env files" selected):

- **Secret cleanup (.env.example)** — shows each flagged real secret with its planned placeholder replacement. Single-select: Replace all, Skip, or Other (free-text). Does not touch any other env file.
- **Placeholder cleanup (other env files)** — shows unfilled placeholders grouped by file. Dev fills in values manually, then marks done.
- **Env sync** — per file. Shows which keys are missing and where they exist in other files. Dev adds them manually per file, marks done before moving to the next.
- **Missing from env** — all at once. Keys referenced in code but absent from all env files, with file:line locations. Dev adds them manually, marks done.
- **Hardcoded values** — shows all findings with severity, file:line, redacted value, and suggested env var name. Single-select: Replace all, Skip, or Other. If Replace all and more than 10 findings: processes one source file at a time, committing after each.
- **Never referenced** — lists keys with no code reference, with a hint on whether each looks infrastructure-only or stale. Dev responds in free text — delete some, keep some, or wire up to code.
- **Env formatting** — single-select: Proceed or Skip. Fixes formatting, removes duplicate keys (keep last), adds category groupings consistently across all env files. Preserves existing group order.

**.gitignore sub-flows** (if ".gitignore" selected):

- **Missing entries** — shows entries with reason, single-select Add all/Skip. If approved, immediately checks for tracked files matching the new patterns.
- **Tracked files** — per-file confirmation to run `git rm --cached`. Triggered automatically after missing entries are added, and for any pre-existing tracked files found in Step 1.9.
- **Entries to remove** — shows overly broad or harmful entries with reason, single-select Remove all/Skip.
- **Gitignore formatting** — single-select Proceed/Skip. Fixes formatting, removes duplicates, adds category groupings.

### 4. Commit, push, and promote

After all fixes are applied, stages what's left (explicit paths, never `git add -A`) and commits — per [git.md's Auto-Commit rule](../rules/git.html#auto-commit): silent if `git-auto-commit: true`, otherwise confirms first. Commit message: `chore(env): audit and fix env configuration`. If nothing was staged (e.g. the Hardcoded values sub-flow already committed everything per-file), there's nothing to commit here — but under GitHub Flow this doesn't automatically mean nothing to do: it checks whether the temporary branch has commits ahead of main (the Hardcoded values sub-flow's own commits, made directly on that branch), and if so still merges that work into main rather than discarding it by deleting the branch. Only when the branch has no commits ahead of main does it skip straight to cleanup. Solo Mode has no such case, since Hardcoded values commits there already land directly on main.

Push always requires its own confirmation regardless of `git-auto-commit`, per git.md's Auto-Commit exception and rules/safety.md's unconditional push-confirmation rule. If environment branches exist (same detection [`/helm:ship`](ship.html) uses), asks which should also receive the update, then merges main into each and pushes.

**Solo Mode** commits directly to main, confirms the push, then runs environment promotion. **GitHub Flow** commits on the temporary branch, merges it into main, confirms the push, runs environment promotion, deletes the temporary branch (locally and remotely if pushed), and returns to whichever branch the command was originally run from.

### 5. Completion report

Summarises the outcome grouped by target (Env files, .gitignore): counts replaced, added, deleted, or fixed for each of the eleven sub-flows — including how many never-referenced keys were deleted vs kept. Lists anything needing manual follow-up: skipped secret cleanup (real secrets still in `.env.example`) and skipped tracked files. Also reports whether the commit was pushed (or is waiting locally for a manual push), which environment branches were promoted, and — under GitHub Flow — the temporary branch's fate and which branch the command returned to.

## Stop conditions

- **No findings at all.** Nothing to fix — the report shows a clean audit.
- **User selects neither target, or Skips every sub-flow.** Clean exit, no changes written.

## See also

- [`/helm:ship`](ship.html) — run env before shipping to catch missing vars before they reach production
- [`/helm:refactor`](refactor.html) — broader code quality pass that pairs with env for a full pre-release sweep
