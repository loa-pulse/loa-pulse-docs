# 05. API 설계

> 백엔드가 프론트엔드에 제공하는 HTTP API와 SSE의 공통 규칙, 엔드포인트 지도 · 작성일 2026-11-01

화면은 [Figma 디자인](https://www.figma.com/design/jvFsxaWsaW1VrpXBOMlHlw)과 [08-frontend-plan.md](../planning/08-frontend-plan.md)를 기준으로 맞췄다.

## 이 문서와 OpenAPI의 관계

| 대상 | 기준 문서 | 시점 |
|---|---|---|
| 공통 규칙, 엔드포인트 목록, SSE 이벤트 형식 | **이 문서** | 계속 |
| 요청·응답의 필드 단위 명세 | **OpenAPI** (springdoc, Swagger UI) | 계약을 코드로 정의한 뒤부터 |

- 백엔드는 기능마다 **응답 DTO와 컨트롤러 시그니처를 먼저 만들고** OpenAPI를 생성한다. 프론트엔드는 그 명세로 타입과 MSW 목업을 만든다. 구현은 그다음에 채운다.
- 이 문서의 필드 표와 응답 예시는 **OpenAPI가 나오기 전까지의 합의안**이다. 해당 엔드포인트의 OpenAPI가 나오면 필드 표를 지우고 OpenAPI를 가리킨다. 두 곳에 같은 명세를 두지 않는다.
- SSE 이벤트는 OpenAPI로 표현하기 어려우므로 이 문서가 계속 기준이다.

### OpenAPI 전달

| 항목 | 내용 |
|---|---|
| 명세 | `https://api.{도메인}/v3/api-docs` |
| Swagger UI | `https://api.{도메인}/swagger-ui.html`. 포트폴리오 방문자가 직접 호출해 볼 수 있도록 운영에서도 공개한다 |
| 프론트엔드 CI가 가져오는 곳 | 백엔드 레포에 커밋한 `openapi.json`. 백엔드 CI는 생성한 명세가 커밋된 파일과 다르면 실패한다. 프론트엔드 CI는 이 파일로 타입을 다시 생성해 차이가 생기면 실패한다 (프론트엔드 1-13, 1-14) |

## 공통 규칙

### 주소와 출처

- 모든 API는 `https://api.{도메인}/api`로 시작한다. 프론트엔드는 `VITE_API_BASE_URL`로 기본 주소를 받는다.
- CORS는 `https://{도메인}` 하나만 허용하고 자격 증명(쿠키) 전송을 허용한다 ([D28](decisions.md#d28-프론트엔드는-vercel-백엔드는-홈서버에-배포하고-cloudflare-도메인의-서브도메인으로-나눈다)).
- 로컬 개발에서는 Vite 개발 서버가 `/api`를 `http://localhost:8080`으로 넘긴다.

### 인증

- **조회 API는 모두 로그인 없이 쓴다** ([03-screens-flows.md](../planning/03-screens-flows.md)). 로그인하면 응답에 사용자별 값(예: 관심 등록 여부)이 더해진다.
- 로그인은 세션 쿠키로 한다. 쿠키 도메인 `.{도메인}`, `SameSite=Lax`, `Secure`, `HttpOnly` (D28).
- 로그인이 필요한 API를 로그인 없이 부르면 `401`을 돌려준다. 프론트엔드는 `401`이면 로그인 안내를 띄운다.

### CSRF

- `POST`, `PATCH`, `PUT`, `DELETE` 요청은 CSRF 토큰이 필요하다.
- 백엔드는 토큰을 `XSRF-TOKEN` 쿠키(도메인 `.{도메인}`, JavaScript에서 읽을 수 있음)로 내려주고, 프론트엔드는 같은 값을 `X-XSRF-TOKEN` 헤더로 보낸다. Spring Security의 `CookieCsrfTokenRepository` 기본 방식이다.
- 2026-11-01에 이 방식으로 확정했다 ([08-frontend-plan.md](../planning/08-frontend-plan.md#기술)).

### 형식

| 항목 | 규칙 | 예 |
|---|---|---|
| 필드 이름 | camelCase | `currentPrice` |
| 시각 | ISO-8601, UTC (`Z`). 화면 표시(KST, "N초 전")는 프론트엔드가 한다 | `"2026-11-01T05:32:05Z"` |
| 날짜 | `yyyy-MM-dd`, KST 기준 날짜 | `"2026-10-31"` |
| 가격 | 골드. 최저가에서 나온 값은 정수, 평균가는 소수 첫째 자리까지. **모두 `bundleCount`개 묶음 기준**이다 ([02-data-model.md](02-data-model.md)) | `152`, `149.5` |
| 변동률 | 퍼센트 값, 소수 둘째 자리까지. 상승은 양수, 하락은 음수 | `2.01`, `-5.9` |
| 아이템 ID | 로스트아크 API의 `Id` (정수) | `66102007` |
| 빈 값 | 값이 없으면 `null`. 필드를 빼지 않는다 | `"changeRate1h": null` |

### 목록과 페이지

목록은 1부터 시작하는 페이지 번호를 쓴다. 화면의 페이지 번호와 같다.

```
GET /api/items?page=1&size=10
```

```json
{
  "content": [ ... ],
  "page": 1,
  "size": 10,
  "totalElements": 86,
  "totalPages": 9
}
```

### 오류

[RFC 9457](https://www.rfc-editor.org/rfc/rfc9457) `application/problem+json` (Spring `ProblemDetail`)에 프론트엔드가 분기할 `code`를 더한다.

```json
{
  "type": "about:blank",
  "title": "Conflict",
  "status": 409,
  "detail": "활성 알림은 20개까지 등록할 수 있습니다.",
  "code": "ALERT_LIMIT_EXCEEDED"
}
```

| 상태 | 언제 | 프론트엔드 처리 (08-frontend-plan) |
|---|---|---|
| `400` | 요청 값 검증 실패. `errors`에 필드별 사유를 담는다 | 폼 오류 표시 |
| `401` | 로그인이 필요한 API를 로그인 없이 호출 | 로그인 안내 (3-4) |
| `403` | CSRF 토큰이 없거나 다른 사용자의 자원 | 오류 상태 |
| `404` | 없는 아이템, 없는 알림 | 상세 화면 오류 상태 |
| `409` | 업무 규칙 위반 (`code`로 구분) | `detail`을 토스트로 |
| `5xx` | 서버 오류 | 오류 상태 (1-5) |

## 엔드포인트 지도

마일스톤은 [05-milestones.md](../planning/05-milestones.md), 프론트엔드 단위는 [08-frontend-plan.md](../planning/08-frontend-plan.md)의 번호다.

### M1. 시세 조회 (F1~F3)

| 메서드 | 경로 | 용도 | 로그인 | 화면 | 프론트엔드 |
|---|---|---|---|---|---|
| `GET` | `/api/categories` | 카테고리 탭과 탭별 아이템 수 | - | 시세 목록 | 1-7 |
| `GET` | `/api/items` | 아이템 목록, 이름 검색, 정렬 (F1) | - | 시세 목록, 헤더 검색 | 1-6, 1-7 |
| `GET` | `/api/items/{itemId}` | 아이템 상세: 현재가, 전일 대비, 마지막 갱신 시각, 오늘 최저·최고 (F2) | - | 아이템 상세 | 1-9 |
| `GET` | `/api/items/{itemId}/prices` | 오늘 분 단위 추이 (F3) | - | 아이템 상세 차트 [오늘] | 1-10, 1-11 |
| `GET` | `/api/items/{itemId}/daily-prices` | 최근 일별 시세: 평균가, 거래량, 시가·고가·저가·종가 (F3) | - | 아이템 상세 차트 [2주], 최근 일별 시세 표 | 1-11 |
| `GET` | `/api/collection/summary` | 수집 신선도 띠: 평균 반영 지연, 수집 중인 아이템 수, API 호출 여유 | - | 메인, 시세 목록 머리말 | 1-12 |

### M2. 실시간 (F4, F5)

| 메서드 | 경로 | 용도 | 로그인 | 화면 | 프론트엔드 |
|---|---|---|---|---|---|
| `GET` | `/api/rankings` | 급등락 상승·하락 TOP 10 첫 화면 (F5) | - | 메인 | 2-6 |
| `GET` | `/api/stream` | SSE 연결 ([SSE](#sse)) | - (로그인 시 본인 알림 포함) | 전체 | 2-1 ~ 2-7, 3-14 |

### M3. 계정과 알림 (F6, F8~F10)

| 메서드 | 경로 | 용도 | 로그인 | 프론트엔드 |
|---|---|---|---|---|
| `GET` | `/api/auth/login?returnTo={경로}` | 디스코드 로그인 시작. 로그인이 끝나면 `https://{도메인}{returnTo}`로 돌려보낸다 | - | 3-2 |
| `GET` | `/login/oauth2/code/discord` | 디스코드 OAuth 콜백 (Spring Security). 프론트엔드가 직접 부르지 않는다 | - | - |
| `POST` | `/api/auth/logout` | 로그아웃 | ✓ | 3-1 |
| `GET` | `/api/me` | 내 정보. 로그인하지 않았으면 `401` | ✓ | 3-1, 3-15 |
| `PATCH` | `/api/me/settings` | 알림 받기 설정 (급변동 알림 구독) | ✓ | 3-16 |
| `POST` | `/api/me/dm-test` | 테스트 DM 보내기. 결과로 DM 수신 상태를 돌려준다 | ✓ | 3-15 |
| `DELETE` | `/api/me` | 회원 탈퇴. 계정, 알림, 관심 아이템, 알림 내역을 바로 삭제 (D19) | ✓ | 3-16 |
| `GET` | `/api/alerts?status={상태}&itemId={아이템}` | 등록한 알림 목록과 상태별 개수. `itemId`로 상세 화면의 "이 아이템 알림"을 조회 | ✓ | 3-10, 3-11 |
| `POST` | `/api/alerts` | 목표가 알림 등록 (F8) | ✓ | 3-9 |
| `PATCH` | `/api/alerts/{alertId}` | 조건, 기준가, 받을 곳 수정 | ✓ | 3-12 |
| `POST` | `/api/alerts/{alertId}/pause` | 일시정지 (`ACTIVE` → `PAUSED`) | ✓ | 3-12 |
| `POST` | `/api/alerts/{alertId}/resume` | 다시 켜기 (`PAUSED`, `TRIGGERED` → `ACTIVE`) | ✓ | 3-12 |
| `DELETE` | `/api/alerts/{alertId}` | 삭제 | ✓ | 3-12 |
| `GET` | `/api/notifications` | 받은 알림 내역, 최근 30일 (F10) | ✓ | 3-13 |

### M4. 관심 아이템 (F7)

| 메서드 | 경로 | 용도 | 로그인 | 프론트엔드 |
|---|---|---|---|---|
| `GET` | `/api/me/watch-items` | 내 관심 아이템과 현재가, 전일 대비, 오늘 추이(스파크라인) | ✓ | 4-1, 4-3 |
| `PUT` | `/api/me/watch-items/{itemId}` | 관심 등록. 이미 등록돼 있어도 성공 | ✓ | 4-1, 4-2 |
| `DELETE` | `/api/me/watch-items/{itemId}` | 관심 해제. 등록돼 있지 않아도 성공 | ✓ | 4-1, 4-2 |

- 로그인 상태면 `/api/items`, `/api/items/{itemId}`의 응답에 `watched`(관심 등록 여부)를 채운다. 로그인하지 않았거나 M4 전에는 `false`다.

### M5. 서비스 상태 (F12)

| 메서드 | 경로 | 용도 | 로그인 | 프론트엔드 |
|---|---|---|---|---|
| `GET` | `/api/status` | 서비스 상태, 등급별 수집 현황, 최근 30일 장애 목록 | - | 5-1, 5-2 |

- 현재 값(서비스 상태, 등급별 현황, 장애 목록)은 이 API로, 반영 지연 추이 그래프는 Grafana 공개 대시보드 임베드로 보여준다 (D30, 프론트엔드 5-3).
- `/api/collection/summary`(M1)와 `/api/status`는 collector가 직접 계산한 값을 준다. Prometheus나 Grafana를 조회하지 않는다 (D30).

## M1 엔드포인트 상세

OpenAPI가 나오기 전까지 프론트엔드 목업(1-3)의 기준이다. 아이템 정보와 현재가, 전일 평균가는 2026-11-01에 받은 실제 응답([07-api-findings.md](../planning/07-api-findings.md))이고, 직접 수집해야 나오는 값(1시간 변동률, 오늘 최저·최고, 거래량, 시가·고가·저가·종가)은 형식을 보여주기 위한 예시다.

### `GET /api/categories`

화면의 탭은 로스트아크 카테고리를 서비스에서 묶은 것이다. `code`는 서비스가 정한 값이다.

| 탭 `code` | 이름 | 로스트아크 카테고리 코드 |
|---|---|---|
| `all` | 전체 | 아래 전부 |
| `refining` | 재련 재료 | 50010 |
| `refining-extra` | 재련 추가 재료 | 50020 |
| `engraving` | 각인서 | 40000 |
| `battle` | 배틀 아이템 | 60000 (60200, 60300, 60400, 60500) |
| `life` | 생활 재료 | 90000 |
| `cooking` | 요리 | 70000 |
| `etc` | 기타 | 51000, 51100, 230000, 100000, 110000, 140000, 160000, 170000, 220000 |

```json
[
  { "code": "all", "name": "전체", "itemCount": 653 },
  { "code": "refining", "name": "재련 재료", "itemCount": 33 }
]
```

### `GET /api/items`

| 쿼리 | 기본값 | 설명 |
|---|---|---|
| `category` | `all` | 탭 `code` |
| `q` | 없음 | 이름 부분 검색. 헤더 검색은 `category=all&q=` |
| `sort` | `default` | `default`(카테고리 안의 등급순), `change-desc`(전일 대비 높은 순), `change-asc`, `change1h-desc`, `change1h-asc`, `price-desc`, `price-asc` |
| `page`, `size` | `1`, `10` | `size`는 최대 50 |

```json
{
  "content": [
    {
      "id": 66102007,
      "name": "운명의 파괴석 결정",
      "grade": "일반",
      "categoryName": "재련 재료",
      "iconUrl": "https://cdn-lostark.game.onstove.com/efui_iconatlas/use/use_13_249.png",
      "bundleCount": 100,
      "currentPrice": 790,
      "changeRate": -0.2,
      "changeRate1h": -1.25,
      "refreshedAt": "2026-11-01T05:32:05Z",
      "watched": false
    }
  ],
  "page": 1,
  "size": 10,
  "totalElements": 33,
  "totalPages": 4
}
```

| 필드 | 설명 |
|---|---|
| `currentPrice` | 현재 최저가 |
| `changeRate` | 전일 대비. 현재 최저가와 전일 평균가(API의 `YDayAvgPrice`)의 차이. 전일 평균가가 없으면 `null` |
| `changeRate1h` | 최근 1시간. 1시간 전 최저가와의 차이. 수집한 지 1시간이 안 됐으면 `null` |
| `refreshedAt` | 마지막 갱신 시각. 가격이 같아도 수집할 때마다 바뀐다 (D11). 화면은 1분이 넘으면 주황색으로 표시한다 |
| `watched` | 관심 등록 여부 (M4) |

### `GET /api/items/{itemId}`

```json
{
  "id": 66102007,
  "name": "운명의 파괴석 결정",
  "grade": "일반",
  "categoryCode": "refining",
  "categoryName": "재련 재료",
  "iconUrl": "https://cdn-lostark.game.onstove.com/efui_iconatlas/use/use_13_249.png",
  "bundleCount": 100,
  "currentPrice": 790,
  "yesterdayAvgPrice": 791.6,
  "change": -1.6,
  "changeRate": -0.2,
  "todayLow": 785,
  "todayHigh": 801,
  "yesterdayTradeCount": 1240000,
  "refreshedAt": "2026-11-01T05:32:05Z",
  "priceChangedAt": "2026-11-01T05:28:40Z",
  "watched": false
}
```

| 필드 | 설명 |
|---|---|
| `change` | 전일 대비 골드 (현재 최저가 - 전일 평균가) |
| `todayLow`, `todayHigh` | 오늘(KST) 수집한 최저가 중 가장 낮은 값과 높은 값 |
| `yesterdayTradeCount` | 어제 거래량. API 일별 통계. 아직 받지 못했으면 `null` |
| `refreshedAt` | 마지막 갱신 시각 (수집할 때마다) |
| `priceChangedAt` | 가격이 마지막으로 바뀐 시각 |

### `GET /api/items/{itemId}/prices`

오늘(KST 0시부터) 수집한 최저가. 같은 분에 여러 번 수집했으면 그 분의 마지막 값을 쓴다.

```json
{
  "itemId": 66102007,
  "yesterdayAvgPrice": 791.6,
  "points": [
    { "time": "2026-10-31T15:00:00Z", "price": 796 },
    { "time": "2026-10-31T15:01:00Z", "price": 794 }
  ]
}
```

- `yesterdayAvgPrice`는 차트의 "전일 평균가" 기준선이다.

### `GET /api/items/{itemId}/daily-prices`

| 쿼리 | 기본값 | 설명 |
|---|---|---|
| `days` | `14` | 최대 14 (M1). 장기 차트는 로드맵 |

```json
{
  "itemId": 66102007,
  "days": [
    {
      "date": "2026-10-31",
      "avgPrice": 791.6,
      "tradeCount": 1310000,
      "open": 798,
      "high": 805,
      "low": 784,
      "close": 792
    }
  ]
}
```

- 날짜 오름차순이다. 오늘은 집계 중이라 넣지 않는다.
- `avgPrice`, `tradeCount`는 API 일별 통계, `open`~`close`는 직접 수집한 값의 롤업이다 ([02-data-model.md](02-data-model.md)). 롤업이 없는 날(서비스 시작 전)은 `open`~`close`가 `null`이다.
- 알림 설정 모달의 "최근 2주 최저"는 이 응답의 `low`(없으면 `avgPrice`) 중 가장 낮은 값으로 프론트엔드가 계산한다.

### `GET /api/collection/summary`

```json
{
  "serviceStatus": "NORMAL",
  "avgReflectionDelaySeconds": 18,
  "collectedItemCount": 653,
  "apiCallsRemaining": 37,
  "apiCallsLimit": 100,
  "measuredAt": "2026-11-01T05:32:00Z"
}
```

| 필드 | 설명 |
|---|---|
| `serviceStatus` | `NORMAL`(정상), `DELAYED`(수집 지연), `MAINTENANCE`(게임 점검). SSE `freshness`와 같은 값 |
| `avgReflectionDelaySeconds` | 최근 1시간 평균 반영 지연 (D27). 측정값이 없으면 `null` |
| `apiCallsRemaining` | 이번 분에 남은 호출 수 |

## M2 이후 응답의 필드 (요약)

해당 마일스톤을 시작할 때 위와 같은 형식으로 상세를 채운다.

| 엔드포인트 | 필드 (화면 기준) |
|---|---|
| `GET /api/rankings` | `rising[]`, `falling[]`: 순위, 아이템(ID, 이름, 카테고리 이름, 아이콘), 현재가, 1시간 변동률. `calculatedAt` |
| `GET /api/me` | 디스코드 ID, 닉네임, 아바타 URL, 가입일, 서버 참여 여부, DM 수신 상태(`UNKNOWN`, `OK`, `LEFT_GUILD`, `DM_BLOCKED`), 마지막 테스트 DM 시각, 급변동 구독 여부, 활성 알림 수와 한도(20) |
| `GET /api/alerts` | `items[]`: 알림 ID, 아이템(ID, 이름, 아이콘, 현재가), 조건(`BELOW` / `ABOVE`), 기준가, 받을 곳(`dm`, `web`), 상태(`ACTIVE`, `PAUSED`, `TRIGGERED`), 발송 시각. `counts`: 전체, 상태별 개수. `activeLimit`: 20 |
| `POST /api/alerts` | 요청: 아이템 ID, 조건, 기준가, 받을 곳. 활성 알림이 20개면 `409 ALERT_LIMIT_EXCEEDED`, 받을 곳이 하나도 없으면 `400` |
| `GET /api/notifications` | 알림 ID, 아이템(ID, 이름, 아이콘), 발송 당시 가격, 조건과 기준가, 채널별 결과(DM 성공 / DM 실패 / 웹), 시각. 최근 30일 |
| `GET /api/me/watch-items` | 아이템(ID, 이름, 카테고리 이름, 아이콘), 현재가, 전일 대비, 오늘 추이(시간별 최저가 점) |
| `GET /api/status` | 서비스 상태, 갱신 시각, 등급별(`HOT`, `NORMAL`, `COLD`) 아이템 수·수집 주기·평균 반영 지연, 최근 30일 장애(제목, 시작 시각, 지속 시간, 설명, 해결 여부) |

## SSE

[01-data-flow.md](01-data-flow.md#브라우저-실시간-전달-sse), [realtime.md](modules/realtime.md), D4, D15

### 연결

```
GET /api/stream?items=66102007,65203005&ranking=true
Accept: text/event-stream
```

| 쿼리 | 설명 |
|---|---|
| `items` | 구독할 아이템 ID, 쉼표로 구분. 최대 50개. 목록 화면은 보이는 페이지의 아이템, 상세 화면은 그 아이템 하나 |
| `ranking` | `true`면 급등락 랭킹 구독 (메인) |

- 브라우저 하나당 연결 하나다. 페이지를 옮기면 다시 연결한다 (D4).
- 로그인 상태면 세션 쿠키로 사용자를 식별해 본인 `notification` 이벤트도 같은 연결로 보낸다. 프론트엔드는 `withCredentials: true`로 연결한다.
- 연결하거나 재연결하면 구독한 대상의 최신 `price`, `ranking`, `freshness`를 한 번씩 먼저 보낸다. 놓친 이벤트는 다시 보내지 않는다 ([기획 4.6](../planning/04-policies.md#46-실시간-연결)).
- `freshness`가 10초마다 오므로 연결 유지 신호(heartbeat)를 겸한다.
- 탭이 숨겨진 채 10분이 지나면 프론트엔드가 연결을 끊고, 다시 보일 때 재연결한다 (기획 4.6, 프론트엔드 2-5).

### 이벤트

`event:` 이름과 `data:` JSON으로 보낸다.

**`price`**: 구독한 아이템의 가격이 바뀌었을 때 (D11)

```json
{
  "itemId": 66102007,
  "currentPrice": 788,
  "previousPrice": 790,
  "changeRate": -0.45,
  "changeRate1h": -1.5,
  "collectedAt": "2026-11-01T05:35:25Z"
}
```

**`ranking`**: 랭킹을 구독했고 결과가 바뀌었을 때 (30초마다 계산, D12). `GET /api/rankings`와 같은 형식이다.

**`freshness`**: 10초마다 (D15)

```json
{
  "serviceStatus": "NORMAL",
  "items": [
    { "itemId": 66102007, "refreshedAt": "2026-11-01T05:35:25Z" }
  ],
  "sentAt": "2026-11-01T05:35:30Z"
}
```

- `serviceStatus`로 상단 배너를 띄운다 ([기획 4.1](../planning/04-policies.md#41-데이터-신선도)). `DELAYED`는 "시세 수집이 지연되고 있어요", `MAINTENANCE`는 "게임 점검 중".

**`notification`**: 로그인 사용자에게 웹 알림이 생겼을 때. 웹 토스트로 보여준다 (프론트엔드 3-14).

```json
{
  "notificationId": 1024,
  "alertId": 31,
  "item": { "id": 66102007, "name": "운명의 파괴석 결정", "iconUrl": "https://cdn-lostark.game.onstove.com/efui_iconatlas/use/use_13_249.png" },
  "price": 788,
  "direction": "BELOW",
  "threshold": 790,
  "createdAt": "2026-11-01T05:35:26Z"
}
```

## 백엔드 설계에 주는 영향

| 사실 | 영향 | 반영할 곳 |
|---|---|---|
| 카테고리 탭이 하위 카테고리(재련 재료 50010, 재련 추가 재료 50020)로 나뉜다. 검색 응답에는 카테고리 코드가 없다 | 아이템의 카테고리를 알려면 **하위 카테고리 코드로 검색해야** 한다. 검색은 가장 아래 단계 코드로 한다 (한 바퀴 78회) | **반영함**: D29, [collector.md](modules/collector.md) 수집 단위 |
| 목록에 "최근 1시간" 변동률이 있다 | 목록 조회마다 1시간 전 가격이 필요하다. `latest_price`에 1시간 변동률을 함께 두거나 랭킹 계산(30초)과 함께 갱신한다 | [market.md](modules/market.md), [02-data-model.md](02-data-model.md) |
| 상세에 오늘 최저·최고, 어제 거래량이 있다 | 오늘 원본(`price_snapshot`)의 최솟값·최댓값과 `daily_price.trade_count`를 조회한다 | market.md |
| 메인과 목록 머리말에 평균 반영 지연, 수집 중인 아이템 수, 호출 여유가 있다 (M1, 프론트엔드 1-12) | collector가 현재 값을 HTTP API로 제공해야 한다 | **반영함**: D30, [collector.md](modules/collector.md) |

## 정해야 할 것

| 항목 | 내용 | 정할 시점 |
|---|---|---|
| 설정의 "웹 알림" 전체 켜기·끄기 | Figma 설정 화면에 있지만 데이터 모델에는 알림별 `notify_web`만 있다. 사용자 설정으로 둘지, 화면에서 뺄지 | M3 전 |
| 급변동 알림 "하루 최대 5번" | Figma 설정 화면의 문구다. [기획 4.3](../planning/04-policies.md#43-급변동-알림-f11-p1)에는 아이템별 1시간 쿨다운만 있다 | M5 전 |
| 받은 알림 내역(F10)의 마일스톤 | 프론트엔드 계획은 M3(3-13), 마일스톤 문서는 M5다 | M3 전 |
| 기타 탭의 범위 | 모험의 서, 펫, 탈것 등을 기타 하나로 묶을지, 수집 범위에서 뺄지 | M1 수집 대상 관리 전 |