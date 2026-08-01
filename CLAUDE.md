# claude-helm

## Project Identity

**Name:** claude-helm (plugin name: `helm`)
**Stack:** Plugin is pure markdown (no runtime, no framework, no dependencies); a Jekyll + just-the-docs GitHub Pages site lives alongside it in the same repo
**Purpose:** A Claude Code plugin pack for solo developers. Ten slash commands and two rule files covering shipping, refactoring, testing, documenting, auditing, and archiving your own work.
**Blast radius:** Changes to `commands/*.md` affect every user who has the plugin installed. Changes to `rules/` affect every project that adopted them via `/helm:adopt`.

## Project Config

git-strategy: solo
git-auto-commit: true
readme-style: custom

## Dev Commands

No install/build step for the plugin itself — the commands are markdown files. The Jekyll docs site does have a local test step: `bundle install`, then `bundle exec jekyll build && bundle exec htmlproofer _site --disable-external` to catch broken links, missing images, and malformed HTML before shipping. No CI/CD configured (no `.github/workflows/`) — this local check is not wired into `/helm:ship` or run automatically.

Release: `/helm:ship` (handles version bump, tag, push, and GitHub Release creation)
Reload locally: `/reload-plugins` after any change

## Architecture Pointers

- `commands/*.md` — slash command definitions (10: adopt, archive, env, legal, log, manifest, normalize, refactor, ship, test); YAML frontmatter `description:` powers the `/` picker tooltip
- `rules/git.md`, `rules/safety.md` — shipped rule files, installed into adopting projects via `/helm:adopt`
- `prompts/archive/step{1-5}-*.md` — sub-step files for `/helm:archive`; `commands/archive.md` is the orchestrator that reads and sequences them
- `docs/commands/*.md` (+ `docs/commands/archive/`) — human-readable detail pages per command, published as Jekyll site content; not command source
- `docs/rules/` — detail pages for the two rule files
- `docs/legal/*.md` — this repo's own generated legal documents (privacy policy, terms, disclaimer, EULA); excluded from the Jekyll build and not yet linked from the site or README
- `.claude/helm/*.json` — this plugin's own ledgers (`refactor-log.json`, `test-log.json`, `legal-manifest.json`; `normalize-log.json` once `/helm:normalize` runs here). Namespaced under `helm/` rather than flat in `.claude/` to avoid colliding with another plugin's own generically-named files. Commands transparently migrate a legacy flat-path file on first run after update.
- `.claude-plugin/plugin.json` — the version file; bump this on every release
- `.claude-plugin/marketplace.json` — marketplace registration metadata; `/helm:ship` keeps its `plugins[].version` in sync with `plugin.json` automatically
- `_config.yml` + `Gemfile` — Jekyll/GitHub Pages site config; `docs/` files serve as site pages

## Domain Rules

- `docs/commands/*.md` are detail pages, not command source — editing them never changes command behavior; the real definitions live in `commands/*.md`.
- `prompts/archive/*.md` must never move into `commands/` — that would make each step appear as its own standalone slash command, breaking `/helm:archive`'s single-entry orchestration.
- Ledger files (`.claude/helm/refactor-log.json`, and `.claude/helm/test-log.json` if a project runs `/helm:test`) are append-only audit trails — a scan appends new findings, it never overwrites the array. `normalize-log.json` is a different shape (one entry per branch, replaced/dropped as branches come and go) — not append-only in the same sense.
- Docs pages (`docs/commands/`, `docs/rules/`) must stay behaviorally consistent with the command/rule files they describe, but nothing enforces this automatically — drift is caught only by manual review or a `/helm:refactor` pass.
- AskUserQuestion caps at 4 options — any command building a picker from a dynamic list (environment branches, refactor categories, legal documents) must merge or truncate down to 4, never assume the list stays small.
- `README.md` ships in three real contexts (Jekyll site homepage, GitHub.com's own renderer, the distributed plugin package via `marketplace.json`), only one of which resolves relative links to `docs/`. Its doc-page links and banner image use absolute `https://myowinthein.github.io/claude-helm/...` URLs for this reason — don't "simplify" them back to relative paths.

## Behavior Rules

- Run `/helm:ship` for all releases — it handles versioning, tagging, and pushing
- Conventional Commits required on every commit (scope is mandatory)
- No PRs, no feature branches — commit directly to main
- `git push` always requires its own confirmation, even with `git-auto-commit: true` — the commit itself may be silent, but push is not

## Hard Safety Rules

- Never delete or rewrite published tags — users pin to specific versions
- Never commit secrets, credentials, or real user data (none expected in this repo)

## Known Traps

- **Plugin cache is stale after releases.** `/plugin update` + `/reload-plugins` does not invalidate `~/.claude/plugins/cache/`. After each release, command files must be manually copied from `~/.claude/plugins/marketplaces/claude-helm/commands/` into the active cache version directory, then `/reload-plugins` must be run again. Workaround: `cp ~/.claude/plugins/marketplaces/claude-helm/commands/*.md "$(ls -d ~/.claude/plugins/cache/claude-helm/helm/*/commands/ | tail -1)"` then `/reload-plugins`.
- **Don't "fix" `prompts/` or `docs/commands/` by moving or redefining them.** See Domain Rules above — `prompts/archive/*.md` staying out of `commands/`, and `docs/commands/*.md` not being command source, are both easy mistakes to reach for when something there looks structurally odd.
- **No CI to gate on.** `/helm:ship` and `/helm:refactor`'s environment-promotion CI-status checks both no-op silently here — there's no `.github/workflows/`. `bundle exec jekyll build && bundle exec htmlproofer` (see Dev Commands) verifies the site locally, but nothing runs it automatically before or after a release.
- **`git filter-repo` removes the `origin` remote by default** after rewriting history, since it assumes it's operating on a disposable clone. `/helm:normalize`'s preferred rewrite path captures `origin`'s URL beforehand and restores it after — if that logic is ever touched, this is why it's there.

## Rules

This project follows the rules shipped in claude-helm:
- rules/git.md
- rules/safety.md

<!-- last-reviewed: 69afcb461d5b538710db0e11a1c6d4892b5d96ce -->
