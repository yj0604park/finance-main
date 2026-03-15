### Postmortem: GraphQL schema/codegen setup – what went wrong and how to avoid it

#### What slowed us down

- Using the wrong Python interpreter
  - Symptom: TypeError on union type hints (e.g., `TypeError: unsupported operand type(s) for |: 'type' and 'NoneType'`).
  - Cause: System Python 3.9 was used instead of the project virtualenv (3.10+). Strawberry/Django types expected newer Python.
  - Fix: Always use the project venv (`backend/.venv/bin/python`).

- Missing env vars for Django settings
  - Symptom: `ImproperlyConfigured: Set the USE_DOCKER environment variable` and DB/broker errors.
  - Cause: `config/settings/local.py` expects `USE_DOCKER` and a DB URL; celery also expects a broker.
  - Fix: Provide minimal env when exporting schema: `DATABASE_URL`, `CELERY_BROKER_URL`, `USE_DOCKER=no`, `DJANGO_DEBUG=True`.

- Codegen config conflicts
  - Symptom: YAML duplicated keys; codegen failed to parse.
  - Cause: Two separate config sections pasted into one file.
  - Fix: Keep a single `schema/documents/generates` block. Validate YAML.

- Codegen reading generated files
  - Symptom: “Not all operations have an unique name” due to duplicates.
  - Cause: `documents` glob included generated files.
  - Fix: Scope `documents` to `src/queries/**/*` and exclude generated artifacts.

- Frontend query/schema drift
  - Symptom: Validation errors: unknown input types (e.g., `IDBaseFilterLookup`) and wrong boolean usage (`{exact: true}`).
  - Cause: UI queries diverged from current Strawberry schema filter shapes, or frontend codegen was generated from an outdated SDL.
  - Fix: 절대 프론트 코드에서 임의 타입으로 “억지 수정”하지 말고, 먼저 `backend/schema.graphql`를 최신 런타임으로 재생성한 뒤(codegen 재실행 포함) 쿼리를 맞춤. 필요시 Docker 내에서 SDL 추출 → 프론트 `npm run codegen` 순서로 진행.

- CORS and auth friction
  - Symptom: Potential SPA access issues to GraphQL.
  - Cause: CORS not allowing localhost; GraphQL was `csrf_exempt` (insecure) or unauthenticated.
  - Fix: Add `localhost` origins, secure GraphQL with `login_required` (and plan auth strategy for SPA).

#### How to avoid this next time

- Always use the project Python
  - Use `backend/.venv/bin/python` for scripts and management commands. Consider `make` targets to enforce this.

- Provide default env for scripts
  - For local utilities (like schema export), set safe defaults if env is missing. Example env for exports:
    - `DATABASE_URL=sqlite:////tmp/finance.sqlite3`
    - `CELERY_BROKER_URL=memory://`
    - `USE_DOCKER=no`
    - `DJANGO_DEBUG=True`

- Lock a single codegen config
  - `frontend/codegen.yml` should have one `schema/documents/generates` block and target only source queries, not generated files.

- Validate queries against schema early
  - Run `npm run codegen` (or GraphQL ESLint) during development and in CI.
  - Keep `backend/schema.graphql` up to date via a script and commit it. SDL이 런타임과 다르면 SDL을 우선 재생성한다.

- Write queries from the schema, not memory
  - Inspect `backend/schema.graphql` when adding/modifying operations—match filter input names and shapes.

- CORS and auth checklist
  - Add `http://localhost:3000` and `http://127.0.0.1:3000` to allowed origins (regex anchored and escaped).
  - Protect `/money/graphql` and decide on session vs token auth for SPA.

#### Quick commands

```sh
# Export schema (local)
cd backend && DATABASE_URL=sqlite:////tmp/finance.sqlite3 \
  CELERY_BROKER_URL=memory:// USE_DOCKER=no DJANGO_DEBUG=True \
  .venv/bin/python scripts/export_schema.py

# Generate frontend types
cd frontend && npm run codegen
```


