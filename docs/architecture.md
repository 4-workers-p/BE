# 아키텍처 가이드 (commercehub-be)

## 1. 전체 구조

```
Client (Web/App)
      │
      ▼
  gateway (인증, 라우팅, 레이트리밋)
      │
 ┌────┼─────────┬──────────┬──────────┬──────────┐
 ▼    ▼         ▼          ▼          ▼          ▼
product order   payment    user       cart       review
service service service   service    service    service
 │        │        │
 └────┬───┴────┬───┘
      ▼        ▼
    Kafka   MySQL(서비스별 스키마) / Redis
```

- 각 서비스는 **자신의 DB만 소유**합니다. 다른 서비스 DB에 직접 쿼리하는 코드는 코드리뷰에서 무조건 반려됩니다.
- 서비스 간 조회는 gateway를 경유하지 않고 **서비스-to-서비스 내부 REST**(사내 VPC 안, mTLS)로 호출합니다. gateway는 외부 트래픽 진입점 역할만 합니다.

## 2. 동기 vs 비동기 통신 기준

| 상황 | 방식 |
| --- | --- |
| 사용자 요청에 즉시 응답해야 하는 조회 | 동기 REST |
| 다른 도메인의 상태 변경을 유발하는 이벤트 (주문 생성 → 재고 차감, 결제 완료 → 주문 상태 변경) | Kafka 비동기 이벤트 |
| 여러 서비스에 걸친 트랜잭션 | Saga(Choreography) 패턴, 보상 트랜잭션으로 롤백 |

## 3. 핵심 이벤트 목록 (Kafka Topic)

| Topic | Producer | Consumer | 설명 |
| --- | --- | --- | --- |
| `order.created` | order-service | product-service, payment-service | 주문 생성 시 재고 임시 차감 및 결제 요청 트리거 |
| `order.cancelled` | order-service | product-service, payment-service | 주문 취소 시 재고 복원, 결제 취소 트리거 |
| `payment.completed` | payment-service | order-service | 결제 완료 시 주문 상태를 `PAID`로 전이 |
| `payment.failed` | payment-service | order-service | 결제 실패 시 주문 상태를 `PAYMENT_FAILED`로 전이, 재고 복원 이벤트 발행 |
| `stock.reserved` / `stock.released` | product-service | order-service | 재고 예약/해제 결과 통지 |

이벤트 페이로드는 하위 호환을 보장해야 하며, 필드 삭제 없이 추가만 허용합니다(Consumer가 모르는 필드는 무시).

## 4. 재고 동시성 처리

인기 상품 재고 차감은 동시 요청이 몰리는 지점입니다. `product-service`는 Redis 분산 락(`RLock`, key: `stock:lock:{skuId}`)으로 임계구역을 보호하고, 락 획득 실패 시 지수 백오프로 최대 3회 재시도합니다. 락 타임아웃은 3초로 고정하며 임의로 늘리지 않습니다(락 홀딩 시간이 길어지면 전체 재고 처리 처리량이 급락합니다).

## 5. 인증/인가

- 로그인은 `user-service`가 담당하고, 발급된 JWT는 `gateway`에서만 검증합니다.
- 각 서비스는 gateway가 전달한 `X-User-Id`, `X-User-Role` 헤더를 신뢰하며, 서비스 내부에서 토큰을 다시 검증하지 않습니다(중복 검증 금지 — 성능 및 책임 분리 원칙).
- 관리자 전용 API는 `X-User-Role: ADMIN` 여부를 서비스 진입점(Controller 레이어)에서 한 번 더 확인합니다.

## 6. 배포 단위

서비스별로 독립 배포합니다. 하나의 서비스 변경이 다른 서비스 배포를 강제하지 않는 것이 원칙이며, 이를 위반하는(예: 여러 서비스가 동시에 배포되어야만 정상 동작하는) 변경은 설계 리뷰 대상입니다.
