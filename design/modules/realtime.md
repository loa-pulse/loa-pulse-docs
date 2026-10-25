# realtime 모듈

> 기술 지원 · SSE 연결 관리와 인스턴스 간 전달을 맡는다

## 구성 요소

| 구성 요소 | 설명 |
|---|---|
| `SseConnectionRegistry` | 이 인스턴스에 연결된 브라우저와 각자 구독한 대상 (아이템들, 랭킹, 본인 알림) |
| `BroadcastBus` | 인스턴스 간 전달. **M1~M3에서는 메모리로, M4부터 Redis Pub/Sub으로 구현만 교체한다** |

## 전달 흐름

```
PriceUpdated / RankingUpdated 수신 → BroadcastBus로 모든 인스턴스에 전달
  → 각 인스턴스가 자기에게 연결된 브라우저 중 구독자에게만 전송
```

## 주기 작업

10초마다 연결된 브라우저에 `freshness` 이벤트(구독한 아이템별 마지막 갱신 시각, 서비스 상태)를 보낸다. 이벤트 종류 전체는 [01-data-flow.md](../01-data-flow.md)를 따른다 (D15).

- 마지막 갱신 시각은 market의 `LatestPriceQuery`로, 서비스 상태는 collector의 `CollectionStatusQuery`로 조회한다. **Redis를 직접 읽지 않는다** (D18, D24).

## 재연결

놓친 이벤트를 다시 보내지 않고, **구독한 대상의 최신 값을 한 번 보낸다.** 시세는 최신 상태만 의미가 있기 때문이다 ([기획 4.6](../../planning/04-policies.md)).

## 포트와 공개 API

realtime은 의존 방향의 가장 아래에 있다. 다른 모듈은 realtime에 의존하지 않는다 (D24).

### realtime이 쓰는 위쪽 모듈의 공개 API와 이벤트

| 대상 | 정의한 곳 | 용도 |
|---|---|---|
| `PriceUpdated`, `RankingUpdated` 이벤트 | `market.api` | `price`, `ranking` 이벤트 전송 |
| `LatestPriceQuery` | `market.api` | `freshness` 이벤트, 재연결 시 최신 값 전송 |
| `CollectionStatusQuery` | `collector.api` | `freshness` 이벤트의 서비스 상태 |

### realtime이 구현하는 위쪽 모듈의 인터페이스

| 인터페이스 | 정의한 곳 | 제공하는 것 |
|---|---|---|
| `NotificationSender` | `alert.api` | 웹 채널 발송 (접속 중인 사용자에게 토스트) |
