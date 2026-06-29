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
| `/ship` | Versioned release flow. Calculates the next version from Conventional Commits, runs tests, tags, and promotes to environment branches if present. |
| `/refactor` | Branch off, run a targeted refactor by category (architecture, code quality, tests, dependencies), then merge or open a PR. |
| `/test` | Detect test framework, assess coverage, prioritize gaps, write tests for changes or full project. |
| `/legal` | Generate GDPR-aware privacy, terms, and conditional legal pages (cookies, disclaimers) based on a project scan. |
| `/update-claude` | Keep `CLAUDE.md` in sync with the codebase. Full scan on first run, gap update on subsequent runs. |
| `/update-readme` | Keep `README.md` in sync with the codebase. Follows the Standard Readme spec. |

### Rules

| Rule | What it does |
|---|---|
| `git.md` | Defines team mode and **Solo Mode**. Activate Solo Mode by setting `git-solo: true` in your `CLAUDE.md` Project Config section to commit straight to `main`, no feature branches, no PRs. |
| `safety.md` | Operational safety scan: deployment risk, git exposure, secret handling, destructive operations. Run during project bootstrap to surface what should never be automated. |

## Solo Mode

The defining flavor of this pack. Most Claude Code workflows assume team conventions: feature branches, pull requests, code review. Solo Mode flips that for indie developers shipping their own products. You stay disciplined where it matters (Conventional Commits, semver, real tests) and skip the ceremony that has no value when you are the only reviewer.

To activate, add this under a `## Project Config` section in your `CLAUDE.md`:

```
- `git-solo: true`
```

The rules in `git.md` adjust accordingly: no PR flow, direct commits to `main`, version tags from the `/ship` command.

## Versioning

Releases follow [Semantic Versioning](https://semver.org/). Version bumps follow [Conventional Commits](https://www.conventionalcommits.org/): `feat` triggers a minor bump, `fix` triggers a patch bump, `BREAKING CHANGE` triggers a major bump. Tagged releases live at [github.com/myowinthein/claude-helm/tags](https://github.com/myowinthein/claude-helm/tags).

## Contributing

Issues and pull requests are welcome at [github.com/myowinthein/claude-helm/issues](https://github.com/myowinthein/claude-helm/issues). The intent of this pack is opinionated solo workflows, so feature requests that conflict with that scope may be declined politely.

## License

[MIT](LICENSE)
