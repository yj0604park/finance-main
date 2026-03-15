# Frontend

* http://minitwo.tail591527.ts.net:3001/management/accounts 에서 balance가 모두 0으로 나오는 문제

* http://minitwo.tail591527.ts.net:3001/management/accountDetails 을 왼쪽 사이드바로 진입시 Account not selected가 나오는데 여기서 선택할수 있어야함

* Graphql을 사용하는 view 들에서 error 케이스에 대한 처리가 미흡

* 유틸리티 함수에 대한 unit test 필요

* 기존 graphql 관련 수동으로 생성한 코드 제거 필요

* AccountState의 accountId 의 타입을 애초에 string으로 가지고 있으면 편할듯? bankId도 마찬가지

* 가능한 any 타입을 사용하지 않도록 `interface TransactionListProps {  error: any; }`

# Backend

* http://localhost:58000/money/detail_item_list 에서 ValidationError at /money/detail_item_list ["“FUNITURE” must be a subclass of <enum 'DetailItemCategory'>."] 에러 


# 데이터

* Income 세부 항목에서 `401(k)` 와 `401(K)` 가 존재. 하나로 통일 필요.

# 기능 추가

* Income 신규 입력 기능 추가
   - 날짜 등 필수 정보입력
   - Pay detail / Adjustement detail / Tax detail / Deduction detail 추가 가능할 것
   - 해당하는 Transaction 선택 가능능


# Fixed issue
* http://minitwo.tail591527.ts.net:3001/management/Income/yearDetails/2022 에서 Summary 숫자 값이 NaN으로 표기됨
  - 원인: GraphQL Decimal 응답(string/null)이 수동 타입(`Salary`) number와 불일치 → 합산 시 NaN
  - 조치: 코드젠 타입 사용 및 `NumberHelper.ToNumber`로 안전 변환
