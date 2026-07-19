# 코딩 컨벤션 (commercehub-be)

## 1. 패키지 구조 (서비스 공통)

```
com.commercehub.{service}
├── presentation/   Controller, Request/Response DTO
├── application/    UseCase/Service, 트랜잭션 경계
├── domain/         Entity, VO, 도메인 서비스, Repository 인터페이스
└── infrastructure/ Repository 구현체, 외부 API 클라이언트, Kafka Producer/Consumer
```

의존 방향은 항상 `presentation → application → domain ← infrastructure`이며, `domain`은 다른 레이어를 참조하지 않습니다.

## 2. 네이밍 규칙

- 클래스: `PascalCase`, 메서드/변수: `camelCase`, 상수: `UPPER_SNAKE_CASE`
- Controller는 `~Controller`, UseCase는 `~Service` 또는 `~UseCase`, Kafka 리스너는 `~EventListener`
- DTO는 방향에 따라 `~Request` / `~Response` 접미사를 붙입니다.
- 도메인 용어는 반드시 [`domain-glossary.md`](domain-glossary.md)에 등록된 표준 영문 표기를 사용합니다. 예) "옵션조합" → `optionCombination` (임의로 `optionSet`, `variant` 등을 새로 만들지 않음)

## 3. API 응답 규격

```json
{
  "success": true,
  "data": { },
  "error": null
}
```

실패 시:

```json
{
  "success": false,
  "data": null,
  "error": { "code": "ORDER_4001", "message": "재고가 부족합니다" }
}
```

- HTTP 상태 코드와 `error.code`를 이중으로 관리합니다. HTTP 상태는 트랜스포트 레벨 의미(400/404/409/500)만 나타내고, 세부 원인은 `error.code`로 구분합니다.
- 목록 조회는 페이지네이션 정보를 `data.pageInfo`에 포함합니다: `{ "page": 0, "size": 20, "totalElements": 134, "hasNext": true }`

## 4. 에러 코드 표 (발췌)

| 코드 | 도메인 | 의미 |
| --- | --- | --- |
| `ORDER_4001` | 주문 | 재고 부족으로 주문 생성 불가 |
| `ORDER_4002` | 주문 | 이미 취소/환불된 주문 |
| `PAY_5001` | 결제 | PG사 응답 타임아웃 |
| `PAY_5002` | 결제 | 결제 승인 거절 |
| `USER_4101` | 회원 | 이미 사용 중인 이메일 |
| `CART_4201` | 장바구니 | 품절 상품 담기 시도 |

신규 에러 코드 추가 시 이 표를 갱신하는 PR을 함께 올립니다.

## 5. 로깅 규칙

- 구조화 로깅(JSON) 사용, 필수 필드: `traceId`, `service`, `level`, `message`
- 금지: 카드번호, CVC, 비밀번호, 주민등록번호, 전체 주소 등 개인정보/민감정보 로그 출력
- `traceId`는 gateway가 발급해 `X-Trace-Id` 헤더로 전파하며, 모든 서비스는 이를 로그에 포함해야 장애 추적이 가능합니다.

## 6. 테스트 규칙

- 단위 테스트: JUnit5 + Mockito, `application` 레이어 중심
- 통합 테스트: Testcontainers(MySQL, Kafka)로 실제 인프라와 유사한 환경에서 검증
- PG 연동처럼 외부 의존이 있는 로직은 반드시 Mock 서버(WireMock)로 대체 테스트를 작성하고, 실제 PG 샌드박스 호출은 CI에서 실행하지 않습니다.

## 7. 코드 리뷰 체크리스트 (발췌)

- [ ] 금액/수량 연산에 `BigDecimal`/락을 올바르게 사용했는가
- [ ] 서비스 경계를 넘는 직접 DB 접근이 없는가
- [ ] 신규 API가 응답 규격을 따르는가
- [ ] 민감정보 로깅이 없는가
- [ ] 도메인 용어가 용어집과 일치하는가
