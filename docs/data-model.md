# 데이터 모델 정의

> 백엔드 Django 모델 구조와 관계를 정리한 문서.
> Claude에게 스키마 기반 작업을 지시할 때 이 문서를 참고 기준으로 사용한다.

---

## 핵심 모델 관계도

```
Bank ──< Account ──< Transaction >── Retailer
                         │
                    TransactionDetail >── DetailItem
                         │
                    AmazonOrder (transaction FK)
                    StockTransaction (related_transaction FK)
                    Salary (transaction FK)
                    Exchange (from_transaction / to_transaction FK)
```

---

## Bank

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| name | CharField | 은행명 |

---

## Account

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| bank | FK → Bank | |
| name | CharField | 계좌명 |
| alias | CharField (nullable) | 별칭 |
| type | AccountType | 종류 |
| amount | Decimal | 현재 잔액 |
| currency | CurrencyType | KRW / USD |
| is_active | Boolean | 활성 여부 (기본 true) |
| last_transaction | DateField (nullable) | 마지막 거래일 |
| first_transaction | DateField (nullable) | 첫 거래일 |
| first_added | Boolean | 최초 등록 여부 |
| last_update | DateTimeField (nullable) | 마지막 업데이트 |

**AccountType 값:**
- `CHECKING_ACCOUNT` 입출금
- `SAVINGS_ACCOUNT` 저금
- `INSTALLMENT_SAVING` 적금
- `TIME_DEPOSIT` 예금
- `CREDIT_CARD` 신용카드
- `STOCK` 주식
- `LOAN` 대출

---

## Transaction

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | 거래일 |
| account | FK → Account | |
| retailer | FK → Retailer (nullable) | 가맹점 |
| amount | Decimal | 금액 (음수 = 지출, 양수 = 수입) |
| balance | Decimal (nullable) | 거래 후 잔액 |
| type | TransactionCategory | 분류 |
| note | TextField (nullable) | 메모 |
| is_internal | Boolean | 내부이체 여부 |
| requires_detail | Boolean | 세부 내역 필요 여부 |
| reviewed | Boolean | 검토 완료 여부 |
| related_transaction | FK → Transaction (self, nullable) | 연관 거래 |
| created_at | DateTimeField | |
| updated_at | DateTimeField | |

**TransactionCategory 값:**
- `EAT_OUT` 외식
- `GROCERY` 식료품
- `CLOTHING` 의류
- `TRANSPORTATION` 교통/주유
- `MEDICAL` 의료
- `LEISURE` 여가
- `SERVICE` 서비스
- `MEMBERSHIP` 멤버십
- `HOUSING` 주거
- `DAILY_NECESSITY` 생필품
- `INCOME` 수입
- `TRANSFER` 이체
- `STOCK` 주식
- `CASH` 현금
- `PRESENT` 선물
- `PARENTING` 육아
- `INTEREST` 이자
- `ETC` 기타

> **주의**: GraphQL에서 `transactionRelay`의 `TransactionFilter`는 `id`, `date`, `account` 필터만 지원. `retailer`나 `reviewed` 필터 없음.

---

## Retailer

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| name | CharField | 가맹점명 |
| type | RetailerType | 유형 |
| category | TransactionCategory | 기본 거래 분류 |

**RetailerType 값:** `ETC`, `STORE`, `PERSON`, `BANK`, `SERVICE`, `INCOME`, `RESTAURANT`

---

## AmountSnapshot

대시보드 잔액 추이 차트에 사용되는 일별 스냅샷.

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | 스냅샷 날짜 |
| amount | Decimal | 총 잔액 |
| currency | CurrencyType | KRW / USD |
| summary | JSONField (nullable) | 추가 요약 데이터 |

---

## Stock

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| name | CharField | 종목명 |
| ticker | CharField (nullable) | 티커 |
| currency | CurrencyType | KRW / USD |

---

## StockTransaction

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | 거래일 |
| stock | FK → Stock | |
| account | FK → Account | |
| related_transaction | FK → Transaction (nullable) | 연관 거래 |
| price | Decimal | 주당 가격 |
| shares | Decimal (4 decimal) | 수량 (음수 = 매도) |
| amount | Decimal | 총 금액 |
| balance | Decimal (nullable) | 보유 수량 누적 |
| note | TextField (nullable) | 메모 |

> **주의**: GraphQL에 `stockTransactionRelay` 쿼리 없음. `createStockTransaction` 뮤테이션만 존재.

---

## Salary

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | 급여일 |
| transaction | FK → Transaction | 연관 거래 |
| currency | CurrencyType | KRW / USD |
| gross_pay | Decimal | 총 급여 |
| total_adjustment | Decimal | 조정액 |
| total_withheld | Decimal | 원천징수 |
| total_deduction | Decimal | 공제 |
| net_pay | Decimal | 실수령액 |
| pay_detail | JSONField | 급여 항목 상세 |
| adjustment_detail | JSONField | 조정 항목 상세 |
| tax_detail | JSONField | 세금 항목 상세 |
| deduction_detail | JSONField | 공제 항목 상세 |

---

## W2

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | |
| year | IntegerField | 연도 |
| currency | CurrencyType | |
| wages | Decimal | 임금 |
| income_tax | Decimal | 소득세 |
| social_security_wages / tax | Decimal | 사회보장 |
| medicare_wages / tax | Decimal | 메디케어 |
| box_12 | JSONField (nullable) | W2 박스 12 |
| box_14 | CharField (nullable) | W2 박스 14 |

---

## AmazonOrder

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | 주문일 |
| item | TextField | 상품명 |
| is_returned | Boolean | 반품 여부 |
| transaction | FK → Transaction (nullable) | 연결된 거래 |
| return_transaction | FK → Transaction (nullable) | 반품 연결 거래 |

---

## Exchange

환전 내역.

| 필드 | 타입 | 설명 |
|------|------|------|
| id | int (PK) | |
| date | DateField | |
| from_transaction | FK → Transaction | 원화 출금 거래 |
| to_transaction | FK → Transaction | 외화 입금 거래 |
| from_amount / to_amount | Decimal | 환전 금액 |
| from_currency / to_currency | CurrencyType | |
| ratio_per_krw | Decimal (nullable) | KRW 기준 환율 |
| exchange_type | ExchangeType | `ETC`, `BANK`, `WIREBARLEY`, `CREDITCARD` |

---

## TransactionDetail / DetailItem

거래 세부 내역 (가맹점별 구매 품목). 현재 프론트엔드에서 미구현.

- **TransactionDetail**: Transaction ↔ DetailItem 연결, 수량(`count`), 금액(`amount`), 메모
- **DetailItem**: 품목명, 카테고리(`DetailItemCategory`)

---

## GraphQL Global ID

모든 노드의 `id`는 base64로 인코딩된 글로벌 ID.
실제 DB PK가 필요할 때: `atob(id).split(":")[1]`
