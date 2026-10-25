# user 모듈

> 지원 · DDD 전술 설계는 가볍게 적용한다

## 구성 요소

| 구성 요소 | 설명 |
|---|---|
| `User` | 디스코드 ID, 닉네임, 아바타, 급변동 알림 구독 여부, DM 수신 상태 |
| `WatchItem` | 관심 아이템 (사용자 ID, 아이템 ID) |

## DM 수신 상태

| 상태 | 의미 |
|---|---|
| `UNKNOWN` | 아직 DM을 보내본 적 없음 |
| `OK` | 정상 수신 |
| `LEFT_GUILD` | LoaPulse 디스코드 서버를 나가서 DM을 받을 수 없음 |
| `DM_BLOCKED` | 디스코드 설정에서 서버 멤버 DM을 막아 둠 |

- 웹 상단 배너([기획 4.4](../../planning/04-policies.md) 실패 대비책)를 띄우는 근거다.
- alert가 DM 결과를 user의 공개 API(`DmStatusRecorder`)로 알려주면 user가 상태를 바꾼다. DM이 다시 성공하면 `OK`로 돌아온다.

## 로그인 흐름

```
디스코드 OAuth 완료 → User 생성 또는 프로필 갱신
                   → 봇으로 LoaPulse 서버에 추가 (guilds.join 토큰은 이때만 쓰고 저장하지 않음)
                   → 최초 로그인이면 테스트 DM 안내
```

## 회원 탈퇴

한 트랜잭션 안에서 `UserDeletionHook`을 호출해 다른 모듈의 데이터를 지우고, 자기 데이터(`watch_item`, `app_user`)를 지운다 ([02-data-model.md](../02-data-model.md), D19).

## 포트와 공개 API

user는 collector에만 의존한다. **업무 모듈인 alert에는 의존하지 않는다** (D24).

### `user.api`: 공개 API (user가 정의하고 구현)

| API | 쓰는 곳 | 용도 |
|---|---|---|
| `UserQuery` | alert | 알림 받을 사용자 정보(디스코드 ID, DM 수신 상태), 급변동 구독자 목록 |
| `DmStatusRecorder` | alert | DM 발송 결과를 반영해 DM 수신 상태 변경 |

### `user.api`: 아래쪽 모듈이 구현하는 인터페이스 (의존성 역전)

| 인터페이스 | 구현하는 곳 | 용도 |
|---|---|---|
| `UserDeletionHook` | alert | 탈퇴한 사용자의 알림 데이터 삭제 (D19) |

### user가 구현하는 위쪽 모듈의 인터페이스

| 인터페이스 | 정의한 곳 | 제공하는 것 |
|---|---|---|
| `DemandSignalPort` | `collector.api` | 아이템별 관심 등록 수 |
