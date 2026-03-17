확인해보면 좋은 점
TransactionCategory와 SALARY
104줄 근처에 “수입성(INCOME, SALARY)”라고 되어 있는데, CLAUDE.md·choices.py 기준으로는 카테고리 enum에 INCOME만 있고 SALARY는 없을 수 있습니다.

의도: Income 타입을 부르는 다른 이름이라면 용어를 통일하거나,
실제로 SALARY가 있다면 코드와 맞는지 한 번만 확인하는 것이 좋습니다.
/transactions 필터 vs GraphQL
문서에는 “분류(TransactionCategory) 필터”가 가능하다고 되어 있는데, CLAUDE.md에는 TransactionFilter가 id, date, account만 지원한다고 되어 있습니다.

현재 백엔드/스키마에 category 필터가 없다면, 문서를 “클라이언트 필터” 또는 “미구현” 등으로 조정하는 게 나중에 혼동을 줄입니다.
retailer first: 1000
137줄에 “first: 1000 으로 단일 요청”이라고 되어 있고, CLAUDE.md에는 first 최대 100이라고 되어 있습니다.

실제로 relay limit이 100이면 1000은 불가이므로, “100씩 여러 번 요청” 또는 “백엔드 제한에 맞춤” 등으로 문서를 실제 동작에 맞추는 편이 좋습니다.
/categories의 type=STOCK 제외
216줄에 “type=STOCK 거래 — 소비 아님 (TODO: 현재 미구현)”이라고 되어 있어, “설계는 되어 있으나 아직 적용 전”이라는 점이 명확합니다. 나중에 구현 시 이 문구를 “구현됨”으로 바꾸기만 하면 됩니다.

