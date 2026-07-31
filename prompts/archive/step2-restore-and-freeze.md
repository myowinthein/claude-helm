# Step 2 — Restore and Freeze

Restore this project and freeze its environment for long-term archive recovery.

Read Step 1's report from `.archive/step1-explore.md` and use it as context. If it is missing, stop and report that Step 1 must complete first.

Goal: Get the project running locally, containerize it with pinned image versions, and export the built images to tarballs so the environment can be recovered years later without depending on external registries.

---

## Rules

- Preserve the original project as much as possible. Prefer compatibility fixes over upgrades, restoration over modernization.
- Record every file created, modified, or deleted with the reason.
- Use placeholder values for external services.
- Do not connect to production systems, live databases, or external services.
- Do not send emails, SMS, notifications, webhooks, or invoke payment gateways without explicit approval.
- Do not change Git state — no commit, push, pull, fetch, checkout, reset, rebase, merge, or history rewrite.
- Work only within the current repository. If another repository appears required, report which one and why, then stop and ask how to proceed — either continue without it (noting the gap in the report), or wait while you make it available, then continue on your signal.

---

## Restoration Philosophy

The primary goal is not just to run the project today — it is to freeze the environment so it can be recovered 5–10 years from now, regardless of what runtimes, package versions, or registries are available at that time.

Docker is the preferred environment freeze mechanism. Base image tags and registries can disappear over time, so after a successful build, export all images to gzipped tarballs stored in `recovery/docker/`. Recovery then requires only Docker and the tarballs — no internet access, no registry dependency.

For project types where Docker does not apply, document exact runtime versions, SDK versions, and toolchain versions in `docs/setup.md` as the freeze mechanism instead.

---

## Restoration Strategy

Use Step 1's findings to select the approach. Docker is the primary choice for most project types.

**Containerize with Docker (primary for most projects):**

If a Docker or Docker Compose setup already exists: use it, but verify all image tags are pinned to specific versions (e.g. `node:18.17.1-alpine`, not `node:18` or `node:latest`). Update any unpinned tags before building.

If no Docker setup exists: create a `Dockerfile` and `docker-compose.yml` appropriate for the project's stack and runtime. Pin all base image versions specifically.

By project type:

- **Web backend / API** (any language or framework): Docker Compose with app + database + any required services
- **CMS** (WordPress, Drupal, etc.): Docker Compose with web server + database
- **Web frontend** (React, Vue, Angular, Svelte, etc.): Docker for the build environment and a simple static server for the output
- **Static site** (Jekyll, Hugo, Gatsby, Astro, etc.): Docker for the build environment; output is static files
- **Full-stack / monorepo**: Docker Compose for all required services; restore minimum components for a working demonstration
- **Data science / ML** (Jupyter, Python scripts, R, etc.): Docker with pinned base image and all dependencies installed; verify notebooks or scripts execute inside the container
- **CLI tool or script** (with non-trivial runtime): Docker with pinned runtime image

**Document versions instead of containerizing (where Docker does not apply):**

- **Mobile** (React Native, Flutter, iOS, Android): Docker cannot meaningfully freeze a mobile build environment. Document exact SDK version, toolchain version, target platform version, and any required IDE version in `docs/setup.md`.
- **Desktop** (Electron, Tauri, etc.): prefer Docker for the build step if possible; otherwise document exact Node or Rust version.
- **Browser extension**: document exact browser version and extension API versions used.
- **Library / package**: Docker for the test environment; document the published package version.

---

## Data Restoration

First determine the project's data layer from Step 1's findings and the project type, then restore accordingly:

- **Database-backed** (backend / API, CMS, full-stack): restore from a data source — follow the priority list below.
- **Consumes an external or backend API** (frontend-only): no local database to restore. Record how it gets data — whether the API is part of this repo (restore it too) or external (note as a dependency) — and how demo data is provided (mock data, fixtures, recorded responses).
- **Local or file-based storage** (desktop, mobile, CLI, data science using SQLite, embedded stores, or data files): restore the local data files or embedded database if present in the repo.
- **No data layer** (static site, stateless CLI, library): skip — state that no data restoration is required.

For database-backed projects, discover all available data sources in the repository, assess each for completeness and suitability, and use the best in this order:

1. Repository-contained database dumps or backup files (SQL, MongoDB, SQLite, Redis snapshots, etc.)
2. Seed scripts, fixture files, or sample datasets
3. Schema migrations — reconstruct from scratch
4. File-based data sources (CSV, JSON, XML data files used by the application)
5. Documented external exports
6. Existing localhost database — only if no repository-contained source exists

If no data source exists and the project requires one: report the gap and explain the impact on the demonstration, then stop and ask how to proceed — either continue without data (build with an empty or schema-only database, noting the limitation in the report), or wait while you provide a dump or point to a data source, then continue on your signal.

If restoration requires a network, VPN-accessible, or cloud database: do not connect. Report which source needs it and why, then stop and ask how to proceed — either skip it and fall back to the next-best source (noting the reduced data in the report), or wait while you arrange access or provide a dump, then continue on your signal.

---

## Image Export

After the project is verified as running:

Export all Docker images to `recovery/docker/` as gzipped tarballs:

```
docker save {service} | gzip > recovery/docker/{service}.tar.gz
```

After each export, verify the file is not empty:
```
[ -s recovery/docker/{service}.tar.gz ] || echo "ERROR: tarball is empty — re-export before proceeding"
```

The shell pipe may silently produce an empty file if `docker save` fails; this check catches silent failures.

Export one tarball per service (web, database, cache, etc.).

After exporting each tarball, validate it by running:
```
docker load < recovery/docker/{service}.tar.gz
```
If `docker load` fails, the tarball is corrupt — re-export before proceeding.

Add `recovery/docker/` to `.gitignore` — tarballs are stored locally alongside the repo, not committed.

Document the recovery commands in `docs/setup.md` (include the data-restore step from Data Persistence below, so recovery does not come up empty):

```
docker load < recovery/docker/web.tar.gz
docker load < recovery/docker/db.tar.gz
# restore data — see Data Persistence (auto-init from the committed dump, or restore the data-volume tarball first)
docker-compose up
```

---

## Data Persistence

Exporting the service images does not capture the restored data. A database keeps its data in a **volume**, not its image — so `docker save postgres` preserves the engine, not your rows. Without an extra step, recovery via `docker load` + `docker-compose up` comes up with an empty database.

Skip this section for the non-database data layers (API-consuming frontend, no data layer). For **database-backed** and **local/file-based** projects, make the restored data recoverable — pick the mechanism that fits the source:

- **Re-runnable source in the repo** (dump, seed, or migrations — the common case from the priority list): the source already travels with the repo. Wire the recovery so the database restores from it on startup — mount the dump into the image's init directory (e.g. `/docker-entrypoint-initdb.d/` for Postgres/MySQL) or document the exact restore command. Keeps the data human-readable and portable across future DB versions.
- **No re-runnable source** (data exists only in the live volume — e.g. restored from an existing localhost DB): export the data volume to `recovery/docker/` as a gzipped tarball, and validate it is non-empty:
  ```
  docker run --rm -v {db_volume}:/data -v "$(pwd)/recovery/docker:/backup" alpine tar czf /backup/{db}-data.tar.gz -C /data .
  [ -s recovery/docker/{db}-data.tar.gz ] || echo "ERROR: data tarball is empty — re-export before proceeding"
  ```
  Recovery restores the volume from this tarball before `docker-compose up`.

Whichever mechanism is used, document its exact recovery step in `docs/setup.md` alongside the image-load commands.

---

## Tasks

1. Select restoration strategy using Step 1 context
2. Configure the local environment
3. Create or update Docker setup with pinned image versions
4. Install dependencies
5. Restore available data
6. Build and start the containers
7. Verify the project works
8. Export all images to `recovery/docker/` as gzipped tarballs
9. Make the restored data recoverable (re-runnable source on startup, or a data-volume tarball) and document its recovery step
10. Add `recovery/docker/` to `.gitignore`
11. Record all changes made

---

Output the report in this format:

## Restoration Strategy
Chosen approach and why. Whether Docker was created or already existed. What was tried first if the first choice failed.

## Docker Setup
Base images used with exact pinned versions. Services defined. Whether Dockerfile and docker-compose.yml were created or already existed.

## Image Exports
Images exported to `recovery/docker/`. Size of each tarball. Recovery commands.

## Data Persistence
How the restored data is made recoverable: re-runnable source on startup, or an exported data-volume tarball. State "not applicable" for non-database data layers. The exact recovery step documented in `docs/setup.md`.

## Data & Database
Data source selected and why. How data was restored. Demo accounts or sample data available if any.

## Changes Made
Every file created, modified, or deleted: path, what changed (or why it was removed), and why it was necessary.

## Environment Variables
Any variables added or changed from the original.

## Commands Used
Exact commands run in order.

## Access Points
How to access or run the project locally (URLs and ports for web projects, commands for CLI tools, entry points for scripts or notebooks). State "not applicable" if the project type has no access point.

## Verification Results
What was tested and what the results were. Any critical errors found.

## Restoration Outcome
One of: Fully Restored / Partially Restored / Buildable But Not Runnable / Blocked / Cannot Be Restored Without Missing Dependencies — one sentence explaining why.

## Remaining Blockers
Anything that prevented full restoration or needs attention before proceeding to Step 3.

---

Do not commit or push. Output the report and stop. Wait for explicit approval before proceeding to Step 3.

If Restoration Outcome is anything other than Fully Restored: clearly state that proceeding to Step 3 is not recommended until blockers are resolved. List what needs to be fixed.
