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
  Start([User runs /helm:env]) --> Branch{On main or master?}
  Branch -->|no| Stop[/Stop: switch to main first/]
  Branch -->|yes| Scan[Scan all env files,<br/>source code, and .gitignore<br/>8 checks — read-only]

  Scan --> Report[Present findings — ENV AUDIT REPORT]
  Report --> Ask[Ask: which targets to work through?<br/>Env files and/or .gitignore]

  Ask -->|Env files| EnvFlows[Env-file sub-flows, each with a Skip:<br/>Secret cleanup · Placeholder cleanup ·<br/>Env sync · Missing from env ·<br/>Hardcoded values · Never referenced ·<br/>Env formatting]
  Ask -->|.gitignore| GitFlows[.gitignore sub-flows, each with a Skip:<br/>Missing entries · Tracked files ·<br/>Entries to remove · Formatting]
  Ask -->|neither| SkipExit[/Exit: no changes/]

  EnvFlows --> Commit[Ask: commit now?]
  GitFlows --> Commit
  Commit --> Done([Report: ENV COMPLETE])
```

## Steps

### 1. Branch check

Only runs from `main` or `master`. Halts on any other branch.

### 2. Scan

Eight read-only checks across all env files, source code, and `.gitignore`. Never stops early — a partial scan produces missed findings that silently persist.

| Check | What it finds |
|---|---|
| 2.1 Detect env files | All `.env*` files and their inferred purpose |
| 2.2 Env sync | Keys present in some files but missing from others |
| 2.3 Source code refs | Every env var accessed in source, per stack |
| 2.4 Cross-reference | Keys missing from all env files; keys never referenced in code |
| 2.5 Secret/placeholder | Real secrets in `.env.example`; unfilled placeholders in other env files |
| 2.6 Hardcoded values | String literals in source that should be env vars |
| 2.7 Env formatting | Duplicate keys, spacing, missing category groupings |
| 2.8 Gitignore | Missing entries, overly broad entries, tracked files matching patterns, formatting |

### 3. Report

Findings presented as a single `ENV AUDIT REPORT` grouped by category. Includes a total issue count and a multi-select to choose which **targets** to work through — **Env files** and/or **.gitignore**. Selecting a target runs every one of its sub-flows in Step 4 (each with its own Skip); selecting neither exits without changes. The report still lists every finding regardless of what's selected.

### 4. Apply fixes

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
- **Tracked files** — per-file confirmation to run `git rm --cached`. Triggered automatically after missing entries are added, and for any pre-existing tracked files found in Step 2.8.
- **Entries to remove** — shows overly broad or harmful entries with reason, single-select Remove all/Skip.
- **Gitignore formatting** — single-select Proceed/Skip. Fixes formatting, removes duplicates, adds category groupings.

### 5. Commit

After all fixes are applied, asks whether to commit. Stages only the files modified during this command — never `git add -A`. Commit message: `chore(env): audit and fix env configuration`.

### 6. Completion report

Summarises the outcome grouped by target (Env files, .gitignore): counts replaced, added, or fixed. Lists skipped tracked files and never-referenced keys under manual action for follow-up.

## Stop conditions

- **Not on `main` or `master`.** Switch back first.
- **No findings at all.** Nothing to fix — the report shows a clean audit.
- **User selects neither target, or Skips every sub-flow.** Clean exit, no changes written.

## See also

- [`/helm:ship`](ship.md) — run env before shipping to catch missing vars before they reach production
- [`/helm:refactor`](refactor.md) — broader code quality pass that pairs with env for a full pre-release sweep
