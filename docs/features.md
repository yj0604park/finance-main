# 페이지별 기능 정의

> 각 페이지에서 사용자가 할 수 있는 작업과 기대 동작을 정리한 문서.
> Claude에게 작업을 지시할 때 이 문서를 참고 기준으로 사용한다.

---

## /dashboard

**목적:** 전체 자산 현황을 한눈에 파악한다.

**볼 수 있는 것:**
- KRW / USD 총 잔액 (모든 계좌 합산)
- KRW / USD 잔액 추이 차트 (AmountSnapshot 기반)
- 은행별 카드 (active 계좌 수, 통화별 잔액)
- 마지막 거래 날짜 (헤더 우측)

**할 수 있는 것:**
- 은행 카드 클릭 → /accounts?bank=<id> 로 이동 (해당 은행 필터 적용)

**기대 동작:**
- 페이지 진입 시 최신 데이터를 보여준다
- 은행 카드 색상은 화면에 표시되는 순서대로 순환 적용된다 (indigo → teal → violet → amber → sky → pink)

---

## /accounts

**목적:** 전체 계좌 목록을 관리한다.

**볼 수 있는 것:**
- 계좌명, 은행, 종류(입출금/신용카드 등), 통화, 잔액, 첫 거래일, 최근 거래일, 활성 여부
- 필터 우측에 현재 필터 기준 통화별 잔액 합계

**할 수 있는 것:**
- 은행 / 활성 여부로 필터링
- 계좌 행 클릭 → 계좌 상세로 이동
- \"계좌 추가\" 버튼으로 새 계좌 생성 (계좌명, 은행, 종류, 통화)

**기대 동작:**
- 비활성 계좌는 기본적으로 숨겨진다 (필터로 볼 수 있다)
- `/accounts?bank=<id>` 진입 시 해당 은행 필터가 자동 적용된다
- 계좌 추가 폼: 은행 필터가 걸려 있으면 해당 은행이 미리 선택된 상태로 열린다

---

## /accounts/:accountId

**목적:** 특정 계좌의 거래 내역을 확인하고 거래를 추가한다.

**볼 수 있는 것:**
- 계좌 기본 정보 (이름, 은행, 종류)
- 잔액, 첫/마지막 거래일, 총 거래 수
- 거래 목록 (검토 버튼, 날짜, 가맹점, 분류, 금액, 잔액, 메모, 플래그)
- (TODO) 잔액 일간 변화 그래프 — 거래의 `balance` 필드로 구성 가능

**할 수 있는 것:**
- "거래 추가" 버튼 → 계좌 상세 페이지 인라인 섹션으로 폼 펼침 (모달 아님)
  - 안 되는 경우 모달로 fallback
- "미검토만" 토글로 미검토 거래만 필터링
- 거래 행의 원형 버튼 클릭 → 검토 완료 처리 (REST API, 단방향)
- 페이지 이동 (이전/다음, 50건씩)

**기대 동작:**
- 거래 목록은 날짜 내림차순으로 표시된다
- 분류는 한글 레이블로 표시된다 (CATEGORY_LABELS 사용)
- 검토 완료된 거래는 흐리게 표시되고 버튼이 비활성화된다
- 검토 완료 취소(toggle off)는 이 페이지에서 지원하지 않는다 (거래 상세 페이지에서 처리 예정)

### 거래 추가 폼 (Bulk Input)

**UX 구조:**
- 다이얼로그가 아닌 인라인 영역(또는 큰 모달)에 테이블 형태 행 입력
- 기본 1행, "행 추가(+)" 버튼으로 행을 추가함
- "제출" 누르면 모든 행을 한꺼번에 제출 (각 행 독립적으로 mutation 호출)
- 오류 발생한 행은 빨간 테두리 + 오류 메시지 표시, 나머지 행은 계속 제출됨
- 성공한 행은 목록에서 제거(or 회색 처리), 실패한 행만 남아 재시도 가능

**필드 (행당):**

| 필드 | 비고 |
|---|---|
| 금액 | 아래 상세 참조 |
| 날짜 | 아래 상세 참조 |
| 거래 유형(type) | 아래 상세 참조 |
| 가맹점(retailer) | 아래 상세 참조 |
| 메모(note) | 자유 텍스트 |
| 내부이체(isInternal) | 체크박스/토글 — on이면 type=TRANSFER 강제, retailer 비워짐 (수정 불가) |
| 행 삭제 버튼 | 해당 행 제거 |

**내부이체(isInternal) 토글 동작:**
- ON으로 전환 시:
  - `type` → TRANSFER 강제 설정, 필드 disabled (수정 불가)
  - `retailer` → 빈 값으로 초기화, 필드 disabled (수정 불가)
- OFF로 전환 시:
  - `type` → ETC로 리셋 (retailer 재선택 시 auto-fill 다시 동작)
  - `retailer` → 필드 다시 활성화

**금액 입력 UX:**
- 지출 = 음수, 수입 = 양수
- 부호 토글 버튼(±) 제공 — 클릭 시 부호 전환
- 숫자 입력 중 `-` 키 입력 시 부호 토글 (숫자의 일부로 처리하지 않음)
  - 예: `3` → `2` → `-` → `.` → `2` 입력 시 결과는 `-32.2`
- 경고(warning) 표시 조건:
  - 거래 유형이 지출성(EAT_OUT, GROCERY, CLOTHING, TRANSPORTATION, MEDICAL, LEISURE, SERVICE, MEMBERSHIP, HOUSING, DAILY_NECESSITY)인데 금액이 양수 → "지출 항목인데 양수입니다. 환불이라면 무시하세요"
  - 거래 유형이 수입성(INCOME, SALARY)인데 금액이 음수 → "수입 항목인데 음수입니다"
  - TRANSFER, STOCK, CASH 등은 부호 제한 없음

**날짜 입력 UX:**
- 폼 최초 열 때 기본값: 이 계좌의 마지막 transaction 날짜 (ID 기준 max → 가장 최근에 추가된 row의 날짜)
  - 오늘 날짜가 아님 — 과거 내역을 소급 입력하는 경우가 많기 때문
- 행 추가 시 기본값: 바로 위 행의 날짜 복사
- 날짜 입력 필드 옆에 **-1일 / +1일 버튼** 제공
  - 같은 날짜의 다른 거래 → 그대로 행 추가
  - 다음 날로 넘어갈 때 → +1일 클릭 한 번으로 이동
  - 전날로 소급할 때 → -1일 클릭
- 날짜는 행별로 독립적으로 관리됨 (같은 폼에서 여러 날짜 혼재 가능)

**거래 유형(type) 입력 UX:**
- 가맹점 선택 시 해당 retailer의 `category` 필드로 자동 설정됨
  - `Retailer.category`가 그 가맹점의 default transaction type
  - 파이브가이즈 선택 → type이 EAT_OUT으로 자동 채워짐
- 자동 설정 후 수동으로 변경 가능 (진리가 아님, override 허용)
- 선택지: CATEGORY_LABELS 기준 한글 표시

**가맹점(retailer) 입력 UX:**
- **Combobox (타이핑 검색 + 드롭다운)** — 이름이 영어/한국어/약자 혼재, 전체 목록 탐색 불가
  - 타이핑하면 fuzzy match로 후보 목록 필터링
  - 전체 retailer 목록은 클라이언트에서 검색 (서버 검색 아님)
- **Retailer 목록 로딩 전략: 앱 레벨 캐싱**
  - 입력창 열 때마다 재로드 하지 않음 — retailer는 자주 바뀌지 않으므로
  - 앱 초기화 시 또는 첫 번째 필요 시점에 전체 목록을 한번만 로드
  - pagination이지만 retailer 수가 제한적이므로 `first: 1000` 으로 단일 요청 처리
  - Apollo 캐시에 저장되어 이후 요청은 캐시 히트
  - **인라인 retailer 신규 생성 시**: Apollo 캐시에 새 retailer를 직접 추가 (`cache.modify`) → 재로드 없이 즉시 목록에 반영
- isInternal=true 이면 비활성
- **인라인 retailer 생성**: 검색 결과 없을 때 "새 가맹점 추가" 옵션 표시
  - 클릭 시 retailer 생성 폼이 인라인으로 열림 (모달 중첩 없이)
  - retailer 생성 폼 컴포넌트를 `/retailers` 페이지와 **공유** (중복 제거)
    - `CreateRetailerForm` 로직을 별도 컴포넌트로 추출 (`/features/retailers/create-retailer-form.tsx`)
    - retailers-page.tsx의 `CreateRetailerDialog`와 transaction 폼 모두 이를 사용
  - 생성 완료 시: 새 retailer가 자동 선택되고 type도 category로 자동 설정
  - transaction 입력 내용(금액, 날짜, 메모 등) 그대로 유지됨

---

## /transactions

**목적:** 전체 거래 내역을 검색하고 탐색한다.

**볼 수 있는 것:**
- 날짜, 계좌(계좌명 + 은행명 작은 글씨), 가맹점, 분류(한글), 금액, 메모, 플래그

**할 수 있는 것:**
- 계좌별 필터
- 날짜 범위 필터 (기본값 없음 — 전체 기간)
- 분류(TransactionCategory) 필터
- 페이지 이동 (이전/다음, 50건씩)

**기대 동작:**
- 기본값은 필터 없이 최신순으로 표시된다 (날짜 범위 미적용)
- 날짜 필터는 사용자가 직접 입력할 때만 적용된다
- 내부이체(isInternal) 거래는 별도 뱃지로 표시된다
- 지출은 빨간색, 수입은 기본색으로 표시된다
- 분류는 한글 레이블로 표시된다 (CATEGORY_LABELS 사용)

**설계 고민:**
- 가맹점별 거래 조회가 `/retailers/:retailerId`와 역할이 겹칠 수 있음
  → 가맹점 필터를 이 페이지에 추가하거나, 컴포넌트를 공유하는 방향 검토 필요

---

## /transactions/:transactionId

**목적:** 특정 거래의 상세 정보를 확인하고 수정한다.

**볼 수 있는 것 (설계안):**
- 거래 기본 정보 (날짜, 계좌, 가맹점, 분류, 금액, 잔액)
- 메모
- 플래그 (isInternal, reviewed, requiresDetail)

**할 수 있는 것 (설계안):**
- 거래 정보 수정 (분류, 가맹점, 메모, isInternal 등)
- reviewed 토글 (검토 완료 ↔ 미검토 — 여기서만 toggle off 가능)

**기대 동작 (설계안):**
- `/accounts/:accountId`, `/transactions`, `/review` 등 여러 곳에서 진입 가능
- 뒤로 가기 시 진입한 페이지로 돌아간다

**현황:** 미구현 (TODO)

---

## /categories

**목적:** 월별 지출을 카테고리별로 분석한다. 이번 달 지출 분포를 지난달 또는 평균 대비 비교하는 것이 주목적.

**볼 수 있는 것:**
- 카테고리별 지출 도넛 차트 (색상 + 범례)
- 카테고리별 지출 금액 / 비율 테이블 (지출 큰 순)
- 카테고리별 수입 테이블 (있을 경우)

**할 수 있는 것:**
- 통화(KRW/USD) 필터
- 날짜 범위 필터 (기본값: 이번달 1일 ~ 오늘)
- (TODO) 카테고리 on/off 토글 — 자동차 구매 등 일시적 대형 지출이 있는 카테고리를 해당 달에 한해 제외

**기대 동작:**
- 월 단위 집계가 기본 (이번달: 1일~오늘, 과거: 해당 월 전체)
- DB 저장 날짜 기준으로 집계 (타임존 혼재 상황에서 단순화)
- 집계에서 제외되는 거래:
  - `isInternal=true` 거래 (계좌간 이체, 환전 포함) — 소비가 아님
  - `type=STOCK` 거래 — 현금→증권 자산 이동이므로 소비 아님 (TODO: 현재 미구현)
- 전체 데이터를 클라이언트에서 집계 (100건씩 자동 로드, 날짜 범위가 넓으면 느릴 수 있음)

**설계 고민 — 수입 대비 지출:**

월별로 총 수입 vs 총 지출을 비교해서 저축률이나 흑자/적자를 파악하고 싶음.
- `/categories` 하단 또는 `/spending-trends`에 같이 배치 검토
- (TODO) 구현 위치 결정 필요

**설계 고민 — 카테고리 월별 트렌드 그래프:**

"식료품 지출이 최근 몇 달간 증가했나?" 같은 시계열 분석 필요.
- `/categories`에 추가 vs 별도 페이지(`/spending-trends` 등) 고민
- `/categories`는 단일 기간 스냅샷, 트렌드는 6~12개월 범위가 적합 → 날짜 필터 충돌 가능
- 별도 페이지 권장: 각자 독립적인 날짜 범위와 집계 로직 적용 가능
- (TODO) `/spending-trends` 페이지 설계 필요

**설계 고민 — 일시적 대형 지출 제외:**

자동차 구매처럼 일회성 대형 지출이 비율 분포를 크게 왜곡하는 문제.

| 방법 | 설명 | 상태 |
|---|---|---|
| 카테고리 토글 | 특정 카테고리를 페이지에서 on/off | 추천 단기 해결책 |
| 세션 내 거래 제외 | 상위 N개 거래를 표시하고 체크박스로 제외 | 추천 장기 해결책 |
| 금액 임계값 자동 제외 | X원 이상 자동 제외 | 기준 설정이 자의적이라 비추 |
| `exclude_from_stats` 플래그 | 백엔드 필드 추가, 거래마다 마킹 | 마킹 누락 가능성 있어 비추 |

---

## /retailers

**목적:** 가맹점별 지출을 분석하고 가맹점을 관리한다.

**볼 수 있는 것:**
- 상위 10개 가맹점 막대 차트
- 가맹점별 지출 / 수입 / 거래 수 테이블

**할 수 있는 것:**
- 통화(KRW/USD) 필터
- 월 단위 필터 (이전/다음 월 이동, 기본값: 이번 달)
- 가맹점 행 클릭 → 가맹점 상세로 이동
- "가맹점 추가" 버튼으로 새 가맹점 생성 (이름, 유형, 거래 분류)

**기대 동작:**
- 가맹점 없는 거래는 "(No Retailer)"로 묶인다
- "(No Retailer)"는 상세 페이지 링크가 없다
- 가맹점 분석에서 제외되는 거래:
  - `isInternal=true` 거래 — 계좌간 이체
  - `type=TRANSFER` 거래 — 이체성 거래
  - `type=STOCK` 거래 — 자산 이동, 소비 아님
- 가맹점 없는 거래의 주요 원인: 이체(TRANSFER), 주식(STOCK) 타입은 대개 가맹점 없이 입력됨

---

## /retailers/:retailerId

**목적:** 특정 가맹점의 거래 내역을 월별로 분석한다.

**볼 수 있는 것:**
- 가맹점명, 분류
- 총 거래 수, 총 지출, 총 수입
- 월별 지출/수입 막대 차트
- 거래 목록 (날짜, 계좌, 분류, 금액, 메모)

**할 수 있는 것:**
- 날짜 범위 필터 (기본 최근 12개월)
- 뒤로 가기 → /retailers

**기대 동작:**
- 거래 필터링은 클라이언트 사이드에서 수행한다 (GraphQL이 retailer 필터를 지원하지 않음)

---

## /income

**목적:** 연도별 급여 현황을 파악한다.

**볼 수 있는 것:**
- 연도별 총 지급액 요약 테이블 (연도, 총 급여)
- 급여 막대 차트

**할 수 있는 것:**
- 연도 행 클릭 → 연도 상세로 이동

---

## /income/:year

**목적:** 특정 연도의 급여 명세를 상세 확인하고 수정한다.

**볼 수 있는 것:**
- 합계 카드 (총 급여, 조정액, 원천징수, 공제, 실수령액)
  - 숫자 색상: 음수=빨강, 양수=기본색 (항목 종류와 무관)
- 급여 기간별 막대 차트
- 급여 항목 테이블
- 행 클릭 시 확장: payDetail / adjustmentDetail / taxDetail / deductionDetail 항목별 내역 표시

**할 수 있는 것:**
- 급여 항목 수정 (연필 아이콘 → 수정 다이얼로그)
- 급여 항목 추가 (급여 추가 버튼 → 추가 다이얼로그, 거래 연결 필요)

**기대 동작:**
- 항목 수정/추가 후 테이블이 즉시 갱신된다
- 숫자 색상은 값의 부호 기준: 음수=빨강, 양수=기본

**수정/추가 다이얼로그 — Validity Checks:**

다이얼로그의 Summary 탭에는 아래 5가지 일치 여부를 검사한다 (0.01 이하 오차 허용):

| 검사 항목 | 조건 |
|---|---|
| Gross = sum(payDetail) | grossPay와 payDetail 항목 합산이 일치하는지 |
| Adjustment = sum(adjustmentDetail) | totalAdjustment와 adjustmentDetail 항목 합산이 일치하는지 |
| Withheld = sum(taxDetail) | totalWithheld와 taxDetail 항목 합산이 일치하는지 |
| Deduction = sum(deductionDetail) | totalDeduction과 deductionDetail 항목 합산이 일치하는지 |
| Net = Gross + Adj + Withheld + Ded | netPay = grossPay + totalAdjustment + totalWithheld + totalDeduction인지 |

각 탭(Pay Detail, Adjustments, Taxes, Deductions)에서도 해당 항목의 합산 vs 총액 실시간 표시.

---

## /stocks

**목적:** 가격 기록을 남길 주식 종목 목록을 관리한다.

**개념 정리:**
- `Stock` = 가격 추적 대상 티커 (예: AAPL, 삼성전자)
- `StockPrice` = 특정 날짜의 종가 기록 (날짜, 가격만 — 계좌/수량 없음)
  - 시장 가격 추적 전용. 실제 거래 가격과 다를 수 있음 (할인/프리미엄 등)
- `StockTransaction` = 실제 매수/매도 거래 (날짜, 가격, 수량, 계좌 포함)
  - 계좌 상세 페이지에서 추가 예정
- 현금 입출금은 `/accounts/:accountId`의 `Transaction(type=STOCK)`으로 별도 기록

**⚠️ 레거시 데이터 주의:**
- 과거에는 `StockPrice` 모델이 활용되지 않았고, `StockTransaction.price`를 종가 근사값으로 사용한 기록이 있을 수 있음
- 즉, 기존 `StockTransaction` 중 일부는 실제 매수/매도가 아닌 종가 기록 목적으로 추가된 것일 수 있음
- 앞으로는 종가 기록은 `StockPrice`, 실제 거래는 `StockTransaction`으로 명확히 분리

**볼 수 있는 것:**
- 티커, 종목명, 통화
- USD / KRW 종목 수 요약 카드

**할 수 있는 것:**
- "종목 추가" 버튼으로 새 종목 생성 (종목명, 티커, 통화)
- 종목 행 클릭 → 종목 상세로 이동

---

## /stocks/:stockId

**목적:** 특정 종목의 종가 기록을 조회하고 추가한다.

**볼 수 있는 것:**
- 종목명, 티커, 통화
- 최근 가격, 최근 기록일, 총 기록 수
- 가격 추이 AreaChart (기록이 2개 이상일 때)
- 가격 기록 테이블 (날짜, 가격 — 최신순)

**할 수 있는 것:**
- 가격 기록 추가 폼 (날짜, 가격만 — 간결하게)

**기대 동작:**
- 저장 성공 시 토스트 알림 + 목록 자동 갱신
- 실제 매수/매도(`StockTransaction`) 추가는 계좌 상세 페이지에서 처리 예정

---

## 주식 거래 추가 폼 (StockTransaction)

**진입점:** 계좌 상세 페이지 (`/accounts/:accountId`) — 해당 계좌가 증권 계좌인 경우 "주식 거래 추가" 버튼 제공

**입력 필드:**

| 필드 | 설명 |
|---|---|
| 종목 (stock) | Combobox — 아래 상세 참조 |
| 날짜 (date) | 일반 거래 폼과 동일한 UX (계좌의 마지막 거래 날짜 기본값, ±1일 버튼) |
| 단가 (price) | 1주당 가격 |
| 수량 (shares) | 양수 = 매수, 음수 = 매도 |
| 총액 (amount) | 아래 상세 참조 |
| 메모 (note) | 자유 텍스트 |

**부호 규칙 (헷갈리기 쉬움 — 반드시 준수):**

| | 매수 (Buy) | 매도 (Sell) |
|---|---|---|
| `shares` | **+** (주식 취득) | **-** (주식 처분) |
| `price` | + (항상 양수, 단가) | + (항상 양수, 단가) |
| `StockTransaction.amount` | **-** (현금 유출) | **+** (현금 유입) |
| `Transaction.amount` | **-** (계좌에서 빠짐) | **+** (계좌로 들어옴) |

- `StockTransaction.amount`와 `Transaction.amount`는 **항상 같은 부호, 같은 값** (audit에서 검증)
- `shares`와 `amount`는 **항상 반대 부호**: shares+ → amount-, shares- → amount+

**총액(amount) 계산 UX:**
- `shares` 부호로 amount 부호 자동 결정: `shares > 0` → amount 음수, `shares < 0` → amount 양수
- 절대값 계산: `|price × shares|`로 자동 채움
- 수수료·소수점 차이를 반영하기 위해 사용자가 절대값 부분만 직접 수정 가능 (부호는 shares 기준 자동 유지)
- **3-필드 연동 규칙** (price, shares, amount 절대값 중 하나가 비었을 때만 자동 계산):
  - price와 shares 입력 → amount 자동 계산
  - price와 amount 입력 + shares 비어있음 → shares 자동 계산
  - shares와 amount 입력 + price 비어있음 → price 자동 계산
  - 셋 다 채워진 상태에서 하나를 수정하면 나머지를 자동으로 건드리지 않음 (무한루프 방지)
- **경고 표시**: `|price × shares| - |amount|` / `|amount| > 0.5%` 이면 "단가 × 수량과 총액 차이가 큽니다" 워닝
- **부호 불일치 경고**: shares와 amount 부호가 위 규칙에 어긋나면 즉시 경고 (저장 차단은 안 함)

**저장 시나리오 — 입력 폼에서 선택:**

| 시나리오 | 언제 | 처리 |
|---|---|---|
| A. 둘 다 신규 | 주식 거래 + 현금 내역 동시에 없는 경우 | Transaction 생성 → StockTransaction 생성 + 연결 |
| B. Transaction 이미 있음 | 은행 내역 import로 현금 거래는 기록됐는데 주식 디테일이 없는 경우 | StockTransaction만 생성, 기존 Transaction 검색해서 연결 |
| C. StockTransaction 이미 있음 | 반대로 주식 거래만 기록됐고 Transaction 링크 없는 경우 | 링크 연결 페이지에서 처리 (아래 참조) |

**시나리오 A (기본값):** 폼 하단에 "현금 거래 직접 입력" 토글 ON
- Transaction 생성 후 StockTransaction 생성 + 연결
- 두 단계 중 하나가 실패하면 오류 표시
- **고아 레코드 허용**: 당분간 연결 안 된 상태로 존재할 수 있음 — review 링크 연결 페이지에서 수동 연결, audit에서 감지

**시나리오 B:** 폼 하단 "이미 입력된 거래와 연결" 토글 ON
- 현재 계좌의 `type=STOCK` 거래 목록 검색/선택 (날짜, 금액으로 필터)
- 선택한 Transaction ID를 related_transaction으로 사용
- 총액 필드에 선택한 Transaction의 금액이 자동으로 채워짐 (수정 가능)

**링크 연결 페이지 (review 섹션 — 아래 별도 정의):**
- 시나리오 C (StockTransaction만 있고 Transaction 없음) 처리
- 미연결 항목끼리 매칭 UI 제공

**종목(stock) 입력 UX:**
- Combobox (타이핑 검색) — ticker 또는 회사명 둘 다 검색 가능
  - 예: `msft` 또는 `microsoft` 입력 → Microsoft 종목 매칭
  - 표시 형식: `MSFT — Microsoft` (ticker 앞, 이름 뒤)
  - 검색은 ticker와 name 필드 모두에 대해 case-insensitive substring 매칭 (클라이언트 필터)
- **종목 목록 로딩 전략: retailer와 동일하게 앱 레벨 캐싱**
  - 폼 열 때마다 재로드 하지 않음
  - `first: 1000` 단일 요청으로 전체 목록 로드, Apollo 캐시 재사용
  - 인라인 신규 추가 시 Apollo 캐시에 직접 추가 (`cache.modify`) → 즉시 반영
- 검색 결과 없을 때 "새 종목 추가" → 인라인 폼 (ticker, name, currency)
  - `Stock` 생성용 `CreateStockForm`을 `/stocks` 페이지의 생성 폼과 공유

**링크 네비게이션:**
- `Transaction(type=STOCK)` 상세 페이지 → 연결된 `StockTransaction` 정보 표시 및 링크
- `StockTransaction` 목록 (계좌 상세 내) → 연결된 `Transaction` 날짜/금액 표시 및 링크
- 연결 없으면 "미연결 — 연결 페이지로 이동" 버튼 표시

**audit 연계:**
- `StockTransaction`에 `related_transaction`이 없으면 audit 페이지 1-A 섹션에서 표시
- `Transaction(type=STOCK)`인데 StockTransaction 링크 없으면 동일하게 표시
- 금액/부호 불일치도 audit에서 감지

---

## /review

**목적:** 미검토 거래를 확인하고 검토 완료 처리한다. 거래 타입에 따라 검토 조건이 다르다.

**거래 타입별 검토 기준:**

| 타입 | 검토 조건 | 블록 조건 |
|---|---|---|
| 일반 (식비, 쇼핑 등) | 눈으로 확인 후 완료 | 없음 |
| Internal transfer (같은 통화) | `related_transaction` 연결 필요 | 연결 없으면 완료 불가 |
| FX 환전 (Internal, 다른 통화) | `Exchange` 레코드 연결 필요 | 연결 없으면 완료 불가 |
| Income | 금액·날짜 근접 `Salary` 레코드 연결 필요 | 연결 없으면 완료 불가 |
| Amazon | 연결된 AmazonOrder 있는지 소프트 확인 | 경고 표시 (완료 가능, 금액 오차 허용) |

**Internal transfer 매칭 규칙:**
- 같은 통화: 금액 일치 + 날짜 근접 (한국 은행은 당일만, 미국 은행은 몇 business day 허용)
- FX 환전 (다른 통화): `Exchange` 모델 사용 (`from_transaction`, `to_transaction`, `ratio_per_krw`, `exchange_type`)
  - 파는 통화 기록이 먼저, 사는 통화 기록 나중
  - 미국 시차로 날짜가 역전될 수 있음 (한국 시간 02일 출금 → 미국 시간 01일 입금). UTC 미적용, 알고만 있을 것

**Income 매칭 규칙:**
- `Salary.transaction` FK가 이 거래를 가리켜야 reviewed 가능
- 연결 없으면 블록 + `/audit` 페이지에서 추적

**Amazon 특이사항:**
- N:1 구조: 여러 AmazonOrder → 1 Transaction
- 한국 카드 사용 시: USD 결제 → 월별 KRW 청구 (환율 미확정 시점 존재)
- 완벽한 금액 reconciliation 불가 (세금 집계, 아이템별 할인 귀속 불명확, 해외 결제 수수료)
- 소프트 검증: 연결된 order 있으면 OK, 없으면 경고

**볼 수 있는 것:**
- 미검토(reviewed=false) 거래 목록, 최신순, 20건씩 페이지네이션
- 날짜, 계좌, 가맹점, 분류, 금액, 메모, 플래그(isInternal, requiresDetail)

**할 수 있는 것:**
- 거래 행 클릭 → 거래 상세 페이지로 이동 (상세 검토/편집)
- 행의 원형 버튼 → 검토 완료 (타입별 조건 충족 시에만 가능)
- "이 페이지 전체 완료" 버튼 (Promise.all 병렬 처리)
- 날짜 범위 필터 (선택사항, 기본 전체)

**기대 동작:**
- 검토 완료 처리한 거래는 즉시 목록에서 사라진다 (낙관적 업데이트)
- REST 엔드포인트: `GET /money/toggle_reviewed/<numeric_id>/`

**리뷰 UX 설계 (구현 필요):**

현재 toggle 버튼 방식은 미스클릭 위험이 높음. 타입에 따라 2-step 확인 흐름이 필요:

**Internal transfer (같은 통화) 리뷰 흐름:**
1. 해당 거래 클릭 → 상세 패널/모달 오픈
2. 후보 counterpart 자동 제안:
   - 조건: 금액 부호 반대, 절댓값 일치
   - 날짜: 한국 계좌 = 당일, 미국 계좌 = ±5 business days
3. 후보 선택 → `related_transaction` 연결
4. "검토 완료" 버튼 → 양쪽 모두 `reviewed=True`

**FX 환전 리뷰 흐름:**
1. 해당 거래 클릭 → 상세 패널/모달 오픈
2. 후보 counterpart 자동 제안:
   - 조건: 다른 통화, 부호 반대, 대략적 환율 기반 금액 근사
   - 날짜: ±2일 (시차 고려, 한국 시간 02일 출금 → 미국 시간 01일 입금 가능)
3. 후보 선택 + 환율 입력 → `Exchange` 레코드 생성
4. "검토 완료" 버튼

**Income 리뷰 흐름:**
1. 해당 거래 클릭 → 상세 패널/모달 오픈
2. 근접 날짜 + 같은 금액(net_pay)의 `Salary` 레코드 후보 제안
3. 후보 선택 → `Salary.transaction` 연결
4. "검토 완료" 버튼 (연결 없으면 비활성)

**참고 - 기존 Django 뷰 패턴:**
- `ReviewInternalTransactionView`: TRANSFER 타입 미검토 거래 별도 리스팅
- `TransactionDetailView`: detail items 합계 ≈ 거래 금액이면 자동 reviewed
- `ReviewDetailTransactionView`: `requires_detail=True` 별도 뷰
- `AmazonListView`: 연결된 AmazonOrder 없는 Amazon 거래만 필터

**TODO (미구현):**
- 타입별 2-step 리뷰 UI (현재는 단순 toggle — 미스클릭 위험)
- Internal / FX counterpart 후보 제안 로직
- Income → Salary 연결 UI
- `Exchange` 레코드 생성 mutation (GraphQL 미지원)

---

## /exchanges

**목적:** KRW ↔ USD 환전 기록 조회.

**볼 수 있는 것:**
- 총 환전 수, 평균 환율 (페이지 기준) 요약 카드
- 환전 내역 테이블: 날짜, 출금 (계좌명 + 금액), 입금 (계좌명 + 금액), 환율 (₩/USD), 구분 (ETC/BANK/WIREBARLEY/CREDITCARD)
- 20건씩 페이지네이션

**기반 모델:** `Exchange` (from_transaction, to_transaction, ratio_per_krw, exchange_type)

**TODO:**
- 환전 기록 생성 (현재 조회만 가능)
- 리뷰 페이지에서 FX 연결 시 Exchange 레코드 자동 생성

---

## /amazon

**목적:** Amazon 주문 내역을 관리하고 거래와 연결한다.

**볼 수 있는 것:**
- 총 주문 수, 거래 연결된 수, 반품 수 요약 카드
- 주문 목록 (날짜, 상품명, 연결된 거래 여부, 금액, 상태)

**할 수 있는 것:**
- "주문 추가" 버튼으로 새 주문 생성 (날짜, 상품명, 거래 연결, 반품 여부)
- 페이지 이동 (30건씩)

**기대 동작:**
- 주문 생성 시 최근 3개월 거래 목록에서 연결할 거래를 선택할 수 있다

---

## /audit (미구현)

**목적:** 연결 누락 거래 감지 + Income(Salary) 데이터 품질 검토.

### 0. 계좌 데이터 이상 (구현됨)

- 비활성 계좌 중 잔고가 0이 아닌 항목 → 클릭 시 계좌 상세로 이동

### 1. 거래 연결 감사

**볼 수 있는 것:**
- Income 거래 중 연결된 `Salary` 레코드가 없는 항목
- Internal 거래 중 `related_transaction`이 없는 항목
- FX 거래 중 `Exchange` 레코드가 없는 항목
- **Internal 거래 데이터 불일치**: `isInternal=true`인데 `type ≠ TRANSFER` 이거나 `retailer`가 있는 항목
- **주식 거래 연결 불일치**:
  - `Transaction(type=STOCK)`인데 연결된 `StockTransaction`이 없는 항목
  - `StockTransaction`인데 `related_transaction`이 없는 항목
  - 연결은 됐지만 금액 불일치: `StockTransaction.amount ≠ Transaction.amount`
  - 연결은 됐지만 부호 불일치:
    - `shares > 0` (매수)인데 `Transaction.amount > 0` (돈이 들어온 것으로 기록)
    - `shares < 0` (매도)인데 `Transaction.amount < 0` (돈이 나간 것으로 기록)
    - `StockTransaction.amount`와 `Transaction.amount` 부호가 다름

**목표:**
- 각 항목에서 바로 연결 액션 수행 가능
- 누락 건수 대시보드 표시

### 1-A. 주식 거래 ↔ Transaction 링크 연결 페이지 (미구현)

**목적:** `StockTransaction`과 `Transaction(type=STOCK)` 중 한쪽만 있고 연결이 없는 항목을 매칭해서 연결한다.

**볼 수 있는 것:**
- 좌측: `related_transaction`이 없는 `StockTransaction` 목록 (날짜, 종목, 수량, 총액)
- 우측: 연결된 `StockTransaction`이 없는 `Transaction(type=STOCK)` 목록 (날짜, 계좌, 금액)

**할 수 있는 것:**
- 날짜/금액 기준으로 자동 매칭 후보 제시 (날짜 동일 + 금액 근사)
- 수동으로 StockTransaction 하나와 Transaction 하나를 선택해서 연결
- 연결 시 금액/부호 불일치 경고 표시 (연결 차단하지는 않음)

**연결 방법 (백엔드):**
- `updateStockTransaction` mutation 필요 (현재 미구현) — `related_transaction` 필드만 업데이트

### 2. Salary 데이터 품질 감사

**문제:**
- `pay_detail`, `adjustment_detail`, `tax_detail`, `deduction_detail` JSON 필드의 항목명이 입력마다 다를 수 있음
  - 예: `"Federal Tax"` vs `"federal tax"` vs `"Federal tax "` (trailing space)
- 같은 개념(예: 세금 withheld)이 다른 키로 기록되면 월간 비교/집계 불가

**볼 수 있는 것:**
- 전체 Salary 기록에서 각 JSON 필드(pay/adjustment/tax/deduction)의 **유니크 키 목록**
- 유사하지만 다른 키 쌍 자동 감지 (대소문자 정규화, 앞뒤 공백 제거 후 중복 탐지)
  - 예: `"Federal Tax"` ↔ `"federal tax"` → 같은 키로 의심, 플래그
- 각 키가 몇 개 Salary 기록에 등장하는지 (드문 키 = 오타 가능성)

**할 수 있는 것:**
- 키 rename: 특정 키를 다른 이름으로 일괄 변경 (해당 키를 가진 모든 Salary 기록에 적용)
- Salary 금액 검증: `gross_pay - total_withheld - total_deduction + total_adjustment ≈ net_pay` 확인, 오차 있는 항목 플래그

---

## 백로그 (미구현 아이디어)

### Grocery/생필품 DetailedItem
- Amazon orders와 유사하게 영수증 항목(사과 2개 $1.5, 당근 1개 $0.8 등)을 거래에 연결
- 수기 입력 시도했으나 포기: 항목 수 과다, 영수증 아이템 코드 역추적 어려움
- **향후 AI 기능으로 구현 예정**: 영수증 이미지 → OCR → 아이템 재구성 → 자동 입력

### /spending-trends
- 카테고리별 월간 지출 트렌드 그래프 (현재 /categories는 특정 월 스냅샷)
- 여러 달을 가로축으로, 카테고리별 bar/line chart

### updateTransaction mutation
- 현재 거래 편집 저장 미지원 (백엔드 mutation 없음)
- `/transactions/:transactionId` 편집 폼은 UI만 존재
