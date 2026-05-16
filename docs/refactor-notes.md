# Refactor Notes

## High-value follow-ups

- **Biome v2 migration:** replace deprecated `files.ignore`, then run `biome check --write` in a dedicated formatting-only change.
- **Relay pagination helper:** extracted for list pages; next step is to standardize all remaining `fetchMore`-style full-load hooks separately.
- **All-data hooks:** keep `useAllTransactions` only for aggregate pages that truly need the full date range; list pages should use server pagination.
- **GraphQL codegen discipline:** run codegen only after checking all modified `.graphql` files, because generated output can include unrelated uncommitted query/schema changes.
- **Submodule commit hygiene:** check root, frontend, and backend status separately before committing; commit submodule changes first, then update root submodule pointers.
- **Review UX:** replace one-click review toggles with type-aware confirmation/linking flows for internal transfers, FX, income, and stock transactions.

## Implementation notes from this pass

- Strawberry Relay `first` is capped at 100, so use `fetchMore` or cursor pagination instead of large single requests.
- For transaction lists, server-side filters (`account`, `date`, `type`) should be preferred over client filtering.
- Lint config errors are different from source lint failures; fix config in isolation to avoid hiding behavior changes inside formatting churn.
- Cursor pagination works best as a small state hook (`cursor`, `currentPage`, `canPrev`, `reset`, `goNext`, `goPrev`) while each page keeps query-specific variables and side effects local.
- Hook refactors should get focused hook tests before wider page rewrites; this catches cursor history edge cases without requiring GraphQL mocks.
- Dev services should use fixed documented ports plus `strictPort`; silent Vite port fallback is a common source of cross-project confusion.
- When running Django management commands in the local compose container, use `bash -lc 'source /entrypoint && ...'`; direct `docker compose exec django python manage.py ...` misses required env vars such as `DATABASE_URL`.
- Account review toggles should update optimistic state from the current displayed reviewed value, not assume the default is unchecked; otherwise bidirectional toggles can appear to do nothing for already-reviewed rows.
- `Account.last_transaction` is now signal-maintained, but bulk imports that call `.save()` per row can still trigger repeated recalculation. If imports become slow, add a batch recompute path that updates affected accounts once after import.
