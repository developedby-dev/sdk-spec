# SDK 동작 명세 v1

> 여덟 언어의 공식 SDK가 따라야 할 클라이언트 동작 명세입니다. 요청·응답 형식과 오류 코드는 `packages/contracts`를 따릅니다.
>
> 클라이언트의 처리 시점과 순서는 이 문서에서 정하고, 수치 기본값은 [`rules.json`](rules.json)에 둡니다. `rules.x.y`는 해당 파일의 경로입니다.
>
> 구현은 [`../conformance`](../conformance)의 시나리오로 검증합니다. 시나리오와 이 문서가 다르면 대조해 수정해야 합니다.
>
> 하네스와 실행기의 통신 규약은 [`harness-protocol.md`](harness-protocol.md)를 참고해 주세요. 근거: ADR-0040.

## 0. 원칙

1. **SDK는 안내서가 판매자에게 하라고 적은 것만 합니다.** 그 밖의 네트워크 호출 · 수집 · 원격 측정을 하지 않습니다.

   SDK는 설정한 API 출처만 호출합니다.
2. **정책 수치는 응답을 따릅니다** (불변식 5). 하트비트 간격(`session.heartbeatSec`), 토큰 수명(`expires_in`), 폴링 간격(`interval`),
   재시도 대기(`retryAfterSec`)는 서버가 준 값을 씁니다. 첫 응답 전에 필요한 값과 서버가 주지 않는 클라이언트 쪽 값만 `rules.json`에 있습니다.
3. **시계, 대기, 난수, HTTP와 저장소는 주입받습니다** (불변식 13). SDK 안에서 시계를 직접 읽는 코드는 주입된 기본 구현 하나뿐입니다.

   **백그라운드 작업은 모두 스케줄러 포트로 시작합니다.** SDK가 별도로 스레드, 타이머나 무한 루프를 만들지 않습니다.

   적합성 시험은 이 포트에 가상 시계를 주입해 동작을 검증합니다.
4. **참/거짓 하나로 이용 가능 여부를 알리지 않습니다** (불변식 9). `isLicensed()` 같은 함수를 두지 않습니다.

   앱은 검증된 토큰의 값(`exp`, `sid`, 이용권
   기간)과 판별 유니온 상태를 받습니다.
5. **새 필드와 열거 값의 추가를 허용합니다** (불변식 8). 모르는 필드는 폐기합니다.

   모르는 열거 값은 원래 문자열을 보존하는 `Unknown("PAUSED")` 같은 값으로 받습니다. 모르는 오류 코드는 HTTP 상태로 분류합니다(3.4절).

6. **분기는 `code`로 합니다.** `message`는 표시용입니다.

   오류에는 응답 헤더 `X-Correlation-Id` 값을 포함합니다. 앱 개발자에게 오류 화면에 이 값을 표시하도록 안내합니다.
7. **원본 지문은 전송하거나 저장하지 않습니다** (불변식 7, 10절).

## 1. 설정

| 이름 | 필수 | 뜻 |
|---|---|---|
| `baseUrl` | 예 | API 출처이며 끝의 `/`를 제거한 값이 발견 문서의 `issuer`와 같아야 합니다(4절). 예: `https://api.developedby.dev`. |
| `clientId` | 예 | 판매자 콘솔이 제품별로 발급한 `client_id` |
| `productId` | 예 | 그 제품의 ID(`prod_…`). 업데이트 확인에 씁니다. |
| `scopes` | 예 | 요청할 스코프 목록. 앱이 실제로 쓰는 것만: 세션을 쓰면 `session.write`, 업데이트를 쓰면 `version.read` |
| `device` | 세션을 쓰면 | `{ name, platform }`. 기본 구현은 운영체제에서 읽습니다(호스트 이름 · `WINDOWS` 등) |
| `app` | 아니요(세션을 쓰면 넣어 주세요) | `{ target, version }`. 실행 중인 앱의 배포 대상(업데이트 확인의 `target`과 같은 값)과 콘솔에 업로드한 버전 문자열. 세션 획득에 싣습니다(8.2). 업데이트 제공 기한이 있는 영구 이용권은 이 값이 없으면 획득이 `ENTITLEMENT_VERSION_NOT_COVERED`로 거절됩니다(ADR-0049). |
| 포트들 | 아니요 | 2절. 주지 않으면 그 런타임의 기본 구현 |

## 2. 포트

> 지터 적용과 초→밀리초 변환으로 구한 지연 시간 `x`는 `floor(x + 0.5)`로 계산해 밀리초 단위 정수로 반올림합니다. 은행가 반올림 등 언어별 기본 방식이 다르므로 이 계산식을 직접 적용해 주세요.

| 포트 | 계약 | 기본 구현 |
|---|---|---|
| `Transport` | `send({ method, url, headers, body }) → { status, headers, body }`. 연결 실패나 시간 초과에는 `NETWORK_ERROR` 오류를 발생시키며 3xx 리다이렉트는 따르지 않습니다. | 해당 언어의 표준 HTTP 클라이언트, 시간 초과 `rules.http.timeoutSec` |
| `CredentialStore` | `load() → 기록 \| 없음`, `save(기록)`, `clear()`. 실패는 `STORAGE_FAILED` | 운영체제 보안 저장소(아래 표) |
| `Clock` | `now() → 시각` (밀리초) | 시스템 시계 |
| `Scheduler` | `schedule(지연 ms, 작업) → 취소`. 스케줄러는 비동기 작업이 완료될 때까지 기다릴 수 있어야 하며, 블로킹 언어의 추가 계약은 아래 안내를 따릅니다. | 해당 런타임의 타이머 |
| `Random` | `float() → [0, 1)`, `uuid() → UUID v4 문자열` | 암호학적 난수 |
| `FingerprintSource` | `read() → 문자열 \| 없음` | 아래 표 |
| `Logger` | `debug`, `info`, `warn`. **토큰과 지문 출처 값은 전달하지 않습니다.** | 로그 기록 없음 |

Go처럼 스레드나 고루틴에서 대기하는 블로킹 언어의 스케줄러는 `sleep(지연)`과 `wait(다른 작업의 끝)`도 제공합니다. SDK의 시간 대기와 single-flight 작업 대기는 이 두 함수를 거쳐야 하네스가 대기 횟수를 측정할 수 있습니다.

공유 상태의 잠금은 짧게 유지하고, 잠금을 보유한 채 포트를 호출하지 않습니다.

| 포트 | Windows | macOS | Linux | 브라우저 확장(MV3) |
|---|---|---|---|---|
| 보안 저장소 | DPAPI(사용자 범위) | 키체인 | Secret Service(libsecret): 없으면 사용자 전용 권한(0600) 파일 + `warn` 로그 | `chrome.storage.local`(암호화 없음: 문서에 기록합니다) |
| 지문 출처 | `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` | `IOPlatformUUID` | `/etc/machine-id` | 설치 때 만든 난수 ID(`chrome.storage.local`): 재설치는 새 기기 |
| 스케줄러 | 타이머 | 타이머 | 타이머 | `chrome.alarms`: 서비스 워커는 쉬면 내려가므로 메모리 타이머를 믿지 않습니다. |

TypeScript, Go, Python, Kotlin, .NET SDK는 `secret-tool` 명령으로 Linux Secret Service에 접근하며, 비밀 값은 표준 입력으로 전달합니다. 이 명령이 없으면 Secret Service 실행 여부와 관계없이 파일 저장소를 사용합니다.

이 SDK들에는 D-Bus 구현이나 의존성을 추가하지 않고, 판매자가 배포 패키지에 필요한 도구를 의존성으로 지정합니다. .deb에는 `libsecret-tools`, RPM과 Arch에는 `libsecret`을 지정해 주세요.

Rust와 C SDK는 D-Bus로 직접 접근합니다. 근거: ADR-0040 ⚠ 54, 2026-10-19 결정.

모바일의 지문 출처는 Android의 `ANDROID_ID`, iOS의 `identifierForVendor`입니다(안내서 03).

저장소 항목의 서비스 이름은 `rules.storage.service`, 계정 이름은 `clientId`이며 값은 3.6절의 기록입니다. 같은 앱이 SDK 버전을 변경해도 같은 항목을 읽습니다.

## 3. 공통 요청

### 3.1 보내는 것

- 본문은 JSON(`Content-Type: application/json`)입니다. OAuth 두 경로는 폼도 받지만 SDK는 JSON으로 보냅니다.
- `User-Agent: bydev-<언어>/<SDK 판>`을 보냅니다(형식 `rules.userAgent.pattern`). 런타임이 막는 곳(브라우저)은 보내지 않습니다.
- 인증 요청은 `Authorization: Bearer <access_token>`.
- `Idempotency-Key`를 받는 세션 획득·기기 해제 경로는 **논리적 작업 하나에 키 하나**를 `Random.uuid()`로 만듭니다. 같은 작업의 재시도에는 같은 키를 사용하고, `force: true`처럼 본문이 달라지면 새 키를 생성합니다.
- 경로 변수는 퍼센트 인코딩합니다. 질의 문자열은 계약의 이름 그대로.

### 3.2 받은 것 읽기

- 2xx이면 본문을 계약 형식으로 읽습니다. 필요한 필드가 없거나 타입이 다르면 `PROTOCOL_ERROR` 오류를 발생시킵니다.

  모르는 필드는 폐기합니다.
- 2xx가 아니면 본문을 `{ code, message, retryAfterSec?, details? }`로 읽어 오류를 생성합니다. 본문 형식이 다르면 HTTP 상태가 5xx일 때 `INTERNAL_ERROR`, 429일 때 `RATE_LIMITED`, 그 밖에는 `PROTOCOL_ERROR`를 사용합니다.
- `retryAfterSec`가 없고 `Retry-After` 헤더(초)가 있으면 그 값을 씁니다.
- 응답 헤더 `X-Correlation-Id`를 에러의 `correlationId`에 포함합니다.
- 인증 응답의 `X-Lease-Expires-At`는 세션의 리스 만료 시각을 변경합니다(8.5절).

### 3.3 에러 타입

오류 타입은 `BydevError { code, message, httpStatus?, correlationId?, retryAfterSec?, details? }`입니다. `code`는 `contracts`의 `ERROR_CODES`에서 생성한 API 오류 상수 또는 `rules.sdkErrorCodes`의 SDK 코드입니다.

생성기는 두 코드 목록이 겹치지 않는지 검사합니다.

| SDK 코드 | 언제 |
|---|---|
| `NETWORK_ERROR` | 연결 실패 · 시간 초과 · 응답을 받지 못함 |
| `PROTOCOL_ERROR` | 응답이 계약 모양이 아님 |
| `STORAGE_FAILED` | 보안 저장소 읽기 · 쓰기 · 삭제 실패 |
| `NOT_SIGNED_IN` | 저장된 로그인이 없는데 인증 요청을 함 |
| `TOKEN_INVALID` | 토큰 서명 · `typ` · `iss` · `aud` · `exp` · 세션 대조 실패 |
| `GENERATION_REGRESSED` | 이미 본 것보다 작은 세대번호의 토큰 |
| `OFFLINE_EXPIRED` | 하트비트에 성공하지 못한 채 licenseToken이 만료됨 |
| `DOWNLOAD_INTEGRITY` | 다운로드한 파일의 SHA-256이 다름 |
| `LOGIN_CANCELLED` | 앱이 로그인 대기를 취소함 |
| `CONFIGURATION` | 설정이 틀림 · 발견 문서의 `issuer`가 `baseUrl`과 다름 |
| `WEBHOOK_INVALID` | 웹훅 서명 · 시각이 맞지 않음(12절) |

### 3.4 일시적 실패와 끝난 실패

재시도 분류는 다음 순서로 판정합니다.

- `rules.transient.codes`의 `NETWORK_ERROR`, `RATE_LIMITED`, `INTERNAL_ERROR`: 일시적 실패로 분류합니다.
- 그 밖의 알려진 코드: HTTP 상태와 관계없이 재시도하지 않는 실패로 분류합니다. `TAKEOVER_LIMIT_EXCEEDED`와 `DEVICE_RELEASE_LIMIT_EXCEEDED`도 HTTP 429이지만 각각 일일·월간 한도 오류이므로 여기에 해당합니다.
- 모르는 코드: HTTP 상태가 429 또는 5xx이면 일시적 실패로 분류하고, 그 밖의 상태이거나 상태가 없으면 재시도하지 않는 실패로 분류합니다.

일시적 실패의 재시도 시점은 흐름별로 정합니다(5, 6, 8절). 이용권·기기·프로필·업데이트의 단일 호출 함수는 자동 재시도 없이 `retryAfterSec`를 포함한 오류를 반환합니다.

### 3.5 인증 요청 감싸기

인증이 필요한 요청은 모두 이 순서를 따릅니다.

1. 메모리에 access_token이 없거나, `받은 시각 + expires_in − rules.token.refreshSkewSec ≤ now`면 갱신합니다(6절). 저장된 로그인이 없으면 `NOT_SIGNED_IN`.
2. 요청을 보냅니다.
3. `EXPIRED_TOKEN`이면 **한 번만** 갱신하고(메모리 토큰을 버리고) 같은 요청을 다시 보냅니다. 다시 보낸 요청이 또 `EXPIRED_TOKEN`이면 메모리 토큰을
   버리고 그 에러를 반환합니다.

   다시 보낼 때 `Idempotency-Key`는 같은 값입니다.

### 3.6 저장 기록

```json
{ "version": 1, "refreshToken": "rt_…", "scope": "entitlement.read session.write", "savedAt": "2026-10-06T14:00:00Z" }
```

access_token은 저장하지 않습니다(메모리에만). 앱을 다시 켜면 첫 인증 요청에서 갱신합니다. 읽은 기록의 `version`이 모르는 값이면 없는 것으로 봅니다.

## 4. 발견 문서

- 첫 요청 전에 `GET /.well-known/openid-configuration`을 한 번 받아 클라이언트 수명 동안 메모리에 둡니다. 실패하면 두지 않고 다음 요청 때 다시 받습니다.
- `issuer`가 `baseUrl`과 다르면 `CONFIGURATION`. 토큰 · 디바이스 코드 · 폐기 요청은 문서의 `token_endpoint` · `device_authorization_endpoint` ·
  `revocation_endpoint`로, JWKS는 `jwks_uri`로 보냅니다. 나머지 경로는 `baseUrl` + 계약의 경로입니다.

## 5. 로그인 (디바이스 플로우: 안내서 01)

```
startLogin() ──POST device_code──▶ 코드 받음 { userCode, verificationUri, verificationUriComplete, expiresAt }
     │                                   │ (앱이 화면에 띄운다)
     ▼                                   ▼
wait() ── interval × U(1.00, 1.15) 뒤 POST token ──┬─ AUTHORIZATION_PENDING → 같은 간격으로 다시
                                                   ├─ SLOW_DOWN → 간격 = retryAfterSec (없으면 + rules.login.slowDownIncrementSec), 다시
                                                   ├─ 일시적 실패 → 같은 간격으로 다시 (만료 전까지 — 429면 그 값과 retryAfterSec 중 큰 쪽)
                                                   ├─ 토큰 → 저장(6.3) 뒤 성공
                                                   ├─ EXPIRED_TOKEN · INVALID_GRANT · 그 밖 끝난 실패 → 그 에러로 끝
                                                   └─ cancel() → LOGIN_CANCELLED로 끝 (보내던 요청은 결과를 버린다)
```

- `expiresAt = device_code 받은 시각 + expires_in`. 다음 폴링 시각이 `expiresAt`을 넘으면 보내지 않고 `EXPIRED_TOKEN`으로 완료합니다.
- **폴링 지터는 간격을 늘리는 방향으로만 적용합니다**(`rules.login.pollJitter`: 1.00 ~ 1.15). 서버는 `interval`보다 이른 폴링을 `SLOW_DOWN`으로 봅니다(RFC 8628 §3.5 :
  "적어도 interval만큼 기다린다"). 하트비트의 ±15%와 다릅니다.
- **429(`RATE_LIMITED` 또는 HTTP 429인 일시적 실패)면 다음 폴링은 `max(지터 적용 간격, retryAfterSec)` 뒤입니다**(8.3 하트비트와 같은 규칙). 간격 자체는
  바꾸지 않습니다(그 다음 폴링은 다시 `interval`): `SLOW_DOWN`은 이 앱이 빨리 물은 것이고, 429는 같은 IP 뒤의 여러 앱이나 앞단 차단이 원인이라
  한 번만 물러납니다. 한 IP(회사 · 학교 NAT) 뒤에서 여러 앱이 동시에 로그인을 기다릴 때 같은 간격으로 계속 물어 차단을 늘리지 않기 위해서입니다(ADR-0010
  덧붙임 「요청 한도 수치 확정」 ⚠). 그 시각이 `expiresAt`을 넘으면 보내지 않고 `EXPIRED_TOKEN`으로 완료합니다.
- 같은 클라이언트에서 로그인은 한 번에 하나입니다. 진행 중에 `startLogin()`을 다시 부르면 앞의 것을 취소합니다.
- 성공하면 메모리에 access_token · `expires_in` · 받은 시각을 두고, 기록(3.6)을 저장합니다. 저장에 실패하면 새 토큰을 쓰지 않고 `STORAGE_FAILED`로
  완료합니다.

## 6. 토큰 갱신 (안내서 02, ADR-0009 6절)

1. **한 번에 하나**: 갱신이 진행 중이면 새로 보내지 않고 그 결과를 함께 기다립니다(single-flight). 앱의 두 곳이 동시에 불러도 요청은 하나입니다.
2. 저장된 기록의 refresh_token으로 `POST token`(`grant_type=refresh_token`)을 보냅니다. 기록은 저장소에서 읽습니다: 메모리에만 있는 refresh_token을 쓰지
   않습니다(저장하지 못한 토큰을 쓰지 않기 위해).
3. **저장한 뒤에 사용합니다.** 새 refresh_token을 저장한 뒤 새 access_token을 메모리에 둡니다.

   저장에 실패하면 새 토큰 둘 다 폐기하고 `STORAGE_FAILED` 오류를 반환합니다. 다음 갱신은 저장소에 남은 이전 refresh_token으로 요청하며, 서버는 직전 발급 토큰이 사용되지 않았고 발급 시점부터 유예 시간 안인 경우에 재시도를 허용합니다.
4. **응답을 받지 못한 경우 같은 토큰으로 재시도합니다.** 일시적 실패에는 `rules.token.refreshRetryDelaysSec` 간격으로 같은 refresh_token을 다시 전송합니다.

   재시도 간격은 서버의 토큰 회전 유예 시간 안에 요청하도록 정하며, 모두 실패하면 마지막 오류를 반환합니다. 서버 유예 시간은 운영 설정 `refreshTokenRotationGraceSec`을 따릅니다.
5. `INVALID_GRANT`면 로그인이 끝난 것입니다: 기록을 삭제하고(`clear`) 메모리 토큰을 버리고 `INVALID_GRANT`를 반환합니다. 앱은 다시 로그인합니다.
6. 그 밖의 끝난 실패는 기록을 삭제하지 않고 그 에러를 반환합니다.

## 7. 한 번 부르는 함수

| 함수 | 요청 | 돌려주는 것 |
|---|---|---|
| `entitlement()` | `GET /v1/entitlements/me` | 이용권(계약 `Entitlement`). `updateCoverage.type`이 모르는 값이면 `Unknown`. `ENTITLEMENT_NOT_FOUND`는 에러로 |
| `devices({ cursor?, limit? })` | `GET /v1/devices` | `{ items, nextCursor }`: 다음 쪽을 저절로 따라가지 않습니다. |
| `releaseDevice(deviceId)` | `DELETE /v1/devices/{deviceId}` + `Idempotency-Key` | `{ id, status, releasedAt }` |
| `profile()` | `GET /v1/profile` | 프로필 |
| `checkForUpdate({ target, currentVersion })` | `GET /v1/products/{productId}/updates` | 계약 `ProductUpdatesResponse` 모양. `install.mode`가 모르는 값이면 `Unknown`. `entitled`는 있을 때만(`release`는 없으면 null) |
| `verifyDownload(파일, sha256)` | 없음 | 파일의 SHA-256(소문자 16진)이 같으면 끝, 다르면 `DOWNLOAD_INTEGRITY` |
| `verifyToken(token)` | JWKS(9절) | 검증된 licenseToken 클레임 |
| `logout()` | `POST revoke`(refresh_token) | `{ serverRevoked }`: 아래 |

- **로그아웃**: 세션이 돌고 있으면 먼저 중단합니다(8.7). 폐기 요청을 **한 번** 보냅니다. 결과와 상관없이 기록을 삭제하고 메모리 토큰을 폐기합니다.

  서버 폐기에 성공했는지를
  `serverRevoked`로 알립니다: 실패해도 이 기기에는 로그인이 남지 않습니다. 기록 삭제에 실패하면 `STORAGE_FAILED`.
- **업데이트**: `entitled`가 있으면 앱은 `latest` · `update` · `install` 대신 `entitled.release`를 따라야 한다고 문서에 기록합니다(범위 밖 버전을 설치하면 세션 획득이 거절된다). `update.status`가 `REQUIRED`면 앱이 실행을 막아야 한다고 문서에 기록합니다. 다운로드한 파일은 설치 전에 `verifyDownload`. `downloadUrl`은 만료되니
  저장하지 않습니다.

## 8. 세션 (안내서 03, ADR-0008)

### 8.1 상태

판별 유니온. 이름은 언어 관례로 대소문자만 변경합니다.

| 상태 | 필드 | 뜻 |
|---|---|---|
| `Starting` | `attempt`, `lastError` | 획득 중. 일시적 실패면 `rules.session.startRetryDelaysSec` 간격으로 다시(목록 끝은 마지막 값 반복) |
| `Active` | `session`(`id` · `productId` · `deviceId` · `generation` · `leaseExpiresAt` · `heartbeatSec`), `licenseToken`(`token` · `expiresAt`), `claims` | 세션을 들고 있습니다. 앱은 `claims`로 기능을 엽니다. |
| `Degraded` | `Active`의 필드 + `consecutiveFailures`, `offlineUntil`, `lastError` | 하트비트가 일시적으로 실패 중. `offlineUntil`(= licenseToken `expiresAt`)까지는 계속 써도 됩니다. |
| `NeedsDecision` | `reason`, `activeSession`, `maxDevices`, `correlationId` | 사용자가 선택해야 합니다: 아래 |
| `Locked` | `code`, `message`, `correlationId`, `details`, `retrying` | 쓰면 안 됩니다. `retrying`이 참(`OFFLINE_EXPIRED`)이면 연결이 돌아올 때 스스로 해제됩니다. |
| `Stopped` | 추가 필드 없음 | 세션이 중단되었습니다. |

`NeedsDecision.reason`:

| reason | 사용자에게 | 앱이 부르는 것 |
|---|---|---|
| `DEVICE_IN_USE` | "`activeSession.deviceName`에서 사용 중입니다. 여기서 사용할까요?" | `takeOver()` 또는 `stop()` |
| `INSTANCE_ALREADY_RUNNING` | "이 기기에서 이미 실행 중입니다" (크래시 직후라면 인수) | `takeOver()` 또는 `stop()` |
| `DEVICE_LIMIT_EXCEEDED` | "기기 수(`maxDevices`)를 넘었습니다" → 기기 목록에서 하나 해제 | `releaseDevice()` 뒤 `retry()` |

### 8.2 획득

`start()`는 `Stopped`에서만 의미가 있습니다(다른 상태면 무시). 실행마다 새 `instanceId`(`Random.uuid()`)를 만들고 `Starting`이 됩니다.

`POST /v1/sessions`에 `{ device: { fingerprintHash, name, platform }, instanceId, force }`와 새 `Idempotency-Key`를 보냅니다. 설정에 `app`이 있으면 본문에 `app: { target, version }`도 싣습니다(인수 · 재획득도 같다). 일시적 실패의 재시도에는 같은 키를 사용합니다.

| 결과 | 다음 상태 |
|---|---|
| 200 + 검증 통과(9절) | `Active` → 하트비트 예약(8.3) |
| 200 + 검증 실패 | `Locked`(`TOKEN_INVALID` · `GENERATION_REGRESSED`) |
| `DEVICE_IN_USE` · `INSTANCE_ALREADY_RUNNING` · `DEVICE_LIMIT_EXCEEDED` | `NeedsDecision` |
| 일시적 실패 | `Starting`(`attempt + 1`, `lastError`) → 재시도 예약 |
| `INVALID_GRANT`(갱신에서) | `Locked`: 기록은 6절에 따라 삭제되어 있습니다. |
| 그 밖 끝난 실패(`ENTITLEMENT_*`(`ENTITLEMENT_VERSION_NOT_COVERED` 포함) · `TAKEOVER_LIMIT_EXCEEDED` · `PRODUCT_*` · 설정 오류 등) | `Locked`(`details` 그대로) |

- `takeOver()`: `NeedsDecision`(`DEVICE_IN_USE` · `INSTANCE_ALREADY_RUNNING`) 또는 `Locked`(`SESSION_SUPERSEDED`)에서 같은 `instanceId`로 `force: true` 획득을
  **새 `Idempotency-Key`**로 보냅니다(본문이 달라서: 같은 키면 `IDEMPOTENCY_KEY_CONFLICT`). 상태는 `Starting`부터 같은 표를 따릅니다.
- `retry()`: `NeedsDecision`(`DEVICE_LIMIT_EXCEEDED`) 또는 재시도하지 않는 `Locked`에서 `force: false` 획득을 새 키로 다시 합니다.

### 8.3 하트비트

- 다음 하트비트는 마지막 성공 시각 + `round(heartbeatSec × 1000 × j)` ms, `j = rules.session.heartbeatJitter.min + (max − min) × Random.float()`.
- 실패 뒤의 다음 시도도 같은 식(새 난수)입니다. 429면 그 값과 `retryAfterSec` 중 큰 쪽.
- 요청 `POST /v1/sessions/{sid}/heartbeat` `{ generation }`.

| 결과 | 다음 상태 |
|---|---|
| 200 + 검증 통과 | `Active`(새 토큰 · 리스 · 간격). `consecutiveFailures`는 0 |
| 200 + 검증 실패 | `Locked`(`TOKEN_INVALID` · `GENERATION_REGRESSED`) |
| 일시적 실패 | `Degraded`(`consecutiveFailures + 1`, `offlineUntil` = 지금 토큰의 `expiresAt`) → 재시도 예약 |
| `SESSION_SUPERSEDED` | `Locked`(`SESSION_SUPERSEDED`): **즉시 사용 중단**. 스스로 다시 얻지 않습니다(두 기기가 서로 뺏지 않게) |
| `SESSION_REVOKED` | **한 번** 다시 획득합니다(8.2, `force: false`, 같은 `instanceId`, 새 키). 리스가 만료되었을 때도 이 코드를 사용합니다(ADR-0008 3절). |
| `ENTITLEMENT_*` 등 끝난 실패(범위가 줄어 받는 `ENTITLEMENT_VERSION_NOT_COVERED` 포함) | `Locked` |

### 8.4 오프라인 유예

`Degraded`에서 `now ≥ offlineUntil`이 되면(재시도가 그 뒤로 예약되어 있으면 그 시각에 맞춰 상태를 바꾸는 작업을 따로 예약합니다) `Locked`(`OFFLINE_EXPIRED`,
`retrying: true`)가 됩니다. 이때도 하트비트 재시도는 계속합니다.

연결이 돌아오면 리스가 끝나 `SESSION_REVOKED`를 받으므로 8.3대로 다시 얻어 `Active`가 됩니다.

### 8.5 리스 연장

세션을 들고 있는 동안 다른 인증 요청의 응답에 `X-Lease-Expires-At`가 있으면 `session.leaseExpiresAt`을 그 값으로 바꾸고 상태를 알립니다. licenseToken과
`offlineUntil`은 바뀌지 않습니다: 하트비트 일정도 유지됩니다.

### 8.6 인증 실패

세션 요청의 `EXPIRED_TOKEN`은 3.5절대로 한 번 갱신하고 다시 보냅니다. 갱신이 `INVALID_GRANT`면 `Locked`(`INVALID_GRANT`), 갱신이 일시적 실패면 그 요청이
일시적 실패한 것으로 봅니다.

### 8.7 멈춤

`stop()`을 호출하면 예약한 작업을 모두 취소하고, 세션 ID가 있으면 `DELETE /v1/sessions/{sid}`를 **한 번** 보냅니다. 반납 요청의 성공 여부와 관계없이 `Stopped`로 전환하고 진행 중인 요청의 결과는 폐기합니다.

반납하지 못한 리스도 유효기간이 지나면 만료됩니다.

### 8.8 알림

상태가 바뀔 때마다(같은 이름의 상태라도 필드가 바뀌면) 구독자에게 새 상태 하나를 알립니다. 알림은 상태를 바꾼 작업 안에서 순서대로 합니다.

## 9. 토큰 검증 (안내서 09 · 12장 1절)

licenseToken(세션 응답마다)과 `verifyToken()`이 같은 함수를 씁니다.

1. 헤더 `alg`가 `EdDSA`, `typ`가 `license+jwt`(access_token을 검증할 때는 `at+jwt`).
2. `kid`로 JWKS 키를 선택합니다(9.1).
3. Ed25519 서명.
4. `iss` = 발견 문서의 `issuer`, `aud` = `clientId`, `exp > now − rules.token.clockLeewaySec`.
5. 세션 응답이면 `sid` = `session.id`, `device_id` = `session.deviceId`, `generation` = `session.generation`.
6. 같은 (`sub`, `device_id`)에서 이미 본 가장 큰 `generation`보다 작으면 `GENERATION_REGRESSED`. 본 값은 클라이언트 수명 동안 메모리에 둡니다.

1~5 중 하나라도 충족하지 않으면 `TOKEN_INVALID` 오류를 반환합니다. JWKS를 받지 못한 일시적 실패는 토큰 검증 실패로 분류하지 않고 일시적 실패로 처리합니다.

### 9.1 JWKS 캐시

- 받은 응답의 `Cache-Control: max-age`(없으면 `rules.jwks.defaultMaxAgeSec`)까지만 둡니다. 지난 캐시는 쓰지 않고 다시 받습니다.
- 다시 받으면 이전 목록에 합치지 않고 목록 전체를 교체합니다. 새 목록에 없는 키는 신뢰하지 않습니다.
- 모르는 `kid`면 다시 받습니다. 단 마지막으로 받은 지 `rules.jwks.unknownKidRefetchMinIntervalSec`이 안 됐으면 받지 않고 `TOKEN_INVALID`. 다시 받아도
  모르면 `TOKEN_INVALID`.
- 키는 `kty=OKP` · `crv=Ed25519`만 씁니다. 다른 키는 폐기합니다.

## 10. 기기 지문

지문 해시는 `fingerprintHash = lower_hex(SHA-256(UTF-8(출처 값) ‖ UTF-8(rules.fingerprint.salt)))`로 계산합니다. 출처 값의 앞뒤 공백을 제거하고, 해시를 생성한 뒤에는 메모리에서도 원본 값을 폐기합니다.

운영체제 식별자가 없는 곳(브라우저 확장 · 읽기 권한이 없는 Linux)의 기본 출처 포트는 **설치 ID를 스스로 만들어 보관**합니다(확장은 `chrome.storage.local`,
그 밖은 앱 데이터 폴더의 파일). 출처 포트가 값을 주지 못하면(없음 · 빈 값 · 실패) SDK는 그 클라이언트 수명 동안만 쓰는 `Random.uuid()`를 출처로 쓰고
`warn` 로그를 기록합니다: 실행마다 새 기기로 등록될 수 있으니 앱이 출처 포트를 주어야 한다는 뜻입니다.

**한 실행 동안 지문은 하나입니다.** SDK는 처음 계산한 지문 해시(물러난 `Random.uuid()`의 해시 포함)를 그 클라이언트 수명 동안 다시 씁니다. 획득 ·
인수 · 재획득(`retry()` · `SESSION_REVOKED` 뒤)마다 출처 포트를 다시 읽지 않습니다. 실행 중에 다시 읽으면 일시적인 읽기 실패 하나로 같은 PC가 다른 기기로
등록되어 기기 수 한도(1대 이용권이면 바로)를 넘습니다(1.2.0, 2026-11-20 Windows 확인: 같은 실행에서 MachineGuid 읽기가 한 번 실패해 설치 ID로 등록됐다).

**기본 출처 포트는 물러나기 전에 다시 읽습니다.** 운영체제 식별자를 읽다가 실패하거나 빈 값을 받으면 설치 ID로 물러나기 전에 모두
`rules.fingerprint.sourceReadAttempts`번까지 읽고, 사이마다 `rules.fingerprint.sourceReadRetryDelayMs`를 기다립니다. 표준 라이브러리 · 대상 런타임으로
레지스트리를 읽을 수 있는 언어(Go `syscall` · Python `winreg` · .NET 8 `Microsoft.Win32.Registry`)는 명령(`reg query`)을 실행하기 전에 그것으로 읽습니다.
GUI 앱이 띄운 `reg.exe`는 2026-11-20 Windows 11 가상 머신에서 순차 200번 가운데 10번(5%)이 `0xC0000142`(초기화 실패)로 끝났고 한 번에 평균 5.4초가
걸렸지만, 레지스트리 API는 400번 모두 바로 읽었습니다. API가 없는 언어(TS/JS · Kotlin/Java · `unsafe`를 금지한 Rust)는 명령 + 재시도입니다.

## 11. 하지 말 것

- 서버 응답의 성공 여부만 보고 기능을 열게 하는 API(`isLicensed()`).
- 정책 수치를 상수로(하트비트 간격 · 토큰 수명 · 폴링 간격).
- 토큰 · refresh_token · 지문 출처 값을 로그에.
- 갱신 요청 둘을 동시에.
- 저장하지 못한 refresh_token을 쓰기.
- 안내서에 없는 서버 호출 · 수집.

## 12. 서버 도우미 (판매자 서버에서도 도는 언어: ADR-0040 5절)

판매자 서버가 쓰는 두 함수. 앱 쪽 클라이언트와 상태를 나누지 않습니다.

**웹훅 서명 검증** `verifyWebhook({ secrets, headers, rawBody })`: 안내서 11장 · Standard Webhooks v1.

1. 본문은 **받은 바이트 그대로**(파싱 · 재직렬화 전)를 씁니다.
2. 헤더 `webhook-id` · `webhook-timestamp` · `webhook-signature`가 다 있어야 합니다.
3. `|now − webhook-timestamp| > rules.webhook.toleranceSec`이면 거부.
4. 서명 대상 `${id}.${timestamp}.${rawBody}`, HMAC-SHA256, 키는 시크릿에서 `rules.webhook.secretPrefix`를 뗀 base64를 디코딩한 바이트.
5. `webhook-signature`는 공백으로 구분된 `v1,<base64>` 목록입니다(시크릿 교체 중). 시크릿도 여러 개를 받습니다(교체 중인 판매자 쪽). 어느 조합이든 하나라도
   맞으면 유효: 비교는 상수 시간으로.
6. 실패는 `WEBHOOK_INVALID`. 성공하면 `{ id, timestamp }`를 반환합니다: 중복 수신은 `id`로 판매자가 필터링합니다.

**토큰 검증** `createServerVerifier({ baseUrl, clientId })`: 안내서 09장. 9절과 같은 JWKS 캐시 · 같은 검증(`typ` · 서명 · `iss` · `aud` · `exp`)을
쓰되 세션 대조(9절 5)는 없습니다. access_token(`at+jwt`)과 licenseToken(`license+jwt`)을 따로 호출합니다.

세대번호 단조 확인(9절 6)은 판매자 서버가 본 값을
보관해야 하므로 앱이 이미 본 가장 큰 세대번호를 넘기면(`minGeneration`) 그보다 작은 토큰을 `GENERATION_REGRESSED`로 거부합니다.

## 13. 버전

이 명세의 버전은 `rules.specVersion`(SemVer)입니다. 동작이 바뀌면 버전을 올리고, 시나리오가 그 버전을 기록합니다.

SDK의 버전은 각자 SemVer이고 README에 지원하는
명세 버전 · API(v1) · 최소 런타임을 기록합니다.
