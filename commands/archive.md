---
description: Explore, restore, containerize, document, and seal a project for long-term archive recovery
---

# archive

## Before starting

Run from whatever branch you consider the real, complete state of the project — archive candidates are often old or neglected, so `main` is not always the branch that was actually last worked on. There is no branch-name requirement; Step 5 makes whichever branch you archived from the sole surviving branch, renamed to `main` if it isn't already.

List any unmerged local branches (`git branch --no-merged`); if any exist, warn the user that their work is not on the current branch and will not be captured — merge what should be kept into the current branch first, then re-run. Do not merge on their behalf.

## State tracking

Progress and each step's report are persisted under `.archive/` so the workflow can resume across the approval gates and survive context loss between sessions. `.archive/` is not gitignored — Step 5 commits it as part of the final seal, so it becomes a permanent audit trail in the archive (the whole point of this command is not to lose things, and nothing commits before Step 5 already switches to the private archive remote — see Task 1 — so it never risks landing anywhere else).

At the start, check for `.archive/state.json`:
- If it exists and `last_completed_step` is 5, all steps are done — tell me the archive already completed and that I can delete `.archive/` to run again. Do nothing else.
- If it exists with `last_completed_step` below 5, read it, tell me which step you are resuming from, and continue from the next step. Do not re-run completed steps.
- If it does not exist, check for `docs/archive-metadata.md` as a secondary signal before assuming a fresh run — this is a belt-and-suspenders check for a repo archived before `.archive/` was committed, or where it was deleted locally after cloning. If `docs/archive-metadata.md` exists, tell me this project appears to already be archived (per that file) and confirm before restarting the whole workflow. If it does not exist either, this is a genuine fresh run — start from Step 1. Do not create `.archive/` yet: Step 1 is read-only and nothing may be modified before its approval.

State file format:
```
{
  "last_completed_step": 0,
  "updated_at": ""
}
```

After I approve each step:
- On the first write (after Step 1's approval), create `.archive/`.
- Write that step's report — verbatim, exactly as the step produced it — to `.archive/step{N}-{name}.md`, matching the step file name (e.g. `.archive/step1-explore.md`). Do not summarize or reformat it.
- Set `last_completed_step` to N and `updated_at` to now in `.archive/state.json`.

Later steps read these report files by name rather than relying on conversation memory.

## Sequence

1. Read prompts/archive/step1-explore.md and follow it exactly.
2. Read prompts/archive/step2-restore-and-freeze.md and follow it exactly.
3. Read prompts/archive/step3-postman.md and follow it exactly.
4. Read prompts/archive/step4-documentation.md and follow it exactly.
5. Read prompts/archive/step5-finalize.md and follow it exactly.

## Important

- Each step file will tell you when to stop and wait for approval. Always stop and wait — never proceed to the next step without explicit confirmation.
- Do not skip, combine, or summarize steps.
