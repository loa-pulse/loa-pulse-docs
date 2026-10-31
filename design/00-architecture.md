# 00. 아키텍처 개요

> LoaPulse 기술 설계 · 작성일 2026-10-07
> 기획은 [../planning/](../planning/00-overview.md)을 따른다. 결정의 이유와 포기한 대안은 [decisions.md](decisions.md)에 남긴다.

## 설계 진행 상태

| 섹션 | 상태 |
|---|---|
| 1. 아키텍처 개요 | **확정** (이 문서) |
| 2. 모듈별 책임과 도메인 모델 | **확정** · [collector](modules/collector.md), [alert](modules/alert.md), [market](modules/market.md), [user](modules/user.md), [realtime](modules/realtime.md) |
| 3. 데이터 흐름과 이벤트 | **확정** · [01-data-flow.md](01-data-flow.md) |
| 4. 데이터 모델과 시계열 보관 정책 | **확정** · [02-data-model.md](02-data-model.md) |
| 5. 장애 처리와 관측성 | **확정** · [03-operations.md](03-operations.md) |
| 6. 테스트 전략 | **확정** · [04-testing.md](04-testing.md) |
| 7. 기술 결정 기록과 미결정 사항 | [decisions.md](decisions.md) (D1~D31), 미결정 사항은 이 문서 끝 |

## 기술 스택

| 영역 | 선택 |
|---|---|
| 백엔드 | Java 21, Spring Boot 4.x, Gradle, Flyway, Testcontainers |
| 로그인 | Spring Security OAuth2 Client (디스코드), Redis 세션 |
| 주 DB | PostgreSQL |
| 캐시·메시지 | Redis |
| 브라우저 실시간 전달 | SSE (Server-Sent Events) |
| API 문서 | springdoc (OpenAPI) |
| 프론트엔드 | Vite, React, TypeScript, shadcn/ui, TanStack Query, lightweight-charts |
| 프론트엔드 보조 도구 | pnpm, React Router, Tailwind CSS, ESLint, Prettier |
| 프론트엔드 API 타입 | 백엔드 OpenAPI 문서에서 자동 생성 (`openapi-typescript` 등) |
| 디스코드 연동 | 백엔드에서 디스코드 REST API로 직접 DM 발송 (상시 연결하는 봇 프로세스 없음) |
| 배포 | 프론트엔드 Vercel, 백엔드 홈서버(Cloudflare Tunnel), 도메인 Cloudflare ([D28](decisions.md#d28-프론트엔드는-vercel-백엔드는-홈서버에-배포하고-cloudflare-도메인의-서브도메인으로-나눈다)) |

## 저장소 구성

```
~/develop/loa-pulse/            일반 폴더 (git 저장소 아님)
├── docs/                       loa-pulse-docs 저장소 (기획, 설계 문서)
├── loa-pulse-backend/          loa-pulse-backend 저장소
└── loa-pulse-frontend/         loa-pulse-frontend 저장소
```

| 저장소 | 내용 |
|---|---|
| [loa-pulse-docs](https://github.com/loa-pulse/loa-pulse-docs) | 기획, 설계 문서 |
| [loa-pulse-backend](https://github.com/loa-pulse/loa-pulse-backend) | 백엔드 |
| [loa-pulse-frontend](https://github.com/loa-pulse/loa-pulse-frontend) | 프론트엔드 |

브랜치 전략과 머지 방식은 [D25](decisions.md#d25-브랜치-전략은-github-flow로-하고-pr은-머지-커밋으로-합친다)를 따른다.

## 시스템 구성도

```
                     ┌──────────────────────────────────────┐
  로스트아크 Open API ◀─┤  loa-pulse-backend (모듈러 모놀리스)     │
  (분당 100회)         │                                      │
                     │  ┌───────────┐  MarketDataSink (호출)   │
                     │  │ collector │ ─────────┐              │
                     │  │ (리더만)   │          ▼              │
                     │  └───────────┘    ┌──────────┐         │
                     │                   │  market  │         │
                     │                   └────┬─────┘         │
                     │             PriceUpdated (이벤트)       │
                     │              ┌─────────┴────────┐      │
                     │              ▼                  ▼      │
  디스코드 API ◀────────┤       ┌──────────┐       ┌──────────┐  │
  (OAuth, DM)        │       │  alert   │       │ realtime │  │
                     │       └────┬─────┘       └────┬─────┘  │
                     │            ▼ 조회              │ SSE    │
                     │       ┌──────────┐            │        │
                     │       │   user   │            │        │
                     │       └──────────┘            │        │
                     └──────────┬────────────────────┼────────┘
                                │                    │
                    ┌───────────┴──────┐             ▼
                    ▼                  ▼     ┌───────────────────┐
             ┌────────────┐     ┌────────┐   │ loa-pulse-frontend │
             │ PostgreSQL │     │ Redis  │   │ (Vite + React SPA) │
             └────────────┘     └────────┘   └───────────────────┘
```

## 배포 단위와 다중 인스턴스

- 백엔드는 **애플리케이션 1개(모듈러 모놀리스)**로 배포한다. 수집기를 별도 서비스로 분리하는 것은 계획에서 제외한다.
- **M1~M3**: 인스턴스 1개로 운영한다.
- **M4 이후**: 인스턴스를 여러 개로 늘리고 **리더 선출**을 붙인다. 리더로 뽑힌 인스턴스 1개만 수집하고, 리더가 죽거나 재배포되면 다른 인스턴스가 이어받는다.
- 리더 선출 방식(DB 락, Redis 락, k8s Lease 등)은 배포 환경을 확정한 뒤 고른다.
- 부하 테스트에서 "API 서버 부하가 수집에 영향을 준다"는 결과가 나오면 그때 수집기 분리를 다시 검토한다.

## 모듈 구성

| 모듈 | 역할 | 종류 | DDD 적용 강도 |
|---|---|---|---|
| collector | 무엇을 언제 수집할지 결정하고 API를 호출 (등급제, 호출 예산) | 핵심 도메인 | 충실하게 (애그리거트, 값 객체, 도메인 서비스) |
| market | 시세 저장, 조회, 롤업, 급등락 랭킹 계산 | 지원 | 가볍게 (계층 구조 중심) |
| alert | 목표가·급변동 알림의 조건 평가, 발송, 상태 관리 | 핵심 도메인 | 충실하게 |
| user | 디스코드 로그인, 프로필, 관심 아이템 | 지원 | 가볍게 |
| realtime | SSE 연결 관리, 인스턴스 간 전달 | 기술 지원 | 해당 없음 |
| shared | 식별자와 값 객체만 둠 (로직 없음) | 공유 커널 | 해당 없음 |

## 모듈 간 의존 방향

[D24](decisions.md#d24-모듈-의존-방향을-고정하고-다른-모듈은-api-패키지만-참조한다)

```
           collector           (가장 위. 다른 업무 모듈에 의존하지 않음)
          ▲    ▲    ▲
     market   user  │
       ▲  ▲    ▲    │
       │  └─ alert ─┘
       │      ▲
       └── realtime            (가장 아래. 아무도 realtime에 의존하지 않음)

  모든 모듈 ──▶ shared
```

- 화살표는 "의존한다"는 뜻이다. **순환은 허용하지 않는다.**
- user는 업무 모듈(alert)에 의존하지 않는다.

### 인터페이스를 누가 정의하는가

| 경우 | 정의하는 쪽 | 예 |
|---|---|---|
| 아래쪽 모듈이 위쪽 모듈을 호출 | **위쪽이 공개 API를 정의하고 구현** | alert → user의 `UserQuery`, realtime → market의 `LatestPriceQuery` |
| 위쪽 모듈이 아래쪽 모듈을 호출 | **위쪽이 인터페이스를 정의하고 아래쪽이 구현** (의존성 역전) | collector의 `MarketDataSink`, `DemandSignalPort` / user의 `UserDeletionHook` / alert의 `NotificationSender` |

어느 경우든 **의존은 항상 아래쪽 → 위쪽**이 된다.

### 패키지 구조

[D31](decisions.md#d31-모듈-안의-계층-의존-방향을-고정하고-jpa-엔티티-위치는-모듈의-ddd-적용-강도에-따라-정한다)

모듈마다 같은 구조를 둔다.

```
com.loapulse.<모듈>
 ├── api             다른 모듈이 볼 수 있는 유일한 패키지: 공개 API, 의존성 역전용 인터페이스, 이벤트
 ├── web             컨트롤러, 요청·응답 DTO
 ├── application     유스케이스 서비스, 이벤트 리스너, 예약 작업, 다른 모듈의 api 인터페이스 구현
 ├── domain          애그리거트, 값 객체, 도메인 서비스, 내부 포트
 └── infrastructure  내부 포트 구현: JPA, 외부 API 클라이언트, Redis. 외부 연결 설정
```

#### 모듈 안의 의존 방향

```
web ──▶ application ──▶ domain ◀── infrastructure
              │                    (domain의 포트를 구현)
              └──▶ 자기 모듈의 api, 다른 모듈의 api
```

| 계층 | 참조할 수 있는 것 | 참조하면 안 되는 것 |
|---|---|---|
| `api` | shared | 자기 모듈의 다른 계층 전부. 다른 모듈에 보여주는 것이라 안쪽 타입이 새면 안 된다 |
| `web` | application, domain, 자기 모듈의 api | infrastructure |
| `application` | domain, 자기 모듈의 api, 다른 모듈의 api | web, infrastructure |
| `domain` | shared | web, application, infrastructure, api |
| `infrastructure` | domain | web, application |

- 모든 계층은 shared를 참조할 수 있다.
- `application`이 저장소나 외부 API를 쓸 때는 `domain`의 포트(인터페이스)를 통한다. 구현은 `infrastructure`에 두고 스프링이 주입한다.

#### JPA 엔티티의 위치

모듈의 DDD 적용 강도(아래 [모듈 구성](#모듈-구성))에 따라 다르다.

| 모듈 | `domain` | 저장 |
|---|---|---|
| collector, alert (충실하게) | **순수 자바.** JPA, 스프링을 참조하지 않는다. 값 객체는 record로 둔다 | `domain`의 저장소 포트를 `infrastructure`가 구현한다. JPA 엔티티는 `infrastructure`에 따로 두고 도메인 객체와 매핑한다 |
| market, user (가볍게) | JPA 엔티티가 곧 도메인 객체다. JPA와 Spring Data 저장소 인터페이스를 `domain`에 둔다 | Spring Data가 구현한다. 대량 저장(시세 원본 등)처럼 JPA가 맞지 않는 곳은 `infrastructure`에 JDBC 구현을 둔다 |
| realtime, shared | 해당 없음 | 해당 없음 |

#### 세부 위치

| 대상 | 위치 | 예 |
|---|---|---|
| 예약 작업 (`@Scheduled`) | `application` | 수집 실행, 롤업, 파티션 관리 |
| 다른 모듈의 api 인터페이스 구현 | `application` | market의 `MarketDataSink` 구현 |
| 외부 연결 설정 (`@Configuration`) | `infrastructure` | 로스트아크 API `RestClient` |
| `infrastructure`의 하위 패키지 | 외부 시스템이나 저장 기술별로 나눈다 | `infrastructure.lostark`, `infrastructure.persistence`, `infrastructure.redis`, `infrastructure.discord` |

## 모듈 간 통신 규칙

> **기본은 조회(직접 호출)로 하고, "받는 쪽이 여럿이고 보내는 쪽이 누가 받는지 몰라도 될 때"만 이벤트를 쓴다.** 인터페이스를 어느 쪽이 정의할지는 위의 의존 방향으로 정한다.

| 흐름 | 방식 | 인터페이스 위치 | 이유 |
|---|---|---|---|
| collector → market (수집 결과 저장) | 호출 | `collector.api.MarketDataSink` (market이 구현) | 받는 쪽이 하나다. collector가 저장 성공 여부를 알아야 다음 수집 시각을 정하고 실패 시 다시 시도할 수 있다 ([D8](decisions.md#d8-collector--market은-인터페이스-호출로-한다)) |
| collector → market (아이템별 1시간 변동률) | 조회 | `collector.api.PriceVolatilityPort` (market이 구현) | 등급 재계산 때 필요한 값을 묻는 질문이다 |
| collector → alert, user (활성 알림 수, 관심 수) | 조회 | `collector.api.DemandSignalPort` (alert, user가 구현) | 이벤트로 하면 유실 시 숫자가 영구히 어긋난다. 등급은 어차피 주기적으로 재계산한다 |
| market → alert, realtime (시세 갱신) | **이벤트** | `market.api.PriceUpdated` | 받는 쪽이 여럿이다. 이벤트 하나를 놓쳐도 다음 갱신이 최신 상태를 다시 전달한다 |
| alert → user (알림 받을 사용자 정보, DM 결과 반영) | 조회·호출 | `user.api.UserQuery`, `user.api.DmStatusRecorder` | 바로 답이 필요한 질문이다 |
| user → alert (회원 탈퇴 시 알림 데이터 삭제) | 호출 | `user.api.UserDeletionHook` (alert가 구현) | 유실되면 개인정보가 남는다. 한 트랜잭션으로 묶는다 (D19) |
| alert → realtime (웹 토스트) | 호출 | `alert.api.NotificationSender` (realtime이 구현) | 발송 워커가 채널별로 발송한다 |
| realtime → market, collector (`freshness`) | 조회 | `market.api.LatestPriceQuery`, `collector.api.CollectionStatusQuery` | 다른 모듈의 Redis 키를 직접 읽지 않는다 (D18) |

- **DM 발송의 신뢰성은 이벤트가 아니라 alert 모듈의 DB 상태로 보장한다.** `Notification`의 상태(`PENDING` → `SENT`/`FAILED`)와 발송 워커의 선점으로 기획 4.2 정책(1번 생성, 실패 원인별 재시도, 실패 기록)을 지킨다.

## 모듈 경계 검사

경계는 **ArchUnit**으로 테스트한다. Spring Modulith는 쓰지 않는다.

- 다른 모듈은 `api` 패키지만 참조할 수 있다.
- 의존은 위의 의존 방향 그림을 따른다. 그림에 없는 의존은 금지한다.
- 모듈 사이에 순환 의존이 없다.
- shared는 다른 어떤 모듈도 참조하지 않는다.

## Redis 사용 범위

| 역할 | Redis 사용 | 도입 시점 |
|---|---|---|
| 최신 시세 캐시 | 사용 | M1 |
| 인스턴스 간 실시간 전달 (Pub/Sub) | 사용 | M4 (인스턴스를 늘릴 때) |
| 로그인 세션 공유 | 사용 | M4 (인스턴스를 늘릴 때). 그전에도 Redis 세션으로 시작 가능 |
| 리더 선출 | 미정 | 배포 환경 확정 후 결정 |
| 호출 제한 관리 | **사용 안 함** | 수집은 리더 1개만 하므로 메모리 카운터와 API 응답 헤더로 충분하다 |
| 알림 중복 방지 | **사용 안 함** | 알림 상태의 원본이 DB이므로 DB의 조건부 갱신으로 처리한다 |

## 미결정 사항

| 항목 | 결정 시점 |
|---|---|
| 홈서버 실행 형태 (k8s-lab, Docker Compose 등) | 사용자 검토 중. 공개 경로는 Cloudflare Tunnel로 정했다 (D28) |
| 도메인 이름 | Cloudflare에서 구매 예정. 문서에는 `{도메인}`으로 적는다 (D28) |
| 모니터링·로그 도구 조합 (Prometheus, Grafana, Loki 등) | 배포 환경 확정 후 |
| 리더 선출 방식 | 배포 환경 확정 후 |
