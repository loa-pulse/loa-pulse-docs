# LoaPulse Docs

> 로스트아크 거래소 시세를 실시간으로 추적하고, 원하는 가격이 되면 디스코드로 알려주는 서비스의 기획·설계 문서

## 레포지토리

| 레포 | 내용 |
|---|---|
| [loa-pulse-docs](https://github.com/loa-pulse/loa-pulse-docs) | 기획·설계 문서 (이 레포) |
| [loa-pulse-backend](https://github.com/loa-pulse/loa-pulse-backend) | 백엔드 (Java 21, Spring Boot) |
| [loa-pulse-frontend](https://github.com/loa-pulse/loa-pulse-frontend) | 프론트엔드 (Vite, React, TypeScript) |

## 기획 (`planning/`)

| 문서 | 내용 |
|---|---|
| [00-overview.md](planning/00-overview.md) | 개요 |
| [01-glossary.md](planning/01-glossary.md) | 로스트아크 도메인 용어집 |
| [02-features.md](planning/02-features.md) | 기능 정의와 우선순위 |
| [03-screens-flows.md](planning/03-screens-flows.md) | 화면 구성과 사용자 흐름 |
| [04-policies.md](planning/04-policies.md) | 서비스 정책 |
| [05-milestones.md](planning/05-milestones.md) | 마일스톤, 성공 지표, 리스크 |
| [06-reference-analysis.md](planning/06-reference-analysis.md) | 참고 서비스 분석 |

## 설계 (`design/`)

| 문서 | 내용 |
|---|---|
| [00-architecture.md](design/00-architecture.md) | 아키텍처 개요 |
| [01-data-flow.md](design/01-data-flow.md) | 데이터 흐름과 이벤트 |
| [02-data-model.md](design/02-data-model.md) | 데이터 모델과 시계열 보관 정책 |
| [03-operations.md](design/03-operations.md) | 장애 처리와 관측성 |
| [04-testing.md](design/04-testing.md) | 테스트 전략 |
| [decisions.md](design/decisions.md) | 기술 결정 기록 |

### 모듈 (`design/modules/`)

[collector](design/modules/collector.md) · [market](design/modules/market.md) · [realtime](design/modules/realtime.md) · [alert](design/modules/alert.md) · [user](design/modules/user.md)
