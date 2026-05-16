# TODO

## Backend

- [ ] P1: Resolver/API tests 보강
  - `money/helpers/` 계산 로직(`charts.py`, `snapshots.py`, `yearly.py`)
  - `toggle_reviewed` REST endpoint 동작 테스트
  - GraphQL mutation 실패 케이스(필수 필드 누락, 미인증 차단, 반환 구조)
  - `test_transaction_create_view_post`의 302/200 허용 구조를 실제 POST 성공 검증으로 정리
- [ ] P1: GraphQL 필터/페이지네이션 보강
  - 필요한 화면이 생기면 `TransactionFilter.retailer` 등 서버 필터 추가
  - Relay connection 정렬/필터 패턴 일관화
- [ ] P1: API docs
  - backend `README.md`에 GraphQL schema 위치와 REST docs 링크 정리
- [ ] P2: Hardening
  - query depth/complexity limit 검토
  - `DjangoOptimizerExtension` 적용 범위 확인
  - healthcheck endpoint 추가

## GraphQL / CI

- [ ] P0: CI에서 GraphQL operation/schema validation 실행
- [ ] P1: GraphQL Inspector diff로 breaking schema change 감지
- [ ] P1: Deprecation policy 문서화
- [ ] P1: Relay pagination fragment/공통 fragment 표준화

## Frontend

- [ ] P0: query-driven page loading/error state 잔여 점검
  - Accounts, AccountDetails, Transactions, Income, AmazonOrders, Dashboard Finance
- [ ] P1: Pagination UX 개선
  - 대형 목록에서 load-more/infinite-scroll 필요 여부 재검토
- [ ] P1: Mutation UX 개선
  - form validation, submit 중 disable, 안전한 optimistic update
- [ ] P2: `/spending-trends` 신규 페이지
  - 최근 12개월 카테고리별 월별 지출 트렌드
  - 통화 필터, 카테고리 선택 필터
- [ ] P2: `/categories` 카테고리 on/off 토글
  - 특정 달의 일시적 대형 지출 카테고리 제외
- [ ] P2: 가맹점 필터 설계 결정
  - `/transactions` 필터 vs `/retailers/:retailerId` 컴포넌트 공유
- [ ] P2: E2E smoke tests
  - login → create account → create transaction → dashboard update

## Data

- [ ] Income 세부 항목 `401(k)` / `401(K)` 표기 통일

## Integration & Ops

- [ ] End-to-end smoke test
  - UI로 retailer/account/transaction 생성 후 GraphQL 응답과 차트 갱신 확인
- [ ] Documentation
  - README에 env, local CORS, Celery worker, API docs, GraphQL endpoint 정리
