---
title: Home
nav_order: 1
permalink: /
---

# claude-helm

Take the helm.

<img src="https://myowinthein.github.io/claude-helm/docs/images/banner.jpg" alt="claude-helm" width="600">

A Claude Code plugin pack for solo developers. Ten slash commands and two rule files covering shipping, refactoring, testing, documentation, legal docs, env auditing, commit-history normalization, rule adoption, and archiving your own work.

[Documentation site](https://myowinthein.github.io/claude-helm)

## Install

```bash
# Register this repo as a marketplace on your machine. Runs once.
/plugin marketplace add myowinthein/claude-helm

# Install the helm plugin from the registered marketplace.
/plugin install helm@claude-helm

# Pull the latest version from GitHub when a new release ships.
/plugin update helm@claude-helm

# Remove the plugin from your machine.
/plugin uninstall helm@claude-helm

# Apply changes after install, update, or uninstall.
/reload-plugins

# Verify installation: type /helm in the Claude Code prompt.
# Commands should appear in the picker. If not, see Known Traps in CLAUDE.md.
```

## Usage

### Commands

Links below go to detail pages. Command logic lives in `commands/*.md`.

| Command | What it does |
|---|---|
| [`/helm:ship`](https://myowinthein.github.io/claude-helm/docs/commands/ship.html) | Cut a release: calculate version, test, tag, push, and create a GitHub Release. |
| [`/helm:test`](https://myowinthein.github.io/claude-helm/docs/commands/test.html) | Assess coverage and write missing tests. |
| [`/helm:refactor`](https://myowinthein.github.io/claude-helm/docs/commands/refactor.html) | Run a project-wide refactor on a dedicated branch. |
| [`/helm:env`](https://myowinthein.github.io/claude-helm/docs/commands/env.html) | Audit and fix `.env` files and `.gitignore`: sync, formatting, missing vars, hardcoded values. |
| [`/helm:log`](https://myowinthein.github.io/claude-helm/docs/commands/log.html) | Sync `CLAUDE.md` to the current codebase. The captain's log. |
| [`/helm:manifest`](https://myowinthein.github.io/claude-helm/docs/commands/manifest.html) | Sync `README.md` to the current codebase. The vessel's manifest. |
| [`/helm:legal`](https://myowinthein.github.io/claude-helm/docs/commands/legal.html) | Generate GDPR-aware legal documents from a project scan. |
| [`/helm:normalize`](https://myowinthein.github.io/claude-helm/docs/commands/normalize.html) | Rewrite non-conventional commit messages across full repo history. |
| [`/helm:archive`](https://myowinthein.github.io/claude-helm/docs/commands/archive.html) | Archive an old project for long-term storage and future recovery. |
| [`/helm:adopt`](https://myowinthein.github.io/claude-helm/docs/commands/adopt.html) | Install or update the rule files into the current project. Setup helper, not a workflow command — see below. |

### Rules

| Rule | What it does |
|---|---|
| [`git.md`](https://myowinthein.github.io/claude-helm/docs/rules/git.html) | Git workflow rules covering branching strategy, commit conventions, code quality gates, and environment promotion. |
| [`safety.md`](https://myowinthein.github.io/claude-helm/docs/rules/safety.html) | Operational safety scan covering deployment, git, secrets, and destructive operations. |

## Using the rules

Claude Code does not auto-load a plugin's rule files into projects. Use [`/helm:adopt`](https://myowinthein.github.io/claude-helm/docs/commands/adopt.html) to install the rules into the current project. Two modes:

- **Copy**: writes the rules into `.claude/rules/`. Self-contained and committed with the repo.
- **Reference**: adds a `## Rules` section to `CLAUDE.md` pointing at the installed plugin path.

After updating the plugin, copy mode requires re-running `/helm:adopt` to pull the latest files. Reference mode updates automatically.

## Contributing

Issues and pull requests are welcome at [github.com/myowinthein/claude-helm/issues](https://github.com/myowinthein/claude-helm/issues). The intent of this pack is opinionated solo workflows, so feature requests that conflict with that scope may be declined politely.

## License

[MIT](https://github.com/myowinthein/claude-helm/blob/main/LICENSE)

<!-- last-reviewed: 2454819f5d375089e8695a96c22b5ad966d0deef -->
