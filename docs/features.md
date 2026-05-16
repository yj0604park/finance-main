# Feature Overview

현재 React 프론트엔드 라우트 기준의 기능 요약이다. 세부 구현 규칙보다 "어떤 화면이 있고 무엇을 할 수 있는지"를 빠르게 확인하는 용도로 유지한다.

## 핵심 흐름

- **인증:** `/login`에서 Django session 기반 로그인 후 보호된 화면에 접근한다.
- **자산 개요:** `/dashboard`에서 KRW/USD 잔액 추이, 은행별 잔액, 마지막 거래일을 본다.
- **계좌 관리:** `/accounts`에서 계좌를 필터링/생성하고, `/accounts/:accountId`에서 계좌별 거래와 STOCK 계좌 보유 종목을 본다.
- **거래 관리:** `/transactions`에서 전체 거래를 필터링/페이지 이동하고, `/transactions/:transactionId`에서 거래 날짜/분류/가맹점/메모/내부이체/검토 상태를 수정한다.
- **분석:** `/categories`, `/retailers`, `/income`, `/stocks`, `/exchanges`, `/audit`에서 영역별 집계와 데이터 품질을 확인한다.

## 페이지별 현황

| Route | 구현 현황 |
|---|---|
| `/dashboard` | 총 잔액, 잔액 추이, 은행별 카드, 마지막 거래일 표시 |
| `/accounts` | 은행/활성 여부 필터, 계좌 생성, 계좌 상세 이동 |
| `/accounts/:accountId` | 계좌 요약, 거래 목록, 미검토 필터, 거래 추가, 잔액 업데이트, STOCK 계좌 주식 거래/보유 종목 표시 |
| `/transactions` | 계좌/날짜/분류 필터, 50건 단위 Relay pagination, 거래 상세 이동 |
| `/transactions/:transactionId` | 거래 상세 조회, 기본 정보 수정, reviewed 토글 |
| `/categories` | 월/기간 기준 카테고리별 수입·지출 집계, STOCK/내부이체 제외 |
| `/retailers` | 월별 가맹점 집계, 가맹점 생성, 가맹점 상세 이동 |
| `/retailers/:retailerId` | 특정 가맹점 거래 내역 조회 |
| `/income` | 급여 데이터를 연도별로 집계하고 연도 상세로 이동 |
| `/income/:year` | 연도별 급여 상세, 급여 생성/수정, pay/tax/deduction 상세 펼침 |
| `/stocks` | 주식 종목 목록, 통화별 종목 수, 종목 생성 |
| `/stocks/:stockId` | 종목 가격 이력 차트/목록, 가격 추가 |
| `/stock-transactions/:stockTxId` | 주식 거래 상세, 연결된 일반 거래 보기/연결 해제 |
| `/review` | 미검토 거래 목록, 날짜 필터, 페이지 단위 검토 완료 처리 |
| `/amazon` | Amazon 주문 목록, 주문 생성, 최근 거래 연결, 반품 여부 표시 |
| `/exchanges` | 환전 내역 조회, 페이지 기준 평균 환율 표시 |
| `/audit` | 비활성 계좌 잔고, 내부이체 불일치, 주식 거래 연결 누락, 급여 데이터 품질 점검 |

## 현재 남은 큰 빈틈

- Type-aware review flow: 내부이체, 환전, 급여, 주식 거래를 단순 toggle이 아니라 연결 확인 후 reviewed 처리하는 흐름.
- `/spending-trends`: 최근 6~12개월 카테고리별 지출 추이 화면.
- `/categories` 카테고리 on/off 또는 일시적 대형 지출 제외 기능.
- 환전 생성/수정 UI와 GraphQL mutation.
- E2E smoke test: login → 계좌 생성 → 거래 생성 → dashboard 반영.

## 구현 참고

- 대형 목록은 Relay `first <= 100` 제한에 맞춰 cursor pagination 또는 auto-pagination hook을 사용한다.
- 일반 목록 화면은 서버 필터를 우선 사용하고, 전체 데이터 집계가 필요한 분석 화면에서만 `useAllTransactions`를 사용한다.
- GraphQL 변경 시 `backend/schema.graphql` export → `frontend-v2/schema.graphql` 복사 → `npm run codegen` 순서로 동기화한다.
