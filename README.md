# CommerceHub Backend (commercehub-be)

CommerceHub는 자사 이커머스 플랫폼입니다. 이 저장소는 상품/주문/결제/회원/장바구니/리뷰 도메인을 담당하는 백엔드 MSA(Microservice Architecture) 모노레포입니다.

> ⚠️ 이 저장소는 사내 RAG(검색 증강 생성) 시스템 테스트를 위한 **가상 프로젝트**입니다. 실제 서비스 코드가 아니라 문서 구조와 컨벤션을 검증하기 위한 목데이터이며, 실서비스 배포 정보나 실제 크리덴셜을 포함하지 않습니다.

## 1. 프로젝트 개요

| 항목 | 내용 |
| --- | --- |
| 프로젝트명 | CommerceHub Backend |
| 목적 | 자사몰(직영 이커머스) 주문/결제/상품/회원 API 제공 |
| 운영 조직 | 커머스플랫폼팀 (백엔드 파트) |
| 저장소 유형 | MSA 모노레포 (services/ 하위 서비스별 모듈) |
| 배포 환경 | AWS EKS (Kubernetes), ap-northeast-2 |
| 문서 갱신 주기 | 스프린트 종료 시 (2주 1회) README/CLAUDE.md 리뷰 |

## 2. 기술 스택

- **언어/프레임워크**: Java 17, Spring Boot 3.3, Spring Cloud Gateway, Spring Data JPA, Spring Kafka
- **DB**: MySQL 8.0 (서비스별 스키마 분리), Redis 7 (세션/캐시/재고 락)
- **메시징**: Kafka (주문-결제-재고 이벤트 연동)
- **인프라**: Docker, Kubernetes(EKS), ArgoCD(GitOps 배포), Terraform(IaC)
- **CI/CD**: GitHub Actions → ECR 이미지 빌드 → ArgoCD 동기화
- **관측**: Prometheus/Grafana, Sentry(에러 트래킹), ELK(로그)

## 3. 서비스 구성

```
services/
├── gateway/            API Gateway, 인증 토큰 검증, 라우팅
├── product-service/    상품/카테고리/옵션/재고
├── order-service/      주문 생성, 주문 상태 관리
├── payment-service/    PG 연동, 결제/취소/환불
├── user-service/       회원, 인증, 등급/포인트
├── cart-service/       장바구니, 최근 본 상품
└── review-service/     리뷰, 평점, 신고
```

서비스 간 통신은 원칙적으로 **동기 호출(REST)은 gateway 경유 조회성 트래픽에만 사용**하고, 도메인 간 상태 변경은 Kafka 이벤트로 비동기 연동합니다. 자세한 아키텍처는 [`docs/architecture.md`](docs/architecture.md)를 참고하세요.

## 4. 로컬 개발 환경 구성

```bash
# 1) 인프라 의존성 기동 (MySQL, Redis, Kafka)
docker compose -f infra/docker-compose.local.yml up -d

# 2) 서비스별 실행 (예: order-service)
./gradlew :services:order-service:bootRun --args='--spring.profiles.active=local'

# 3) 전체 서비스 동시 실행
./scripts/run-all-local.sh
```

로컬 접속 정보:

| 서비스 | 포트 | 비고 |
| --- | --- | --- |
| gateway | 8080 | 로컬 진입점 |
| product-service | 8081 | |
| order-service | 8082 | |
| payment-service | 8083 | PG는 샌드박스 모드 고정 |
| user-service | 8084 | |
| cart-service | 8085 | |
| review-service | 8086 | |
| MySQL | 3306 | root/local1234 (로컬 전용, 운영과 다름) |
| Redis | 6379 | |
| Kafka | 9092 | |

## 5. 환경 변수

실제 값은 절대 저장소에 커밋하지 않으며, AWS Secrets Manager → Kubernetes Secret으로 주입됩니다. 로컬은 `.env.local.example`을 복사해 사용합니다.

| 변수명 | 설명 |
| --- | --- |
| `DB_URL`, `DB_USER`, `DB_PASSWORD` | 서비스별 MySQL 접속 정보 |
| `REDIS_HOST`, `REDIS_PORT` | 캐시/락 서버 |
| `KAFKA_BOOTSTRAP_SERVERS` | 이벤트 브로커 |
| `PG_API_KEY`, `PG_MERCHANT_ID` | PG사 연동키 (샌드박스는 팀 노션 참고) |
| `JWT_SECRET` | 게이트웨이 토큰 서명 키 |

## 6. API 문서

- 서비스별 OpenAPI 스펙: `/v3/api-docs` (서비스 기동 후 `http://localhost:{port}/swagger-ui`)
- 사내 API 게이트웨이 통합 문서: Notion `커머스플랫폼 > API 명세` (링크는 팀 채널 상단 고정)
- API 응답 규격, 에러 코드 정책은 [`docs/coding-convention.md`](docs/coding-convention.md) 참고

## 7. 브랜치/배포 정책 요약

자세한 내용은 [`docs/branch-and-release-policy.md`](docs/branch-and-release-policy.md)에 있습니다. 요약:

- 브랜치 전략: `main`(운영) / `develop`(스테이징) / `feature/*`, `fix/*`, `hotfix/*`
- PR 승인: 코드오너 1인 + 도메인 담당자 1인, 총 2 approve 필요
- 운영 배포는 배포 캘린더 등록 후 오후 2~5시 사이에만 진행 (금요일 배포 지양)

## 8. 도메인 용어집

기획/디자인/개발 간 동일 용어를 다르게 쓰는 경우가 많아 혼선이 잦았습니다. 전체 용어집은 [`docs/domain-glossary.md`](docs/domain-glossary.md)에서 관리하며, 신규 용어 추가 시 PR로 반영합니다.

## 9. 담당자

| 영역 | 담당 |
| --- | --- |
| 백엔드 아키텍처 총괄 | 커머스플랫폼팀 백엔드 파트장 |
| 결제/PG 연동 | 결제 도메인 담당자 |
| 인프라/배포 | DevOps 파트 |
| 문의 채널 | Slack `#commercehub-be` |

## 10. 관련 문서

- [`CLAUDE.md`](CLAUDE.md) — AI 코딩 어시스턴트용 규칙 및 아키텍처 가이드
- [`docs/architecture.md`](docs/architecture.md)
- [`docs/coding-convention.md`](docs/coding-convention.md)
- [`docs/branch-and-release-policy.md`](docs/branch-and-release-policy.md)
- [`docs/domain-glossary.md`](docs/domain-glossary.md)
- [`docs/onboarding.md`](docs/onboarding.md)
- [`docs/incident-response.md`](docs/incident-response.md)
- [`docs/supabase-sync.md`](docs/supabase-sync.md) — 이 저장소 문서가 사내 RAG로 자동 동기화되는 방식
