# claude-helm

Take the helm.

![claude-helm](docs/images/banner.png)

A Claude Code plugin pack for solo developers. Ships slash commands and rule files for shipping, refactoring, testing, and documenting your own work.

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
```

## Usage

### Commands

| Command | What it does |
|---|---|
| [`/helm:ship`](docs/commands/ship.md) | Cut a release: calculate version, test, tag, push. |
| [`/helm:test`](docs/commands/test.md) | Assess coverage and write missing tests. |
| [`/helm:refactor`](docs/commands/refactor.md) | Run a project-wide refactor on a dedicated branch. |
| [`/helm:log`](docs/commands/log.md) | Sync `CLAUDE.md` to the current codebase. The captain's log. |
| [`/helm:manifest`](docs/commands/manifest.md) | Sync `README.md` to the current codebase. The vessel's manifest. |
| [`/helm:legal`](docs/commands/legal.md) | Generate GDPR-aware legal documents from a project scan. |
| [`/helm:normalize`](docs/commands/normalize.md) | Rewrite non-conventional commit messages across full repo history. |
| [`/helm:archive`](docs/commands/archive.md) | Archive an old project for long-term storage and future recovery. |

### Rules

| Rule | What it does |
|---|---|
| [`git.md`](docs/rules/git.md) | Git workflow rules: Solo Mode for direct-to-main, GitHub Flow for feature branches and PRs. |
| [`safety.md`](docs/rules/safety.md) | Operational safety scan covering deployment, git, secrets, and destructive operations. |

## Using the rules

Claude Code does not auto-load a plugin's rule files into projects. Use [`/helm:adopt`](docs/commands/adopt.md) to install [`git.md`](docs/rules/git.md) and [`safety.md`](docs/rules/safety.md) into the current project. Two modes:

- **Copy**: writes the rules into `.claude/rules/`. Self-contained, committed with the repo.
- **Reference** (live rules): adds a `## Rules` section to `CLAUDE.md` pointing at the installed plugin path. Updates with `/plugin update`.

## Contributing

Issues and pull requests are welcome at [github.com/myowinthein/claude-helm/issues](https://github.com/myowinthein/claude-helm/issues). The intent of this pack is opinionated solo workflows, so feature requests that conflict with that scope may be declined politely.

## License

[MIT](LICENSE)
