# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

Personal finance management app. Django backend + React frontend.

**Architecture:**
- Backend: Django + Strawberry GraphQL + PostgreSQL
- Frontend: React + TypeScript + Tailwind CSS (v4) + shadcn/ui at `/frontend-v2`
- Task Queue: Celery + Redis
- Containerization: Docker (`local.yml`)

## Development Commands

### Backend (Docker)
```bash
# Start all services (runs on port 58000)
docker compose -f local.yml up -d

# Django management
docker compose -f local.yml run --rm django python manage.py migrate
docker compose -f local.yml run --rm django python manage.py createsuperuser
```

### Frontend (`/frontend-v2`)
```bash
cd frontend-v2
npm run dev        # Dev server (Vite, default port 5173)
npm run build      # Production build
npm run lint       # Biome lint
npm run codegen    # Regenerate GraphQL types from schema
npx tsc -p tsconfig.app.json --noEmit  # Type check
```

**Vite proxy**: `/money`, `/accounts`, `/auth-token`, `/graphql` → `http://localhost:58000`

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
- `transactionRelay` `TransactionFilter` only supports: `id`, `date`, `account` — no `retailer` or `reviewed` filter
- `StockTransactionNode` has no `date` field (use `BaseTimeStampModel.date` but not exposed in GraphQL)
- No `stockTransactionRelay` query — only `createStockTransaction` mutation
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
