## Backend

- [x] P0: Secure GraphQL endpoint
  - Protect `money/graphql` with authentication (e.g., `login_required` or a custom auth check) and remove `csrf_exempt` if not needed.
  - Decide on auth strategy for API calls from the SPA (session vs token/JWT) and implement consistently.
  - Keep CSRF enforced; ensure GraphiQL UI sends `X-CSRFToken` from `csrftoken` cookie (now readable locally).
- [x] P0: CORS for local dev
  - Update `config/settings/base.py` to include `http://localhost:3000` and `http://127.0.0.1:3000` in `CORS_ALLOWED_ORIGIN_REGEXES`.
- [x] P0: Implement salary summary resolver
  - Complete `money/schema.py#get_salary_summary` and expose as a query field (e.g., `salary_summary`). Add tests.
- [ ] P0: Ensure GraphQL schema covers current UI needs
  - Verify queries/mutations used in `frontend/src/queries/*` are supported (banks, accounts, transactions, retailers, stocks, amazon orders, amount snapshots) with needed filters, ordering, and pagination.
- [ ] P1: Add filtering/pagination patterns
  - For connection fields, add commonly-used filters (date ranges, account, retailer, reviewed flag) and sorting.
- [ ] P1: Testing
  - Unit tests for resolvers/mutations (pytest) and a basic GraphQL smoke test.
  - `money/helpers/` (charts.py, snapshots.py, yearly.py) 계산 로직 테스트 없음 — 버그 위험 가장 높은 곳
  - `toggle_reviewed` REST endpoint 테스트 없음
  - `test_transaction_create_view_post` 가 302/200 모두 허용하는 헷지 구조 — POST 성공 여부 실제로 검증 못함
  - GraphQL mutation 테스트는 createTransaction만 있음. 의미 있는 테스트 포인트: 필수 필드 누락 시 error 반환, 미인증 차단, 반환 데이터 구조 확인
- [ ] P1: API docs
  - Add a minimal GraphQL schema doc link in backend `README.md`. Keep DRF Spectacular docs available for REST endpoints that remain.
- [ ] P2: Hardening
  - Add query depth/complexity limits, N+1 protections are partly handled by `DjangoOptimizerExtension`—verify usage.
  - Add healthcheck endpoint.

## GraphQL consistency (Schema-first & type-safe integration)

- [x] P0: Export and commit schema SDL
  - Add a script to export SDL from Strawberry to `backend/schema.graphql` for CI and codegen.
  - Run export on changes and commit the artifact.
- [x] P0: Frontend codegen
  - Add `frontend/codegen.yml` and `npm run codegen` to generate TS types + Apollo hooks from operations.
  - Map custom scalars (`Date`, `DateTime`, `Decimal`, `UUID`).
- [ ] P0: Validate operations against schema
  - Add CI step using `@graphql-inspector/cli validate` to ensure queries match the current schema.
- [ ] P1: Detect breaking schema changes in CI
  - Diff current SDL vs main SDL and fail on breaking changes (use GraphQL Inspector `diff`).
- [ ] P1: Deprecation policy
  - Only additive changes. Deprecate fields/args first, remove after grace period. Track in CHANGELOG.
- [ ] P1: Standardize pagination and fragments
  - Keep Relay connections and use shared fragments for consistency across screens.

## Stock Holdings (주식 보유량 표시)

`/accounts/:accountId` 에서 계좌 타입이 STOCK일 때 현재 보유 종목과 수량을 표시하고 싶음.

**현황:**
- `StockTransaction.balance` 필드에 누적 보유량이 DB에 저장되어 있음
- 백엔드 `get_stock_snapshot()` 함수가 종목별 현재 보유량을 계산하는 로직 보유
- 그러나 `stockTransactionRelay` 쿼리가 GraphQL에 없고, `StockTransactionNode`의 `balance` 필드도 미노출

**옵션 비교:**

| | 옵션 A: stockTransactionRelay 추가 | 옵션 B: stockHoldings 전용 쿼리 |
|---|---|---|
| 백엔드 작업 | 쿼리 추가 + balance 필드 노출 | 커스텀 resolver 작성 |
| 프론트 복잡도 | 계좌별 종목 마지막 balance를 직접 집계 필요 | 결과 그대로 렌더링 |
| 재사용성 | stockTransactionRelay는 다른 곳에도 활용 가능 (거래 이력 조회 등) | 이 용도에 특화 |
| 권장 | A를 먼저 추가하고 프론트에서 집계 | 데이터가 커지면 B로 전환 |

**구현 시 할 일:**
1. `backend/money/types/stocks.py` 에서 `StockTransactionNode`에 `balance` 필드 노출
2. GraphQL schema에 `stockTransactionRelay` 쿼리 추가 (account 필터 포함)
3. `npm run codegen`
4. `/accounts/:accountId` 에서 STOCK 계좌일 때 종목별 보유량 테이블 표시

---

## Frontend

- [ ] P0: Environment config
  - Add `.env.local.example` including `REACT_APP_BACKEND_URL`, `REACT_APP_GRAPHQL_ENDPOINT` (defaults: `http://localhost:58000` and `/money/graphql`).
- [x] P0: Global Apollo error handling
  - Add an error link to catch auth errors (401/403) and redirect to login or show a toast.
  - Also detect HTML login redirects and treat as unauthenticated.
- [ ] P0: Loading and error states
  - Ensure all query-driven pages (Accounts, AccountDetails, Transactions, Income, AmazonOrders, Dashboard Finance) render friendly loading/error UI.
- [x] P1: GraphQL codegen
  - Add GraphQL Code Generator to generate TS types/hooks from the live schema; wire into `npm run codegen`.
- [x] P1: Income summary coloring
  - Use accounting style: negatives red with parentheses, positives default, zero as em dash in secondary color.
- [x] P1: Income screens use generated GraphQL types
  - Switch `Income` and `Income/YearDetails` to `useGetSalaryQueryQuery`/`useGetSalaryFilterQueryQuery` and generated types; remove reliance on manual `SalaryData` interface for these screens.
- [ ] P1: Pagination UX
  - Implement load-more/infinite-scroll where lists can be large (transactions, retailers, stocks).
- [ ] P1: Transaction detail page (`/transactions/:transactionId`)
  - 거래 정보 수정 (분류, 가맹점, 메모, isInternal)
  - reviewed toggle off (계좌 상세에서는 단방향, 여기서만 off 가능)
  - 여러 진입점 지원: `/accounts/:accountId`, `/transactions`, `/review`
  - 거래 행 클릭 시 이 페이지로 이동 (현재 클릭 동작 없음)
- [ ] P1: /categories — STOCK 카테고리 지출 집계에서 기본 제외
  - 주식 매수는 현금→증권 자산 이동이므로 소비 아님
  - category-page.tsx의 집계 로직에서 `tx.type === TransactionCategory.Stock` 조건 추가
- [ ] P2: /spending-trends 페이지 신규 구현
  - 카테고리별 월별 지출 트렌드 막대/선 그래프 (기본 최근 12개월)
  - /categories와 별도 페이지로 분리 (날짜 범위 충돌 방지)
  - 통화 필터, 카테고리 선택 필터 필요
- [ ] P2: /categories — 카테고리 on/off 토글
  - 자동차 구매 등 일시적 대형 지출 카테고리를 해당 달에 한해 제외 가능하게
- [ ] P2: /transactions 분류(TransactionCategory) 필터 추가
- [ ] P2: 가맹점 필터 설계 결정
  - `/transactions`에 가맹점 필터 추가 vs `/retailers/:retailerId`와 컴포넌트 공유
  - 동일 기능이 두 곳에 나뉘지 않도록 정리 필요
- [ ] P1: Mutation UX
  - Validate forms, disable while submitting, and optimistic updates where safe (e.g., create retailer/transaction).
- [ ] P1: Frontend unit test setup
  - No test framework configured in `/frontend-v2`. Set up Vitest + React Testing Library.
  - First target: `CreateAccountDialog` — bank pre-fill 버그 회귀 방지 테스트
    - `open=true` + `defaultBankId` 전달 시 은행 필드가 미리 선택되는지 검증
    - 배경: `handleOpenChange`가 열릴 때 호출되지 않아 `useState` 초기값만 적용되던 버그 (useEffect로 수정)
- [ ] P2: E2E tests
  - Add Playwright smoke tests for the main flows (login → create account → create transaction → dashboard update).

## Integration & Ops

- [ ] P0: End-to-end smoke test
  - Run through: create retailer/account/transaction via UI, verify GraphQL responses and charts reflect changes.
- [ ] P1: Documentation
  - Update both READMEs with environment variables, local CORS, how to run workers (Celery), and how to access API docs and GraphQL endpoint.

