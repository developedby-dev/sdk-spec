# 적합성 하네스 프로토콜 v1

언어마다 작은 명령 줄 프로그램(**하네스**)이 SDK를 몰고, 실행기(`sdks/conformance`)가 시나리오를 읽어 하네스와 이야기한다. 실행기가 곧 가짜 서버다 —
하네스는 HTTP를 실제로 보내지 않고 요청을 실행기에 넘기며, 실행기가 시나리오대로 답하면서 요청은 contracts 요청 스키마로, 답은 응답 스키마로 검사한다.
그래서 시험은 결정적이다: 시계 · 난수 · 응답 순서를 시나리오가 정한다.

> 기본 HTTP 전송(실제 소켓)은 이 프로토콜로 시험하지 않는다. 각 SDK의 단위 시험이 가짜 서버의 HTTP 모드(`pnpm --filter @bydev/sdk-conformance serve`)에
> 대고 따로 확인한다.

## 1. 실행

```
pnpm --filter @bydev/sdk-conformance run conformance -- --harness "<하네스 실행 명령>" [--scenario <이름>]
```

실행기는 시나리오마다 하네스를 새로 띄운다(상태를 나누지 않게). 하네스는 표준 입력에서 한 줄에 JSON 하나씩 읽고, 표준 출력에 한 줄에 JSON 하나씩 쓴다.
표준 오류는 실행기가 실패 보고에 그대로 붙인다(디버그 출력은 거기로).

## 2. 하네스가 만드는 포트

| 포트 | 하네스 구현 |
|---|---|
| `Transport` | `http.request`를 쓰고 같은 `id`의 `http.response`를 기다린다 |
| `Clock` | 가상 시계. `init.now`에서 시작해 `advance`로만 간다 |
| `Scheduler` | 가상 스케줄러. 예약 = (시각, 작업). `advance`가 시각 순서대로 작업을 돌리고 **각 작업이 끝날 때까지 기다린다**. 같은 시각이면 예약한 순서 |
| `Random.float` | `init.random`의 값을 앞에서부터 하나씩 꺼낸다. 다 쓰면 `fatal` |
| `Random.uuid` | `00000000-0000-4000-8000-000000000001`부터 마지막 자리를 하나씩 올린 값(16진 12자리) |
| `CredentialStore` | 메모리. 시작값은 `init.storage`. `save` · `clear`마다 `storage` 사건을 쓴다. `set storage.fail*`로 실패를 낸다 |
| `FingerprintSource` | `init.config.device.fingerprintSource`(null이면 읽지 못함) |

시간은 정수 밀리초다. 지연을 만드는 곱셈(지터)은 명세대로 `round`한다.

## 3. 메시지

### 실행기 → 하네스

| type | 필드 | 하네스가 할 일 |
|---|---|---|
| `init` | `config`(`baseUrl` · `clientId` · `productId` · `scopes` · `device` { `name` · `platform` · `fingerprintSource` } · `app` { `target` · `version` }(없을 수 있다)), `now`(ISO), `random`(수 배열), `storage`(기록 또는 null) | 포트와 SDK 클라이언트를 만든다. 아직 요청을 보내지 않는다 |
| `call` | `calls`: [{ `id`, `method`, `args` }] | 나열한 호출을 **같은 순간에** 시작한다(앞 호출을 기다리지 않는다). 결과는 끝나는 대로 `result` |
| `advance` | `ms` | 가상 시각을 `ms`만큼 옮기며 그 사이에 온 예약 작업을 돌린다 |
| `set` | `key`, `value` | `storage.failLoad` · `storage.failSave` · `storage.failClear`(참/거짓) |
| `http.response` | `id`, `response`({ `status`, `headers`, `body`(문자열) }) 또는 `error`(`"NETWORK"`) | 그 요청을 기다리던 전송에 돌려준다 |
| `end` | — | 열린 작업을 버리고 끝난다(종료 코드 0) |

### 하네스 → 실행기

| type | 필드 | 언제 |
|---|---|---|
| `ready` | `sdk`(언어), `version`, `specVersion` | 띄운 직후 한 번 |
| `http.request` | `id`(하네스가 정함, 겹치지 않게), `now`, `request`({ `method`, `url`, `headers`(이름은 소문자), `body`(문자열 또는 null) }) | 전송이 보낼 때 |
| `event` | `now`, `event` | 아래 사건 |
| `result` | `id`, `now`, `ok`, `value` 또는 `error` | 호출이 끝나면 |
| `idle` | `now` | `init` · `call` · `advance` · `set` 하나를 처리하고 **더 할 일이 없을 때**(돌 수 있는 작업 없음 · 답을 기다리는 요청 없음). 남은 것은 미래 예약과 미래를 기다리는 호출뿐이다 |
| `fatal` | `message` | 하네스가 더 갈 수 없을 때(난수 소진 등). 시험 실패 |

`now`는 그 메시지를 쓸 때의 가상 시각(Unix 밀리초 정수)이다. 실행기는 이 값으로 시각을 확인하고(`atSec`) 응답의 상대 시각(`$at`)을 푼다. `idle`의 `now`는
명령이 끝난 시각이라 `advance` 뒤에는 옮긴 만큼이어야 한다.

`http.response`에는 `idle`을 쓰지 않는다 — 그 응답을 낳은 명령의 `idle` 하나로 끝난다.

**"더 할 일이 없음"은 SDK가 기다리는 비동기 작업이 하나도 없다는 뜻이다** — 답을 기다리는 요청뿐 아니라 런타임이 다른 스레드에서 끝내는 일(암호 연산 ·
파일 읽기 같은)까지. 런타임의 작업 큐가 비었는지만 보면 그런 일이 끝나기 전에 `idle`을 써서 결과가 다음 명령 뒤로 밀린다. TS 하네스는 WebCrypto 호출
수를 센다 — 다른 언어도 그 런타임에서 같은 일을 하는 지점을 센다. 블로킹 언어(Go 등)의 하네스는 "돌고 있는 작업 수"를 센다 — 호출 · 예약 작업이 시작하면 하나
늘리고, 전송 · `sleep` · `wait`에서 기다리는 동안은 하나 줄인다(명세 2절 — SDK의 기다림은 모두 이 포트를 거친다).

### 사건

| kind | 필드 |
|---|---|
| `session.state` | `state` — 명세 8.1의 상태. `type`은 `Starting` 등 명세 이름 그대로 |
| `storage` | `op`(`save` · `clear`), `record`(save일 때) |

### 호출 (`method`)

| method | args | value |
|---|---|---|
| `login.start` | — | `{ userCode, verificationUri, verificationUriComplete, expiresAt }` |
| `login.wait` | — | `{ scope }` |
| `login.cancel` | — | `null` |
| `hasCredentials` | — | 참/거짓 (저장된 기록이 있는지 — 이용 가능 여부가 아니다) |
| `entitlement` | — | 계약 `Entitlement` |
| `devices` | `{ cursor?, limit? }` | `{ items, nextCursor }` |
| `releaseDevice` | `{ deviceId }` | `{ id, status, releasedAt }` |
| `profile` | — | 계약 `Profile` |
| `checkForUpdate` | `{ target, currentVersion }` | 계약 `ProductUpdatesResponse` |
| `verifyDownload` | `{ dataBase64, sha256 }` | `null` |
| `verifyToken` | `{ token }` | licenseToken 클레임 |
| `logout` | — | `{ serverRevoked }` |
| `session.start` · `session.takeOver` · `session.retry` · `session.stop` | — | `null` |
| `session.state` | — | 지금 상태 |

값을 쓸 때 열거는 원래 문자열 그대로, 시각은 ISO 8601 UTC, 없는 값은 `null`이다(선택 필드 — 클레임 `build_id` · 업데이트 확인 `entitled` — 는 `null` 또는 생략). 에러는 `{ code, message, httpStatus, correlationId, retryAfterSec,
details }`(없는 필드는 null).

## 4. 시나리오

`sdks/conformance/scenarios/*.json`. 스키마는 `sdks/conformance/src/scenario.ts`(zod)가 원천이다. 단계는 차례로 돈다:

- **명령 단계**(`call` · `calls` · `advance` · `set`)를 보내고, 하네스가 쓰는 메시지를 뒤따르는 **기대 단계**(`request` · `event` · `result` · `anyOrder`)와
  하나씩 맞춘다. 그 명령의 `idle`이 오면 다음 단계는 명령이거나 끝이어야 한다 — 기대가 남았는데 `idle`이면 "일어나지 않음"으로 실패, 기대하지 않은 메시지가
  오면 "예상 밖"으로 실패다.
- 발견 문서 · JWKS 요청은 시나리오가 `auto`에서 끄지 않으면 실행기가 기본 응답으로 저절로 답한다(기대 단계를 쓰지 않는다).
- 응답 본문의 `{"$licenseToken": {…}}` · `{"$accessToken": {…}}`는 실행기가 시험 키로 서명한 JWT로 바꾼다. `$sessionResponse` · `$tokenResponse` ·
  `$deviceCodeResponse`는 서로 맞물린 값으로 채운 성공 본문이다. 시각은 `{"$at": "+600s"}`(그 요청의 가상 시각 기준 — 초 단위 ISO, 서버처럼 밀리초를
  버린다) · `{"$unixAt": "+600s"}`(초). `{"error": "CODE"}` 응답은 상태 · 메시지를 contracts에서 채우고, 429면 `Retry-After`를 단다.
- 기대의 `atSec`은 그 메시지의 가상 시각이다(시나리오 시작 기준 초).
- 기대 값의 맞추기 연산자: `$any` · `$absent` · `$capture`(이름에 묶기) · `$same`(묶은 값과 같음) · `$differs` · `$token`(서명한 토큰 이름) ·
  `$bearer` · `$fingerprint` · `$at`(시각 — 순간으로 비교) · `$regex` · `$between`. 객체는 부분 일치(적힌 키만 본다), 배열은 길이까지 같아야 한다.
