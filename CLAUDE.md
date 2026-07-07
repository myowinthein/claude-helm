# claude-helm

## Project Identity

**Name:** claude-helm (plugin name: `helm`)
**Stack:** Pure markdown — no runtime, no framework, no dependencies
**Purpose:** A Claude Code plugin pack for solo developers. Slash commands and rule files for shipping, refactoring, testing, and documenting your own work.
**Blast radius:** Changes to command files affect every user who has the plugin installed. Changes to `rules/` affect every project that adopted them via `/helm:adopt`.

## Project Config

git-solo: true

## Dev Commands

No install, build, or test step — the project is markdown files.

Release: `/helm:ship` (handles version bump, tag, and push)
Reload locally: `/reload-plugins` after any change

## Architecture Pointers

- `commands/*.md` — slash command definitions; YAML frontmatter `description:` powers the `/` picker tooltip
- `rules/git.md` — git workflow rules shipped to projects; covers Solo Mode and GitHub Flow
- `rules/safety.md` — operational safety rules shipped to projects
- `steps/archive/` — step files for `/helm:archive` kept here intentionally to avoid appearing as slash commands in the menu
- `docs/commands/` — human-readable detail pages per command (separate from command definitions)
- `.claude-plugin/plugin.json` — the only version file; bump this on every release
- `.claude-plugin/marketplace.json` — Claude Code marketplace registration metadata

## Behavior Rules

- Run `/helm:ship` for all releases — it handles versioning, tagging, and pushing
- Conventional Commits required on every commit (scope is mandatory)
- No PRs, no feature branches — commit directly to main

## Hard Safety Rules

- Never push without running `/helm:ship` — raw `git push` skips the version bump
- Never delete or rewrite published tags — users pin to specific versions
- Never commit secrets, credentials, or real user data (none expected in this repo)

## Known Traps

- **Plugin cache is stale after releases.** `/plugin update` + `/reload-plugins` does not invalidate `~/.claude/plugins/cache/`. After each release, command files must be manually copied from `~/.claude/plugins/marketplaces/claude-helm/commands/` into the active cache version directory, then `/reload-plugins` must be run again.
- **`steps/` directory is intentional.** Files in `steps/archive/` are not commands — they are sub-steps for `/helm:archive`. Do not move them to `commands/`.
- **`docs/commands/` is not the command source.** The slash command definitions live in `commands/`. The `docs/commands/` files are detail pages linked from README — editing them does not change command behavior.

## Rules

This project follows the rules shipped in claude-helm:
- /Users/myowinthein/.claude/plugins/marketplaces/claude-helm/rules/git.md
- /Users/myowinthein/.claude/plugins/marketplaces/claude-helm/rules/safety.md

<!-- last-reviewed: 211af6a615fb64b29dd5b0460ffc6d27115ceb79 -->
