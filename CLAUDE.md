# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

Personal finance management app. Django backend + React frontend.

**Architecture:**
- Backend: Django + Strawberry GraphQL + PostgreSQL
- Frontend: React + TypeScript + Tailwind CSS (v4) + shadcn/ui at `/frontend-v2`
- Task Queue: Celery + Redis
- Containerization: Docker (`backend/local.yml` from repo root)

## Development Commands

### Backend (Docker)
```bash
# Start all services (runs Django on port 58000)
docker compose -f backend/local.yml up -d

# Django management
docker compose -f backend/local.yml run --rm django python manage.py migrate
docker compose -f backend/local.yml run --rm django python manage.py createsuperuser
```

### Frontend (`/frontend-v2`)
```bash
cd frontend-v2
npm run dev        # Dev server (Vite, fixed port 3000; strictPort=true)
npm run build      # Production build
npm run lint       # Biome lint
npm run codegen    # Regenerate GraphQL types from schema
npx tsc -p tsconfig.app.json --noEmit  # Type check
```

**Vite proxy**: `/money`, `/accounts`, `/auth-token`, `/graphql` → `http://localhost:58000`

**Local ports:**
- Frontend Vite: `3000`
- Django: `58000`
- Adminer: `58001`
- Flower: `5555`
- Backend docs: `9000`
- Do not use `5173` for finance; it is free for other Vite apps.
- Full local port map: `docs/ports.md`

## Key Architecture

### Frontend Structure (`/frontend-v2/src/`)
- `features/` — Page components by route (dashboard, accounts, transactions, etc.)
- `components/` — Shared UI components (shadcn/ui based)
- `graphql/queries/` — `.graphql` files per domain
- `graphql/generated/graphql.ts` — Auto-generated types (do not edit manually)
- `hook/` — Custom hooks (`useAllTransactions`, etc.)
- `lib/` — Utilities: `format.ts`, `constants.ts` (CATEGORY_LABELS)

### Backend Structure (`/backend/`)
- `money/models/` — Core models (accounts, transactions, shoppings, stocks, incomes, exchanges)
- `money/views/` — Django views (REST endpoints)
- `money/types/` — Strawberry GraphQL types
- `money/choices.py` — All enum definitions

### GraphQL
- Schema: `/backend/schema.graphql` (source of truth)
- Codegen config: `/frontend-v2/codegen.ts`
- Apollo Client with relay-style pagination (`first` max = 100)

## Critical Constraints

**GraphQL limitations:**
- `transactionRelay` `TransactionFilter` supports: `id`, `date`, `account`, `reviewed`, `isInternal`, `type` — no `retailer` filter
- `StockTransactionFilter.stock` is **required** (`StockFilter!`) — always pass `stock: {}` even when not filtering by stock
- `first` argument max = 100 (Strawberry relay hard limit)

**All-data fetching**: Use `useAllTransactions` hook (`/src/hook/useAllTransactions.ts`) which auto-paginates in 100-item batches. Never use `first: 500` or higher.

**REST endpoints** (toggle_reviewed only):
- `GET /money/toggle_reviewed/<numeric_id>/` — toggles `reviewed` flag
- Numeric ID from global ID: `atob(id).split(":")[1]`

**Colors:**
- Theme uses oklch (Tailwind v4). `hsl(var(--chart-N))` does NOT work in recharts SVG attributes
- Use hardcoded hex in recharts: `#6366f1` (indigo), `#14b8a6` (teal), etc.
- Primary color: `oklch(0.5 0.24 264)` (indigo)

**Enum values** (correct ones from Django `choices.py`):
- `TransactionCategory`: `EAT_OUT`, `GROCERY`, `CLOTHING`, `TRANSPORTATION` (value=`"CAR"`), `MEDICAL`, `LEISURE`, `SERVICE`, `MEMBERSHIP`, `HOUSING`, `DAILY_NECESSITY`, `INCOME`, `TRANSFER`, `STOCK`, `CASH`, `PRESENT`, `PARENTING`, `INTEREST`, `ETC`
- Use `CATEGORY_LABELS` from `/frontend-v2/src/lib/constants.ts` for display names

## Documentation

- `docs/features.md` — Page-by-page feature definitions (what users can see/do per route)
- `docs/data-model.md` — Backend model fields, relationships, enum values

## Work Rules

- Always read `docs/features.md` when implementing a new page or modifying existing page behavior
- Run `npm run codegen` after editing any `.graphql` file
- Run `npx tsc -p tsconfig.app.json --noEmit` to verify types before finishing
- Do not create new UI components if shadcn/ui has an equivalent
- Biome lint `ignore` key warning in `biome.json` is a pre-existing issue — non-blocking

## Standard Procedures

### Starting a new feature / page
1. **Read `backend/schema.graphql`** — check what queries, mutations, and filters actually exist before writing any code. Never assume a filter or mutation is absent; verify first.
2. **Read `docs/features.md`** — understand the intended behavior for the page.
3. **Enumerate all required data** — list what the page needs, then map each to an existing GraphQL query/mutation or decide a new one is needed.
4. **Prefer server-side filtering** — if a filter field exists in the GraphQL schema, use it. Fall back to `useAllTransactions` client-side filtering only when no server-side filter is available.
5. Write the `.graphql` query file → `npm run codegen` → implement the component.
6. `npx tsc -p tsconfig.app.json --noEmit` before finishing.

### Adding a backend GraphQL mutation/query
1. Edit `backend/money/types/*.py` to add the new Input type or Node field.
2. Edit `backend/money/schema.py` to wire up the mutation/query.
3. Re-export schema from the running container:
   ```bash
   docker exec finance_local_django bash -c \
     "DATABASE_URL=postgres://\$POSTGRES_USER:\$POSTGRES_PASSWORD@\$POSTGRES_HOST:\$POSTGRES_PORT/\$POSTGRES_DB \
      CELERY_BROKER_URL=\$REDIS_URL \
      python manage.py export_schema money.schema:schema --path /app/schema.graphql"
   ```
   (The container's `/app` is a volume mount of `backend/`, so this writes directly to `backend/schema.graphql`.)
4. `cp backend/schema.graphql frontend-v2/schema.graphql`
5. Add the corresponding `.graphql` query/mutation in `frontend-v2/src/graphql/queries/`.
6. `cd frontend-v2 && npm run codegen`
7. `npx tsc -p tsconfig.app.json --noEmit`

### Migrating Django views to frontend (feature audit)
When asked to port or audit features:
1. Read **all** files in `backend/money/views/` — list every view/endpoint.
2. Cross-reference against `src/routes.tsx` — identify missing routes.
3. For each missing page, check `backend/schema.graphql` for the relevant query/mutation before implementing.
4. Implement pages in dependency order (shared components first).
