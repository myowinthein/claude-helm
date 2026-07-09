---
title: Rules
nav_order: 3
has_children: true
permalink: /docs/rules
---

# Rules

Rule files installed into projects via [`/helm:adopt`](/docs/commands/adopt). They govern git workflow and operational safety across all projects that use helm.

## Rule files

- [`git.md`](git) — git workflow rules: Solo Mode, GitHub Flow, branch naming, Conventional Commits, code quality gates, and environment branch promotion. Activate features by declaring them in your project's `CLAUDE.md` (`git-solo: true`, `git-auto-commit: true`).
- [`safety.md`](safety) — operational safety rules: what to scan on bootstrap, what never to touch without explicit confirmation, and the minimum evidence checklist for risky operations.
