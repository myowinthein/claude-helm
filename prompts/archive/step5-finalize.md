# Step 5 — Finalize

Prepare the repository for final archival, clean up remotes and branches, and push to the private archive remote.

Read the prior step reports from `.archive/` — `step1-explore.md`, `step2-restore-and-freeze.md`, `step3-postman.md`, and `step4-documentation.md` — and use them as context. If a required report is missing, stop and report that the prior step must complete first.

Goal: Seal the archive — ensure the repository is clean, safe, and pushed to the correct private remote.

---

## Rules

- Present a plan and wait for explicit approval before any destructive operation.
- Do not delete branches without explicit approval.
- Do not push until origin is confirmed as the private archive remote.
- Do not commit until all cleanup tasks are complete.

---

## Task 1 — Remote Verification

List all configured remotes and their URLs. Present them to the developer:

```
REMOTES

origin      https://github.com/company/old-project.git
upstream    https://gitlab.com/client/project.git
```

Ask:
- Which of these point to company, client, or external repositories?
- What is the private archive remote URL? This can be any Git host — GitHub, GitLab, Gitea, Forgejo, or self-hosted. The repository must already exist before proceeding; remind the developer to create it if needed.

Remove all non-personal remotes. Set origin to the provided private archive remote URL.

Do not proceed to any other task until origin is confirmed as the private archive remote.

---

## Task 2 — Branch Cleanup

List all local and remote branches. The branch this workflow was run from is the one being archived — read it from Step 1's report's explicit "Archiving from:" field, not the current branch (a resumed session may have started on a different branch than the original run). If the field is absent (an older Step 1 report written before this field existed), fall back to inferring it from the Git & Repository Safety prose instead, and note in this step's report that the branch was inferred rather than read explicitly. It becomes the single branch to keep, renamed to `main` if it isn't already named `main` or `master`. No merging happens here: nothing is combined, the archived branch is simply relabeled as canonical and everything else is discarded.

Present the plan and wait for explicit approval:

```
BRANCH CLEANUP

Archiving from:  feature/rewrite  → renamed to main
Delete:          main (superseded)   (local + remote)
Delete:          fix/payment-bug     (local + remote)
Delete:          chore/cleanup       (local + remote)
```

If the archived branch is already named `main` or `master`, state that no rename is needed.

Once approved, in this order — deletion must happen before the rename, since git refuses to rename a branch onto a name that's already taken (a distinct old `main`/`master` occupies that name until it's deleted):
1. Delete every branch except the archived branch, locally and remotely — including the old `main`/`master` if it's a distinct, superseded branch
2. Verify deletion: run `git ls-remote --heads origin` and confirm only the archived branch remains
3. If the archived branch is not already named `main`/`master`: rename it locally (`git branch -m {branch} main`), then push it to remote as `main` — the name is now free, so this is a clean push, not a force-push
4. Verify local `main` is in sync with remote after push
5. If local and remote have diverged: report the divergence and stop for approval before resolving

---

## Task 3 — Git LFS and Recovery Tarballs

The Docker image and data tarballs in `recovery/docker/` are the freeze mechanism. For a clean-machine recovery years later they must travel with the archive — otherwise a fresh clone has no tarballs and would have to rebuild from registries that may no longer exist. They are large, so they go through Git LFS.

Report the total size of `recovery/docker/` and any other files over 100 MB (tracked or untracked): path, size, type.

Then confirm how the tarballs travel, since committing multi-GB binaries has storage and quota implications on the remote:

AskUserQuestion:
  question: "The recovery tarballs are {total_size}. Commit them to the archive remote via Git LFS so the project recovers on a clean machine? (Requires an LFS-capable remote with enough quota.)"
  header:   "Recovery tarballs"
  multiSelect: false
  options:
    - label: "Commit via Git LFS (Recommended)"
      description: "Track recovery/docker/ with LFS and push — the archive stays self-contained and recovers without rebuilding from registries."
    - label: "Keep local only"
      description: "Gitignore recovery/docker/ — smaller repo, but you must back the tarballs up separately; a clean clone will not have them."
    - label: "Cancel"
      description: "Stop — decide later."

If **Commit via Git LFS**:
- Set up Git LFS if not already configured.
- Track `recovery/docker/*.tar.gz` — and any other file over 100 MB found above — with patterns in `.gitattributes`.
- Stage `.gitattributes`.

If **Keep local only**:
- Add `recovery/docker/` to `.gitignore`.
- Note in the report that the tarballs are excluded from the archive, must be backed up separately, and must be placed back in `recovery/docker/` before `docker load` on recovery.
- Still LFS-track any other over-100 MB files that are staying in the repo.

If **Cancel** → stop.

---

## Task 4 — Asset Consolidation

Identify archive-worthy assets scattered across the project: screenshots, videos, audio files, sample exports, database dumps, demo files, and other binary or media assets that are not part of the application source code.

For each asset found:
1. Check whether any source file, template, stylesheet, or documentation references its current path
2. Classify as:
   - **Safe to move** — not referenced anywhere in the codebase or documentation
   - **Referenced** — one or more files reference the current path; moving would break them

For safe assets: move to `recovery/assets/` and update any documentation references.

For referenced assets: report the file, what references it, and why moving would break things. Do not move — present findings and wait for the developer to decide.

If no assets worth consolidating are found: skip silently.

---

## Task 5 — Stop Services

Identify all running services started by this project: Docker containers, Node dev servers, PHP built-in servers, Python servers, or any other processes tied to this project.

Present the list and wait for approval before stopping anything:

```
RUNNING SERVICES

docker-compose (project: vote-for-change) — containers: web, db, redis
node (port 3000) — npm run dev
```

Once approved, stop all listed services.

If no running services are found: skip silently.

---

## Task 6 — Final Commit and Push

Show git status — all staged and unstaged changes accumulated across the full workflow.

Confirm before committing and pushing — this is the final destructive action:

AskUserQuestion:
  question: "Seal the archive? This commits all accumulated changes and pushes to {archive_remote_url} (main)."
  header:   "Seal archive"
  multiSelect: false
  options:
    - label: "Commit and push (Recommended)"
      description: "Commit as chore(archive): seal project archive and push to the private archive remote"
    - label: "Cancel"
      description: "Leave changes staged/uncommitted — seal manually later"

If Cancel → stop and leave changes as-is. Do not commit or push.

If Commit and push: stage all tracked and new files explicitly — do not use `git add -A` blindly. Check `git status` first and stage only the files accumulated during this workflow:
```
git add -u                                             # stage all tracked modifications
git add recovery/ docs/ README.md .gitignore .archive/  # stage new files created this session
git commit -m "chore(archive): seal project archive"
```
`.gitattributes` only exists if Task 3 created it (LFS branch, or "Keep local only" with other over-100 MB files) — stage it too in that case, but do not add it unconditionally, since it may not exist and `git add` would fail on a nonexistent path.
`.archive/` is included deliberately — it is not gitignored. Its step reports become a permanent audit trail in the archive, and `state.json` lets a future clone recognize this project as already archived without relying solely on `docs/archive-metadata.md`.
If the tarballs are LFS-tracked (Task 3), confirm they are staged as LFS pointers (`git lfs status`) before committing — not as raw blobs.

Push to origin using the branch kept in Task 2 (now named `main`):
```
git push origin main
```

Confirm the push was successful.

---

Output the report in this format:

## Remote Changes
Previous remote(s) removed. New origin confirmed as private archive remote.

## Branch Cleanup
Branches deleted locally and remotely, including any rename to `main`. Confirmation that local and remote `main` are in sync.

## Recovery Tarballs & Git LFS
How the tarballs travel: committed via Git LFS, or kept local only (with the separate-backup note). Total tarball size. Other files tracked with LFS and `.gitattributes` changes. If no large files at all: state so.

## Asset Consolidation
Assets moved to `recovery/assets/`. Assets that could not be moved due to code references and why. If none: state no assets were consolidated.

## Services Stopped
Services identified and stopped. If none: state no running services were found.

## Final Commit
Commit hash and summary of what was committed.

## Push
Remote URL. Confirmation of success.

## Archive Complete

Archive date, archive remote URL, and restoration complexity from Step 1.

To restore this project:

```
git clone {archive_remote_url}   # install git-lfs first so the recovery tarballs download with the clone
```

If the tarballs were kept local only (not committed), place them back in `recovery/docker/` before the next step.

If Docker was used:
```
docker load < recovery/docker/{service}.tar.gz
# restore data — per docs/setup.md (auto-init from the committed dump, or restore the data-volume tarball first)
docker-compose up
```

If Docker was not used: refer to the version and toolchain documentation in `docs/setup.md`.

Full recovery instructions: `docs/setup.md`

---

Stop. Archive complete.
