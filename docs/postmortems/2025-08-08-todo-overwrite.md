# Postmortem: TODO.md overwrite at repo root (2025-08-08)

## Summary
During a fix to frontend build issues, the repo-root `TODO.md` was unintentionally overwritten with new entries focused on the Apollo Client upgrade.

## Impact
- Existing tasks in `TODO.md` were lost in the working tree (not committed previously), causing loss of context.

## Root Cause
- I operated from the `frontend/` subdirectory and assumed `TODO.md` didn’t exist at the repo root.
- I created/edited `TODO.md` without first checking whether a pre-existing file existed one level up.
- There was no prior commit of `TODO.md`, so version history could not restore the content.

## Detection
- User reported that `TODO.md` exists one level above `frontend/`.

## Resolution
- Created a timestamped backup of the current `TODO.md` to prevent further data loss.
- Will merge recovered/remembered previous items once available from the user or other sources (e.g., local editor history, backup tooling).

## Prevention
- Always check from the repository root before creating or overwriting top-level files.
- Add a pre-change checklist for docs: verify file existence and view diff before save.
- Prefer appending to `TODO.md` with a clearly demarcated session section instead of replacing content.
- Consider enabling automated backups/snapshots of `docs/` and root markdown files.

## Action Items
- [ ] Recover prior `TODO.md` entries (from user memory, local backups, or editor history) and merge.
- [x] Create backup of current `TODO.md` (`TODO.bak.2025-08-08.md`).
- [ ] Update contribution guide to include doc-change checklist.