# claude-helm

## Project Identity

**Name:** claude-helm (plugin name: `helm`)
**Stack:** Plugin is pure markdown (no runtime, no framework, no dependencies); a Jekyll + just-the-docs GitHub Pages site lives alongside it in the same repo
**Purpose:** A Claude Code plugin pack for solo developers. Slash commands and rule files for shipping, refactoring, testing, and documenting your own work.
**Blast radius:** Changes to command files affect every user who has the plugin installed. Changes to `rules/` affect every project that adopted them via `/helm:adopt`.

## Project Config

git-solo: true
git-auto-commit: true

## Dev Commands

No install, build, or test step — the project is markdown files.

Release: `/helm:ship` (handles version bump, tag, push, and GitHub Release creation)
Reload locally: `/reload-plugins` after any change

## Architecture Pointers

- `commands/*.md` — slash command definitions; YAML frontmatter `description:` powers the `/` picker tooltip
- `rules/git.md` — git workflow rules shipped to projects; covers Solo Mode and GitHub Flow
- `rules/safety.md` — operational safety rules shipped to projects
- `steps/archive/` — step files for `/helm:archive` kept here intentionally to avoid appearing as slash commands in the menu
- `docs/commands/` — human-readable detail pages per command (separate from command definitions)
- `.claude-plugin/plugin.json` — the only version file; bump this on every release
- `.claude-plugin/marketplace.json` — Claude Code marketplace registration metadata
- `_config.yml` + `Gemfile` + `_sass/` — Jekyll/GitHub Pages site config; `docs/` files serve as site pages

## Behavior Rules

- Run `/helm:ship` for all releases — it handles versioning, tagging, and pushing
- Conventional Commits required on every commit (scope is mandatory)
- No PRs, no feature branches — commit directly to main

## Writing Style

When writing content for `CLAUDE.md`, `README.md`, or any generated document:
- Use em-dashes sparingly. Only use one when no other punctuation (comma, semicolon, colon, or a new sentence) works as well. When in doubt, restructure the sentence instead.

## Hard Safety Rules

- Never push without running `/helm:ship` — raw `git push` skips the version bump
- Never delete or rewrite published tags — users pin to specific versions
- Never commit secrets, credentials, or real user data (none expected in this repo)

## Known Traps

- **Plugin cache is stale after releases.** `/plugin update` + `/reload-plugins` does not invalidate `~/.claude/plugins/cache/`. After each release, command files must be manually copied from `~/.claude/plugins/marketplaces/claude-helm/commands/` into the active cache version directory, then `/reload-plugins` must be run again.
- **`steps/` directory is intentional.** Files in `steps/archive/` are not commands — they are sub-steps for `/helm:archive`. Do not move them to `commands/`.
- **`docs/commands/` is not the command source.** The slash command definitions live in `commands/`. The `docs/commands/` files are detail pages that serve as Jekyll site content and link from README — editing them does not change command behavior.

## Rules

This project follows the rules shipped in claude-helm:
- rules/git.md
- rules/safety.md

<!-- last-reviewed: 216cea2f3695b4b1930dce053998687e2f252e0e -->
