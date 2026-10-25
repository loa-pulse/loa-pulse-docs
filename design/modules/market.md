# market 모듈

> 지원 · DDD 전술 설계는 가볍게, 저장과 조회 성능이 중심이다

## 구성 요소

| 구성 요소 | 설명 |
|---|---|
| `Item` | 아이템 정보 (ID, 이름, 카테고리, 등급, 묶음 단위) |
| `PriceSnapshot` | 수집 시점의 시세 기록. **추가만 하고 수정하지 않는다** (시계열) |
| `LatestPrice` | 아이템별 최신 시세. 조회는 Redis 캐시가 먼저 받는다 |
| `DailyPrice` | 일별 시세 (API가 주는 최근 2주치 + 직접 롤업한 값) |
| `RankingCalculator` | 1시간 전 대비 변동률로 상승·하락 TOP 10을 계산한다 |

## 저장 흐름 (`MarketDataSink` 구현)

```
수집 결과 받음 → PriceSnapshot 추가 → LatestPrice 갱신(DB + Redis)
             → 가격이 바뀐 아이템만 PriceUpdated 이벤트 발행
```

- **가격이 바뀐 아이템만 이벤트를 발행한다** ([D11](../decisions.md#d11-가격이-바뀐-아이템만-priceupdated를-발행한다)). 같은 가격이 다시 수집된 경우까지 alert와 realtime에 보낼 필요가 없다.
- `LatestPrice`의 **갱신 시각은 가격이 같아도 항상 갱신한다.** 화면의 "마지막 갱신: N초 전"(F2)이 정확해야 하기 때문이다.

## 발행하는 이벤트

| 이벤트 | 내용 | 발행 시점 |
|---|---|---|
| `PriceUpdated` | 아이템 ID, 현재가, 직전 가격, 1시간 변동률, 수집 시각 | 저장한 시세의 가격이 직전과 다를 때 |
| `RankingUpdated` | 상승·하락 TOP 10 | 랭킹을 다시 계산했는데 결과가 바뀌었을 때 |

## 랭킹 계산

시세가 갱신될 때마다 다시 계산하지 않고 **30초마다 주기적으로** 계산한다 ([D12](../decisions.md#d12-급등락-랭킹은-30초마다-주기적으로-계산한다)).

## 시계열 보관

원본(`price_snapshot`)은 날짜별 파티션으로 3일만 보관하고, 일별 값(`daily_price`)은 영구 보관한다. 롤업과 파티션 관리 예약 작업은 market이 소유한다. 자세한 내용은 [02-data-model.md](../02-data-model.md)를 따른다 (D16, D17).

## 포트와 공개 API

market은 collector에만 의존한다 (D24).

### `market.api`: 공개 API와 이벤트 (market이 정의)

| 대상 | 쓰는 곳 | 용도 |
|---|---|---|
| `LatestPriceQuery` | realtime | 아이템별 최신 시세와 마지막 갱신 시각. `freshness` 이벤트와 재연결 시 최신 값 전송에 쓴다. 내부적으로 Redis `price:latest`를 읽는다 |
| `PriceUpdated` 이벤트 | alert, realtime | 시세 갱신 |
| `RankingUpdated` 이벤트 | realtime | 랭킹 갱신 |

### market이 구현하는 위쪽 모듈의 인터페이스

| 인터페이스 | 정의한 곳 | 제공하는 것 |
|---|---|---|
| `MarketDataSink` | `collector.api` | 수집 결과 저장 (D8) |
| `PriceVolatilityPort` | `collector.api` | 아이템별 최근 1시간 변동률 |
