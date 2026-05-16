# Work Log

## 2026-05-16

### Commit / working tree check

- Root repo: `main` at `36d7f6f` (`chore: update frontend-v2 submodule reference`)
- `frontend-v2`: `frontend-v2` at `ef4143e` with many uncommitted changes
- `backend`: `feature/code-quality-improvements` at `cf1e8fc` with uncommitted GraphQL/settings changes
- Decision: no commit was created because both submodules already contain broad uncommitted work. Committing a subset would risk mixing unrelated user work with this cleanup.

### Completed

- Cleaned stale root TODO/docs from already-implemented items.
- Added current stock holdings display for STOCK accounts.
- Added `/transactions` category filter.
- Replaced `/transactions` full-result prefetch + client slicing with server-side Relay cursor pagination.
- Removed `first > 100` usage from touched GraphQL paths.
- Fixed pagination control accessible labels.
- Fixed frontend build config import issue.

### Validation

- `npx tsc -p tsconfig.app.json --noEmit`
- `npm test -- --run --reporter=dot`
- `npm run build`

### Known blocker

- `npm run lint` still fails before checking source because `biome.json` uses the old `files.ignore` key. A config migration should be done separately because switching to `files.includes` exposes many pre-existing formatting/import-order issues across unrelated files.

### Follow-up pass

- Rechecked root/frontend/backend commits and confirmed the working tree is still broadly dirty, so no commit was created.
- Extracted shared cursor pagination state into `frontend-v2/src/hooks/use-cursor-pagination.ts`.
- Applied the hook to Transactions, Review, Exchanges, Amazon Orders, and Account Detail pages.
- Kept behavior equivalent while removing repeated `cursor` / `cursorStack` state and prev/next handlers.

### Follow-up validation

- `npx tsc -p tsconfig.app.json --noEmit`
- `npm test -- --run --reporter=dot`
- `npm run build`

### Hook test pass

- Rechecked commits and dirty state; root/frontend/backend still contain broad uncommitted changes, so no commit was created.
- Added `useCursorPagination` unit tests covering initial state, next/prev cursor history, missing cursors, and reset.
- Simplified `goPrev` to avoid setting one state from inside another state updater.

### Hook test validation

- `npx tsc -p tsconfig.app.json --noEmit`
- `npm test -- --run src/hooks/use-cursor-pagination.test.ts --reporter=dot`
- `npm test -- --run --reporter=dot`
- `npm run build`

### Port cleanup

- Confirmed finance backend is served by `backend/local.yml`, not root `local.yml`.
- Confirmed finance frontend is on `3000`; `5173` is currently used by another Vite app.
- Set Vite `strictPort: true` so finance fails fast instead of moving to an unexpected port.
- Updated root instructions and frontend README with the local port map.
- Added `docs/ports.md` as the canonical local port map and future allocation guide.
- Scanned sibling projects under `/Users/yoonjaepark/code` for common dev ports and documented collision hotspots (`3000`, `5173`, `8000`, `5432`, `5555`, `9000`).

### Cleanup / refactor pass

- Rechecked root, frontend, and backend commits; working trees are still broadly dirty, so no commit was created.
- Reviewed uncommitted changes for high-signal issues before making new edits.
- Fixed account detail review toggles so bidirectional toggles use the transaction's real reviewed state when no local optimistic state exists yet.
- Stabilized STOCK account auto-pagination dependencies by depending on primitive `hasNextPage` / `endCursor` values instead of the whole `pageInfo` object.
- Hardened the backend `last_transaction` signal to skip raw fixture loads, avoid saving stale account instances, and use `date desc, id desc` ordering.
- Added a focused backend regression test for `Account.last_transaction` updates on transaction save/delete.

### Cleanup / refactor validation

- `cd frontend-v2 && npx tsc -p tsconfig.app.json --noEmit`
- `cd frontend-v2 && npm run build`
- Initial backend `manage.py check` failed because direct `docker compose exec` did not load entrypoint-provided env vars (`DATABASE_URL`).
- `docker compose -f backend/local.yml exec -T django bash -lc 'source /entrypoint && python manage.py check && python -m compileall money/signals.py'`
- `docker compose -f backend/local.yml exec -T django bash -lc 'source /entrypoint && pytest money/tests/test_transaction.py::TestTransactionModel::test_account_last_transaction_updates_on_save_and_delete -q'`
