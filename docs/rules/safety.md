# safety.md

The operational safety reference for a project. Defines which actions always need explicit confirmation, what to scan when bootstrapping a new project, and how to document risk categories so future agents read the same constraints.

The file is not procedural; it does not "run." It is loaded as context so that any agent acting in the project respects the same operational guardrails.

## When to run a safety scan

Run during project bootstrap, ideally as part of `/helm:log` (Full scan mode). Update when deployment, infrastructure, or environment configuration changes.

Only document risks supported by evidence in the repository. No assumptions, no speculation.

## Agent Execution Boundaries

These actions always require explicit confirmation, regardless of context or autonomy level:

- `git push`, force-push, tag creation.
- Any deployment command.
- Migrations, database imports, database resets.
- Infrastructure changes, DNS changes, secret rotation.
- Destructive operations: resets, drops, clears, environment recreation.

Projects can extend the list, never shrink it.

## Scan Targets

Bootstrap scan looks at:

- `CLAUDE.md`, `.claude/rules`, README files, project documentation.
- CI/CD pipelines (GitHub Actions, GitLab CI), deployment scripts.
- Composer scripts, package scripts, Makefiles, shell scripts.
- Docker, Lando, Laravel Sail configuration.
- Git remotes, branch structure, protected branches.
- `.env.example`, credential files, service account files, private keys.
- Queue config, scheduler config, background job definitions.
- Cloud/infrastructure config (AWS, DO, Cloudflare, Elasticsearch, Redis, etc.).

## Risk Categories

For each category, scan for evidence and document findings as a pair:

```
Risk: one-line description
Instruction: direct rule for future agents
```

Vague warnings have no value. Every instruction must be actionable.

**Deployment**

Which branches auto-deploy. Whether automatic or approval-gated. Whether tests run before deployment. Which environments are affected.

**Git**

Protected branches. Multiple remotes. Force-push exposure. Mirror repositories. Company-owned or production remotes.

**Database**

Migrations, seeders, bulk scripts, data repair scripts, sync/import jobs that could affect shared environments.

**Queue and Jobs**

Operations that could duplicate, lose, or disrupt background work. Scheduler commands that affect shared state.

**Infrastructure**

Cloud storage, CDN, search indexes, caching systems. Operations that should never be performed automatically.

**Secrets and Credentials**

Where they live: env files, credential files, service accounts, keys, certs, deployment tokens. Never print, commit, copy into docs, or expose in output. Only identify locations and risks; never reproduce values.

**Destructive Operations**

Commands or scripts that delete data, reset environments, recreate databases, destroy infrastructure, or clear storage.

**Project-Specific**

Generated code, vendor-managed code, legacy systems, sync processes, unusual deployment requirements, operational dependencies unique to this repo.

## Environment Classification

For each known environment (local, staging, UAT, production, preview), document:

- Purpose.
- Ownership.
- Whether shared.
- Whether persistent or ephemeral.
- Whether safe for testing.
- Any access restrictions or environment-specific rules.

This block answers the question "if I push here, what breaks for whom?" before the push happens.

## Minimum Evidence Checklist

When stating what the project enforces, infer only from actual CI/hooks/pipelines. Do not claim a check exists unless the project actually runs it.

Examples: unit tests, feature tests, linting, static analysis, frontend build, generated file verification, pre-commit hooks.

## See also

- [`git.md`](git.md) - the workflow rules that reference safety boundaries
- [`/helm:log`](../commands/log.md) - the command that scans the project and surfaces what belongs in this file
