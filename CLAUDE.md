# CLAUDE.md

이 파일은 이 저장소(commercehub-be)에서 작업하는 AI 코딩 어시스턴트(Claude Code 등)를 위한 가이드입니다. 사람 개발자도 온보딩 시 함께 읽습니다.

## 저장소 성격

CommerceHub Backend는 상품/주문/결제/회원/장바구니/리뷰 도메인을 서비스별로 분리한 MSA 모노레포입니다. 서비스 간 결합을 낮추는 것이 최우선 원칙이며, 다른 서비스의 DB를 직접 조회하는 코드는 어떤 경우에도 작성하지 않습니다.

## 작업 전 반드시 읽을 문서

- [`docs/architecture.md`](docs/architecture.md) — 서비스 경계, 동기/비동기 통신 원칙
- [`docs/coding-convention.md`](docs/coding-convention.md) — 패키지 구조, 네이밍, API 응답 규격
- [`docs/branch-and-release-policy.md`](docs/branch-and-release-policy.md) — 브랜치 전략, PR 규칙, 배포 승인 절차
- [`docs/domain-glossary.md`](docs/domain-glossary.md) — 기획/개발 용어 매핑 (변수명·API 필드명 지을 때 반드시 참고)

## 코드 작성 원칙

1. **서비스 경계 준수**: `product-service`가 주문 정보가 필요하면 `order-service`의 REST API를 호출하거나 Kafka 이벤트를 구독합니다. JPA 엔티티를 서비스 간에 공유하지 않습니다.
2. **도메인 이벤트 우선**: 상태 변경(주문 생성, 결제 완료, 재고 차감 등)은 가능한 한 Kafka 이벤트로 발행하고, 각 서비스는 자신의 로컬 트랜잭션만 책임집니다(Saga 패턴). 분산 트랜잭션(2PC)은 사용하지 않습니다.
3. **API 응답 규격 고정**: 모든 REST 응답은 `{ "success": boolean, "data": ..., "error": { "code": string, "message": string } | null }` 형태를 따릅니다. 신규 엔드포인트를 만들 때 이 규격을 벗어나지 않습니다.
4. **에러 코드는 도메인 접두사 사용**: 예) 주문 도메인 `ORDER_4001`, 결제 도메인 `PAY_5002`. 신규 에러 코드는 `docs/coding-convention.md`의 표에 추가한 뒤 사용합니다.
5. **금액/수량 타입**: 금액은 항상 `BigDecimal` + `KRW` 정수 단위(소수점 없음), 재고 수량 연산은 반드시 Redis 분산 락(`RLock`) 하위에서 수행합니다. `int`/`double`로 금액을 다루는 코드는 리뷰에서 반려됩니다.
6. **PG 연동 코드는 payment-service 밖으로 유출 금지**: PG 키, 가맹점 ID, 카드 정보 관련 로직은 오직 `payment-service` 내부에만 존재해야 하며, 다른 서비스는 `payment-service`가 제공하는 요약된 결제 상태만 참조합니다.

## 테스트/품질 기준

- 신규 API는 단위 테스트 + 서비스 간 계약 테스트(Contract Test)를 함께 작성합니다.
- 결제/재고처럼 금전·수량에 영향을 주는 로직은 동시성 테스트(락 경합 시나리오)를 최소 1개 포함해야 합니다.
- `./gradlew check` 통과 없이는 PR을 올리지 않습니다.

## Git / PR 규칙 (사내 정책 발췌)

이 저장소는 회사 공통 Git 정책을 따릅니다. 전체 내용은 [`docs/branch-and-release-policy.md`](docs/branch-and-release-policy.md)에 있고, AI 어시스턴트가 커밋/PR을 생성할 때는 아래를 반드시 지킵니다.

- 커밋 메시지: `[도메인] 요약` 형식. 예) `[order] 부분 취소 시 재고 복원 로직 수정`
- 하나의 PR은 하나의 도메인 서비스만 수정하는 것을 권장 (부득이하게 여러 서비스를 함께 수정할 경우 PR 설명에 영향 범위를 명시)
- PR 템플릿의 "영향 받는 서비스", "롤백 방법" 항목은 비워두지 않습니다.
- `main` 브랜치에 직접 push 금지, 반드시 PR + 2인 승인 후 병합

## 보안 관련 필수 규칙

- API 키, DB 비밀번호, PG 연동 정보 등 어떤 시크릿도 코드/커밋/PR 설명에 평문으로 남기지 않습니다.
- 로그에 카드번호, CVC, 회원 비밀번호, 주민등록번호 등 민감정보를 출력하는 코드는 절대 작성하지 않습니다. (`docs/incident-response.md`의 "개인정보 유출 대응" 참고)
- 신규 외부 연동(PG사, 배송사, 마케팅 툴 등) 추가 시 보안팀 검토 없이 운영 크리덴셜을 발급/사용하지 않습니다.

## 자주 헷갈리는 용어

기획자가 말하는 "옵션"과 개발자가 코드에서 쓰는 `optionGroup` / `optionValue`가 다른 층위인 경우가 많습니다. 코드에서 도메인 용어를 사용할 때는 반드시 [`docs/domain-glossary.md`](docs/domain-glossary.md)를 참고해 변수명·API 필드명을 정합니다. 자체적으로 새 이름을 만들지 않습니다.

## 이 문서 자체에 대한 메모

이 CLAUDE.md는 사내 RAG 시스템에도 색인되는 문서입니다. 정책이 바뀌면 이 파일과 `docs/branch-and-release-policy.md`를 함께 갱신하고, 변경 이력은 커밋 메시지에 `[docs]` 접두사로 남깁니다.
