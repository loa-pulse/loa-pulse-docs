# 02. 데이터 모델과 시계열 보관 정책

## 시세 데이터 보관 정책

[D16](decisions.md#d16-시세-원본은-3일만-보관하고-일별-값은-영구-보관한다), [D17](decisions.md#d17-시세-파티션은-애플리케이션-예약-작업으로-관리한다)

### 기능별로 필요한 데이터

| 기능 | 필요한 데이터 |
|---|---|
| F3 오늘 차트 | 오늘의 수집 원본 |
| F3 2주 차트 | 최근 14일의 일별 값 |
| F5 랭킹, collector 등급, 급변동 판정 | 1시간 전 가격 |
| collector COLD 판정, 급변동 오탐 방지 | 일별 거래량 |
| 로드맵: 장기 차트 | 몇 달 이상의 일별 값 |

원본은 길어야 하루치만 쓰인다. 그보다 오래된 데이터는 모두 일별 값으로 충분하다.

### 보관 구조

```
price_snapshot (원본)       ──보관 3일──▶ 날짜별 파티션째 삭제
    │
    └─ 매일 새벽 롤업 ──▶ daily_price (일별)  ──영구 보관
                          ▲
로스트아크 API 일별 통계 ──┘ (평균가, 거래량)
```

| 테이블 | 내용 | 보관 기간 | 크기 (추정) |
|---|---|---|---|
| `price_snapshot` | 수집할 때마다 1행 (아이템, 최저가, 수집 시각) | 3일 | 하루 약 50만 행, 3일치 약 150만 행 |
| `daily_price` | 아이템별 하루 1행 | 영구 | 1,000개 × 365일 = 연 36만 행 |

- 원본을 3일 두는 이유: 실제로 쓰는 건 하루치지만, 롤업 작업이 실패했을 때 다시 돌릴 여유를 이틀 더 둔다.

### 일별 값의 출처

| 값 | 출처 | 쓰임 |
|---|---|---|
| 시가, 고가, 저가, 종가 (최저가 기준) | 내 원본을 롤업 | 2주 차트, 장기 차트 |
| 평균 거래가, 거래량 | 로스트아크 API 일별 통계 | COLD 판정, 급변동 오탐 방지 |

- 거래량은 직접 계산할 수 없으므로 API에서 받는다.
- API 일별 통계는 아이템마다 하루 1번이면 된다. collector가 수집 예산이 남을 때 낮은 우선순위로 가져온다 (하루 약 650회, 예산의 1% 미만).
- 상세 조회는 최근 14일의 날짜별 평균가와 거래량을 준다. 같은 아이템이 거래 가능 횟수별로 여러 항목으로 오므로, **검색에서 받은 `TradeRemainCount`와 같은 항목만 쓴다** ([07-api-findings.md](../planning/07-api-findings.md#검색-결과와-상세-항목의-대응)).

### 파티션 관리

`price_snapshot`은 수집 시각 기준 날짜별 범위 파티션으로 나눈다. 애플리케이션 예약 작업이 매일 다음을 한다.

- 앞으로 3일치 파티션을 미리 만든다.
- 3일 지난 파티션을 삭제한다 (`DROP` 이므로 행 단위 `DELETE`보다 훨씬 가볍다).
- 인스턴스가 여러 개일 때는 리더만 실행한다.

## 테이블 구조

[D18](decisions.md#d18-모듈은-자기-테이블만-가지고-모듈을-넘는-외래-키는-두지-않는다)

### 원칙

- 모듈은 자기 테이블만 가진다. **다른 모듈의 테이블은 읽지도 JOIN하지도 않는다.** 필요하면 그 모듈의 포트로 묻는다.
- **모듈을 넘는 외래 키(FK)는 두지 않는다.** 다른 모듈의 데이터는 ID 값으로만 참조한다. 모듈 안에서는 FK를 둔다.

### collector

```
collection_target
  id, kind(PRICE | DAILY_STAT), unit_key(카테고리 코드 + 페이지 번호), tier,
  last_collected_at, next_due_at, consecutive_failures, paused_until
collection_target_item
  target_id ─FK─▶ collection_target, item_id
```

- `kind`로 일별 통계 수집도 같은 테이블과 같은 스케줄러로 관리한다. 일별 통계는 주기가 24시간이고 우선순위가 가장 낮은 대상이다.
- 수집 단위는 카테고리 검색 페이지 하나다 (D26). `collection_target_item`은 그 페이지에 든 아이템 `Id` 목록이고, 수집할 때 받은 목록과 다르면 그 카테고리의 페이지를 다시 만든다.

### market

```
item            id(API 아이템 코드), name, category_code, grade, bundle_count,
                trade_remain_count(NULL 가능), icon_url
price_snapshot  item_id, collected_at, lowest_price
                PK(item_id, collected_at), collected_at 기준 날짜별 파티션
latest_price    item_id(PK), price, prev_price, price_changed_at, refreshed_at
daily_price     item_id, date, open, high, low, close, api_avg_price, trade_count
                PK(item_id, date)
```

- `latest_price.refreshed_at`은 가격이 같아도 수집할 때마다 바뀌고(D11), `price_changed_at`은 가격이 바뀔 때만 바뀐다.
- 아이템은 API의 `Id` 하나로 식별한다. `trade_remain_count`는 상세 조회에서 어느 항목을 쓸지 고르는 데만 쓴다.
- 가격은 모두 `bundle_count`개 묶음 기준이다. 개당 가격으로 바꾸지 않고 그대로 저장한다.
- 최저가에서 나온 값(`lowest_price`, `price`, `prev_price`, 시가·고가·저가·종가, 알림의 `threshold`)은 API가 정수로 주므로 `integer`, `api_avg_price`는 소수 첫째 자리까지 오므로 `numeric(12, 1)`로 한다.

### alert

```
price_alert   id, user_id, item_id, direction(ABOVE | BELOW), threshold,
              notify_dm, notify_web, status, triggered_at
              인덱스: (item_id, status, direction, threshold)    ← 조건 매칭
notification  id, user_id, source_type(PRICE_ALERT | SURGE), source_id, channel,
              content, status, attempts, next_attempt_at, claimed_until, last_error, sent_at
              인덱스: (next_attempt_at) WHERE status = 'PENDING'  ← 발송 워커
                      (user_id, created_at)                      ← 알림 내역
surge_event   id, item_id, change_rate, detected_at              ← 급변동 쿨다운 판정, 알림의 원인
```

- `notification`은 알림 내역(F10)이 최근 30일만 보여주므로 **30일 지난 행을 지우는 예약 작업**을 둔다.

### user

```
app_user     id, discord_id(UNIQUE), nickname, avatar_url, surge_subscribed, dm_status
watch_item   user_id ─FK─▶ app_user, item_id
             PK(user_id, item_id)
```

- 테이블 이름 `user`는 PostgreSQL 예약어라서 `app_user`로 한다.

## Redis 키

Redis 키도 테이블처럼 **소유 모듈만 읽고 쓴다** (D18). 다른 모듈은 소유 모듈의 공개 API로 조회한다.

| 키 | 내용 | 소유 모듈 | 다른 모듈이 조회하는 방법 |
|---|---|---|---|
| `price:latest:{itemId}` | 최신 시세 (가격, 직전 가격, 1시간 변동률, 갱신 시각) | market | `market.api.LatestPriceQuery` |
| `ranking:latest` | 최신 랭킹 | market | `RankingUpdated` 이벤트, market의 조회 API |
| `system:status` | 서비스 상태 (정상 / 수집 지연 / 게임 점검) | collector | `collector.api.CollectionStatusQuery` |
| `spring:session:*` | 로그인 세션 | user (Spring Session) | - |
| Pub/Sub 채널 `realtime:*` | 인스턴스 간 전달 (M4부터) | realtime | - |

Redis는 캐시다. 값이 없으면 DB에서 읽어 다시 채운다. (`system:status`는 DB 원본이 없으므로, 값이 없으면 collector가 다음 수집 실행에서 다시 계산한다.)

## 회원 탈퇴 시 데이터 삭제

[D19](decisions.md#d19-회원-탈퇴-시-다른-모듈의-데이터는-user가-정의한-포트로-한-트랜잭션에서-삭제한다)

알림 데이터(`price_alert`, `notification`)는 alert의 테이블이라 user가 직접 지울 수 없다.

```
user: 탈퇴 처리 (한 트랜잭션)
  ├─ UserDeletionHook.onUserDeleted(userId)  → alert가 구현: 자기 알림 데이터 삭제
  └─ watch_item, app_user 삭제
```

- `UserDeletionHook`은 user가 정의하고 alert가 구현한다.
- 같은 DB를 쓰는 모놀리스라서 탈퇴 처리 전체를 한 트랜잭션으로 묶는다. 하나라도 실패하면 탈퇴 전체가 롤백된다.
