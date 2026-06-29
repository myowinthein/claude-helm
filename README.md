# claude-helm

Take the helm.

A Claude Code plugin pack for solo developers. Ships slash commands and rule files for shipping, refactoring, testing, and documenting your own work. A growing collection that gains new commands and rules as workflows mature.

## Install

```bash
/plugin marketplace add myowinthein/claude-helm
/plugin install helm@claude-helm
```

## What's inside

### Commands

| Command | What it does |
|---|---|
| `/helm:ship` | Versioned release flow. Calculates the next version from Conventional Commits, runs tests, tags, and promotes to environment branches if present. |
| `/helm:refactor` | Branch off, run a targeted refactor by category (architecture, code quality, tests, dependencies), then merge or open a PR. |
| `/helm:test` | Detect test framework, assess coverage, prioritize gaps, write tests for changes or full project. |
| `/helm:legal` | Generate GDPR-aware privacy, terms, and conditional legal pages (cookies, disclaimers) based on a project scan. |
| `/helm:log` | Keep `CLAUDE.md` in sync with the codebase. The captain's record of the voyage. Full scan on first run, gap update on subsequent runs. |
| `/helm:manifest` | Keep `README.md` in sync with the codebase. The vessel's public listing of what is aboard. Follows the Standard Readme spec. |

All plugin commands are namespaced under `/helm:` so they never collide with project-level commands of the same name. If your project has its own `.claude/commands/ship.md`, that runs as `/ship` while the plugin version runs as `/helm:ship`. Both coexist.

### Rules

| Rule | What it does |
|---|---|
| `git.md` | Defines team mode and **Solo Mode**. Activate Solo Mode by setting `git-solo: true` in your `CLAUDE.md` Project Config section to commit straight to `main`, no feature branches, no PRs. |
| `safety.md` | Operational safety scan: deployment risk, git exposure, secret handling, destructive operations. Run during project bootstrap to surface what should never be automated. |

## Using the rules

Slash commands work everywhere on your machine the moment you install the plugin. Rule files are different. Claude Code does not auto-load rules from a plugin into every project, so you opt them into a specific project in one of two ways.

### Option A: copy the rule into the project (simplest)

From inside the project where you want the rule active:

```bash
mkdir -p .claude/rules
cp ~/.claude/plugins/claude-helm/helm/rules/git.md   .claude/rules/git.md
cp ~/.claude/plugins/claude-helm/helm/rules/safety.md .claude/rules/safety.md
```

Now `git.md` and `safety.md` live alongside the project, get committed with it, and apply automatically. Update the project's `CLAUDE.md` if needed to reference them.

The plugin install path follows the pattern `~/.claude/plugins/<marketplace-name>/<plugin-name>/`. Adjust the source path if your install location differs.

### Option B: reference them from CLAUDE.md (no copy, no commit)

In the project's `CLAUDE.md`, add a section pointing at the installed plugin:

```markdown
## Rules

This project follows the rules shipped in [claude-helm](https://github.com/myowinthein/claude-helm):

- `~/.claude/plugins/claude-helm/helm/rules/git.md`
- `~/.claude/plugins/claude-helm/helm/rules/safety.md`
```

The rules stay in the plugin install path. Pros: updates with `/plugin update`. Cons: every collaborator needs the plugin installed at the same path, so this option suits solo work better than shared repos.

### Activating Solo Mode

Once `git.md` is in place by either method, add this under a `## Project Config` section in your `CLAUDE.md`:

```
- `git-solo: true`
```

`git.md` reads that flag and switches its behavior. Without the flag, the default team workflow (feature branches, PRs) applies.

## Solo Mode

The defining flavor of this pack. Most Claude Code workflows assume team conventions: feature branches, pull requests, code review. Solo Mode flips that for indie developers shipping their own products. You stay disciplined where it matters (Conventional Commits, semver, real tests) and skip the ceremony that has no value when you are the only reviewer.

To activate, add this under a `## Project Config` section in your `CLAUDE.md`:

```
- `git-solo: true`
```

The rules in `git.md` adjust accordingly: no PR flow, direct commits to `main`, version tags from the `/helm:ship` command.

## Versioning

Releases follow [Semantic Versioning](https://semver.org/). Version bumps follow [Conventional Commits](https://www.conventionalcommits.org/): `feat` triggers a minor bump, `fix` triggers a patch bump, `BREAKING CHANGE` triggers a major bump. Tagged releases live at [github.com/myowinthein/claude-helm/tags](https://github.com/myowinthein/claude-helm/tags).

## Contributing

Issues and pull requests are welcome at [github.com/myowinthein/claude-helm/issues](https://github.com/myowinthein/claude-helm/issues). The intent of this pack is opinionated solo workflows, so feature requests that conflict with that scope may be declined politely.

## License

[MIT](LICENSE)
