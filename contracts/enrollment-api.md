# 계약 — enrollment API

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (Go 클라이언트) ↔ **`pulsemetry-backend`** (`apps/enrollment-api`) |
| 기계 판독 원본 | `telemetryctl/contracts/enrollment-manifest.schema.json`, `enrollment-envelope.schema.json`. §7(설치 보고)은 이 문서의 표가 원본이다 |
| 서버 측 상세 | `pulsemetry-backend/docs/enrollment-server-spec.md` |
| 관련 ADR | backend ADR-0003(계약과 2단 토큰), 0005(부트스트랩·바이너리 서빙), 0007(인증 계층), 0009(스키마 enum) · 허브 [ADR 0010](../adr/0010-installation-heartbeat-and-policy-acknowledgement.md)(설치 보고) |
| 상태 | 확정 — §7(설치 보고)은 변경 중(PROJ-187) |

관리자가 발급한 일회성 초대 코드를 검증·소비해 사용자 PC의 설치(installation)를 만들고,
그 설치에 귀속되는 자격증명과 회사 단위 OTel 설정(manifest)을 내려주는 계약이다.

## 1. 엔드포인트

| 메서드 | 경로 | 인증 | 성공 | 호출자 |
|---|---|---|---|---|
| POST | `/v1/enroll` | 없음 (초대 코드 자체가 자격) | 201 | telemetryctl |
| POST | `/v1/installations/telemetry-token` | `Authorization: Bearer <pit_…>` | 200 | telemetryctl (데몬) |
| POST | `/v1/installations/{installation_id}/heartbeat` | `Authorization: Bearer <pit_…>` | 200 | telemetryctl (데몬) — §7 |
| POST | `/v1/invitations` | `X-Admin-Token` | 201 | 관리자 |
| POST | `/v1/invitations/{id}/revoke` | `X-Admin-Token` | 204 | 관리자 |
| GET | `/windows?code=…` / `/unix?code=…` | 없음 | 200 `text/plain` | 사용자 셸 |
| GET | `/bin/{filename}` | 없음 | 200 `application/octet-stream` | 부트스트랩 스크립트 |
| GET | `/api/v1/check-updates` | 없음 | 200 | telemetryctl (데몬) — [`daemon-updates.md`](daemon-updates.md) |
| GET | `/v1/healthz` | 없음 | 200 | 운영 |

스크립트·바이너리 경로에 `/v1` 접두사가 없는 것은 의도다 — 사용자가 터미널에 붙여넣는 URL이라 짧아야 한다.

### 바이너리 산출물 이름 규칙 — 이 문서가 소유한다

`GET /bin/{filename}`이 서빙하는 파일명은 **정확히 여섯 개**다. 만드는 쪽(`telemetryctl` 릴리스)과
쓰는 쪽(backend `BinaryController` 화이트리스트, 부트스트랩 스크립트)이 서로 다른 레포라 이 목록이 계약이다.

`pulsemetry_windows_amd64.exe` · `pulsemetry_windows_arm64.exe` · `pulsemetry_darwin_amd64` ·
`pulsemetry_darwin_arm64` · `pulsemetry_linux_amd64` · `pulsemetry_linux_arm64`

규칙은 `pulsemetry_{os}_{arch}`(Windows만 `.exe`)다. 이 이름은 서빙 URL의 이름이다.

`telemetryctl` 릴리스는 데몬을 `pulsemetry_cli_{os}_{arch}`(Windows만 `.exe`)로 내고 `SHA256SUMS`를 함께 낸다.
backend는 공개 이름을 같은 대상의 릴리스 자산으로 대응해 서빙한다 — 릴리스 디렉터리의 배치와 해시 확인은
[`daemon-updates.md`](daemon-updates.md) §3이 정한다. 릴리스 이름을 공개 이름으로 바꿔 놓는 별도 단계는 없다.

## 2. `POST /v1/enroll`

**요청**

| 필드 | 값 | 비고 |
|---|---|---|
| `code` | `XXXX-XXXX-XXXX` | Crockford Base32 12자 |
| `platform` | 클라이언트의 `runtime.GOOS` 원문 | macOS는 `darwin`으로 도착. **서버가 `macos`로 정규화**해 저장. `windows`·`linux`는 그대로. 그 밖은 400 |
| `architecture` | | |
| `hostname` | | |
| `client_version` | | |
| `invite` | `""` | **deprecated. 그러나 제거 금지** — Go 쪽에 `omitempty`가 없어 항상 전송되고 서버가 이를 수용한다 |

서버는 unknown 필드를 400으로 거부한다.

**응답 (201) — 정확히 4키**

```json
{ "installation_id": "...", "installation_token": "pit_...", "telemetry_token": "ptt_...", "manifest": { } }
```

클라이언트가 `DisallowUnknownFields`로 파싱하므로 **키를 추가하면 배포된 전 클라이언트가 깨진다**(§6 M7).

**초대 코드 소비는 조건부 UPDATE 한 문장이다.**

```sql
UPDATE enrollment.invitations SET used_at = :now
WHERE code_hash = :codeHash AND used_at IS NULL AND revoked_at IS NULL AND expires_at > :now
```

`SELECT` 후 `UPDATE` 하지 않는다. 동시 요청 N개가 같은 코드로 들어와도 **정확히 하나만 201**을 받고
나머지는 409 `invitation_used`다.

**enroll 성공은 대상 멤버의 `invited → active` 전환 이벤트다.** OTLP 경로의 auth-proxy가
`invited` 멤버의 토큰을 거부하므로, 이 전환 없이는 발급된 telemetry token이 전부 401이 된다
([`telemetry-ingest.md`](telemetry-ingest.md) §3). 재발급(`POST /v1/installations/telemetry-token`)도
같은 전환을 보정한다 — `pit_` 인증이 과거 enroll 완료의 증명이기 때문이다.
전환은 `invited`에서만 일어난다. **`suspended`는 어느 경로도 건드리지 않는다** — 정지 해제는 관리자의 결정이지 설치의 부수효과가 아니다.

## 3. 2단 토큰 모델

| 토큰 | 접두사 | 저장 위치 | 용도 | 교체 |
|---|---|---|---|---|
| `installation_token` | `pit_` | OS 키링 | 이 설치의 장기 신원 — `ptt_` 재발급 요청과 설치 보고(§7)의 인증 | 하지 않는다 |
| `telemetry_token` | `ptt_` | OS 키링 (데몬이 상위 전송 시 `Authorization`에 주입) | 텔레메트리 전송 | 언제든 재발급 |

형식은 둘 다 `접두사 + base64url(32 랜덤 바이트, 패딩 없음)`.

**`ptt_`는 벤더 설정 파일로 나가지 않는다(로컬 파이프라인 배선 시).** enroll이 로컬 파이프라인을 배선하면서 Codex/Claude 설정에는
**로컬 ingest 토큰**이 들어가고, 회사 `ptt_`는 OS 키링에 있다가 데몬이 상위 전송 시 헤더에 주입한다.
단 배선이 강등된(회사 직결 — grpc 테넌트·키링 불가 등, [`telemetry-ingest.md`](telemetry-ingest.md) §6) 설치에서는
벤더 설정의 `Authorization`에 `ptt_`가 실린다.

**봉투 분리** — `installation_id`와 두 토큰은 manifest **밖**, 응답 봉투 상위에 둔다. 이유는 둘이다.
① 설정 재조회 API가 생겼을 때 매번 secret을 실어 나르지 않기 위해.
② 클라이언트의 `DisallowUnknownFields`가 **중첩 manifest까지 적용**되므로, manifest 안에 봉투 필드가 하나라도 있으면 설치가 그 자리에서 실패한다.

## 4. 해시 — ★ 가장 자주 어긋나는 지점

| 대상 | 방식 |
|---|---|
| 초대 코드 | **SHA-256** hex 소문자 64자 (무염) |
| `installation_token` | **SHA-256** hex 소문자 64자 (무염) |
| **`telemetry_token`** | **HMAC-SHA256(`pulsemetry.token-hash-secret`, 토큰 전문)** hex 소문자 64자 |

`telemetry_token`만 HMAC인 이유는 **auth-proxy가 같은 키·같은 연산으로 `token_hash`를 조회하기 때문**이다.
`ai-telemetry-pipeline`의 `apps/auth-proxy/src/shared/crypto/token-hash.ts`는
`createHmac("sha256", TOKEN_HASH_SECRET).update(token, "utf8").digest("hex")`를 쓴다.
**양쪽이 같은 시크릿을 받아야 한다** — dev 인프라에서는 `infra`의 `DevEdgeStack` CfnOutput `TokenHashSecretArn`
(Secrets Manager)이 원본이고, backend는 `PULSEMETRY_TOKEN_HASH_SECRET`, auth-proxy는 `TOKEN_HASH_SECRET`으로 받는다.
prod의 공유 방식은 enrollment 서버 배치와 함께 미결이다(infra `AGENTS.md` 5장 (H)).

> **이 세 값이 갈라지면 발급된 모든 토큰이 401이 된다.** 한쪽만 고치는 PR을 열지 않는다.

해시가 결정론적이어야 유니크 인덱스 조회가 성립하므로 bcrypt·Argon2를 쓸 수 없다.
원본이 고엔트로피 난수(토큰 256비트, 초대 코드 60비트)라 사전 공격 대상이 아니라는 전제 위에 서 있으며,
**사람이 고른 비밀번호에는 이 방식을 쓸 수 없다.**

토큰과 초대 코드의 원본은 DB에 저장하지 않고 **로그·에러 응답에도 담지 않는다** —
파싱 실패 메시지에는 요청 본문 조각이 섞여 있어 그대로 흘리면 코드가 새어 나간다.

### 초대 코드 pepper — 해소됨(PROJ-79) · 시드 일원화됨(PROJ-105)

`ai-telemetry-pipeline`의 `sql/rds/seed.sql`이 `HMAC-SHA256('dev-only-invite-pepper', code)`를 전제해
backend의 무염 SHA-256과 어긋나 있었다. **시드는 dev 편의용이고 진실원은 backend다** — 시드를 backend
방식(무염 SHA-256)으로 맞췄다(pipeline `6543e6d`).

PROJ-105에서 **dev 시드 자체가 backend로 모였다.** 지금은 backend `tools/dev-seed`(Docker 전용 — backend ADR 0031)가
tenant·member·팀·소속·manifest·초대를 넣는다. 시드가 두 벌이면 어느 쪽이 사실인지가 실행 순서로 정해지고,
SQL 시드는 backend Flyway보다 먼저 돌아 스키마가 없는 시점에 적용된다. 해시 방식이 갈릴 여지도 함께 사라졌다 —
시드 도구가 서버와 같은 무염 SHA-256을 쓴다.

## 5. Manifest

회사 단위 OTel 설정. 계약의 기계 판독 원본은 `telemetryctl/contracts/enrollment-manifest.schema.json`이다.

```json
{
  "schema_version": 1,
  "config_revision": 1,
  "otlp": { "endpoint": "https://...", "protocol": "http/protobuf", "compression": "gzip", "timeout_ms": 10000 },
  "signals": { "logs": false, "metrics": true, "traces": true },
  "privacy": { "collect_user_prompts": false, "collect_assistant_responses": false,
               "collect_tool_details": false, "collect_tool_content": false,
               "collect_user_email": false, "collect_raw_api_bodies": false },
  "repository_allowlist": [],
  "resource_attributes": {}
}
```

- 저장은 `enrollment.manifests.manifest`(jsonb). 기존 행을 고치지 않고 **새 `version` 행**을 만든다.
  tenant당 `is_active = true` 행은 **최대 하나** — 부분 유니크 인덱스가 보장한다.
- enroll 응답에는 저장된 manifest를 싣되 **`config_revision`만 `manifests.version`으로 덮어쓴다.**
- 활성 manifest가 없거나 저장된 manifest가 계약을 어기면 enroll은 **409 `manifest_not_configured`**로 실패한다.
- 클라이언트가 한 번 더 검증한다: `otlp.endpoint`는 **https 필수**(`http`는 `localhost`만),
  `protocol`은 `http/protobuf`·`http/json`·`grpc`, `compression`은 `none`·`gzip`,
  `schema_version`이 클라이언트의 `SupportedSchemaVersion`(현재 1)을 넘으면 거부.
- **`manifest` 작성 API가 없다.** 테넌트 온보딩 전 수동 INSERT가 선행돼야 한다.

### 집행되지 않는 manifest 필드

계약 스키마에 있지만 현재 어느 레포도 집행하지 않는 필드가 둘이다. 결정 공백이 아니라 **의도된 미착수**다.

- **`repository_allowlist`** — 어느 레포도 읽어서 판단하지 않는다(telemetryctl은 파싱·복제만 한다).
  근거 정책인 설치 아키텍처 §4.3은 **"권장"** 표기이고 스키마 `description`도 같다 — MVP 범위 밖이다.
  집행 주체(클라이언트 vs 파이프라인)를 정하는 일은 두 레포 이상에 걸리므로, 착수 시 허브 ADR로 결정한다.
- **`resource_attributes`** — 완전 미사용은 아니다. Codex의 `deployment.environment` 한 키에만 쓰이고
  나머지는 어디에도 반영되지 않는다.

[`telemetry-ingest.md`](telemetry-ingest.md) §5 M3가 이 절을 가리킨다.

## 6. 미해결

| # | 항목 | 영향 |
|---|---|---|
| ~~M1~~ | **해소됨(PROJ-105)** — 시드가 backend 시드 도구(`tools/dev-seed`) 한 벌로 모였다. 주입할 두 번째 시드가 없으므로 값을 맞출 배선도 필요 없다. endpoint 기본값 `:4316`은 이제 `:apps:telemetry-ingest`가 로컬에서 듣는 포트이며, 구 auth-proxy의 자리를 물려받은 것이라 데몬 설정을 바꾸지 않는다 | (기존 위험이던 `:4318` 하드코딩 — auth-proxy 우회·자기참조 — 은 제거됨) |
| M7 | **계약 진화 취약** — 클라이언트 `DisallowUnknownFields` + 서버 `FAIL_ON_UNKNOWN_PROPERTIES`. 응답 필드 하나만 추가해도 배포된 전 클라이언트가 파괴된다 | 버저닝 또는 tolerant reader 정책이 필요하다. **현재는 필드 추가가 breaking change다** |
| M8 | enrollment HTTP 클라이언트에 **타임아웃이 없고** 3xx 리다이렉트를 따라가며 초대 코드를 재전송한다 | 무한 대기, 코드 유출 |
| M9 | manifest `protocol: "grpc"`는 서버 검증을 통과하지만 클라이언트가 상위 전송을 지원하지 않아 로컬 파이프라인 배선에서 제외된다(회사 직결 강등 — [`telemetry-ingest.md`](telemetry-ingest.md) §6). 강등 상태에서는 포워더 `Scrub`이 경로 밖이라 **manifest `privacy` 집행에 공백이 생긴다**(같은 문서 §5 M13). 현재 grpc 테넌트는 없다 | 서버에서 grpc를 막거나 클라이언트에 구현해야 한다 |
| ~~M10~~ | **계약으로 해소(PROJ-187, 변경 중)** — 설치의 생존·수집 상태·적용한 manifest 판 보고는 §7(heartbeat)이, 설정 재조회는 [`user-auth.md`](user-auth.md)의 `GET /v1/manifest`가 맡는다. 데몬은 heartbeat 응답의 기대 판과 적용한 판을 비교해 재조회한다. 토큰은 그대로다 — `pit_`는 교체하지 않고 `ptt_`는 언제든 재발급한다(§3) | manifest 변경이 기존 설치에 전파되고 확인되는 경로가 생긴다 ([`../product/prd.md`](../product/prd.md) §8-1) |
| — | `/v1/enroll`에 rate limit이 없다 | 60비트 초대 코드의 유일한 브루트포스 표면이 무방비 |
| — | `POST /v1/invitations/{id}/revoke`에 테넌트 격리가 없다 | 정적 admin 키 보유자가 전 테넌트 revoke 가능 |
| — | `--force` 플래그가 받기만 하고 아무 동작도 하지 않는다(엔드포인트 충돌 감지 미구현) | 사용자 기대와 불일치 |
| — | **해소됨(PROJ-79)** — `privacy.collect_raw_api_bodies`를 `required`에 추가해 Go 구조체와 대칭이 됐다(telemetryctl `8268a3a`. backend 계약 테스트가 갱신된 스키마 원본으로 통과). **기존 저장 manifest 주의** — `required` 추가라 이 필드가 없는 기존 v1 manifest는 서버 검증에서 409 `manifest_not_configured`가 된다. 테넌트 온보딩 전 수동 INSERT 점검이 필요하다 | — |

## 사용자 인증 계약 추가 (PROJ-107)

가입·로그인 JSON API와 CLI 코드 교환은 [사용자 인증 계약](user-auth.md)을 따른다. 기존 설치 봉투는 유지한다. 상태: 변경 중.

## manifest 재동기화 추가 (PROJ-108, 변경 중)

사용자 RT 기반 GET /v1/manifest는 [사용자 인증 계약](user-auth.md)의 별도 5키 봉투를 사용한다.
설치 enroll 4키와 중첩 manifest 스키마는 변경하지 않는다. GET 상태 변경·캐시 금지·응답 유실 정책도 그 계약을 따른다.

초대 폐기는 가입/설치 중 미소비 권한이 남은 경우 허용한다. 양쪽 완료만 invitation_used로 거부한다. 기존 설치·계정은 유지한다.

## 7. 설치 보고 — `POST /v1/installations/{installation_id}/heartbeat` (PROJ-187, 변경 중)

데몬이 주기적으로 자기 상태를 보고한다. 생존, 수집 경로의 상태, **적용한 manifest 판**이 한 요청에 실린다.
정책 적용 확인(ACK) 전용 경로는 없다 — 적용한 판은 모든 보고에 실리는 상태다(허브 [ADR 0010](../adr/0010-installation-heartbeat-and-policy-acknowledgement.md)).
요청·응답의 필드는 아래 표가 원본이다. 기계 판독 스키마 파일은 아직 없다.

### 요청

`Authorization: Bearer <pit_…>`, `Content-Type: application/json`. `ptt_`와 사용자 토큰은 받지 않는다.
경로의 `installation_id`가 자격증명의 설치와 다르면 403이다.

```json
{
  "sent_at": "2026-09-30T01:05:00Z",
  "daemon": { "version": "0.2.0", "platform": "darwin", "architecture": "arm64", "run_id": "b3f1c2a49d5e4f60a1b2c3d4e5f60718" },
  "applied_config_revision": 3,
  "collection": {
    "mode": "local",
    "receiving_since": "2026-09-30T00:00:01Z",
    "forwarding": true,
    "delivered": 120,
    "lost": 0,
    "pending": 2,
    "last_delivered_at": "2026-09-30T01:04:30Z"
  }
}
```

| 필드 | 타입 | 뜻 |
|---|---|---|
| `sent_at` | 시각 | 데몬이 이 보고를 만든 시각(데몬 시계) |
| `daemon.version` | 문자열(1~64자) | 실행 중인 데몬의 버전 |
| `daemon.platform` | `darwin` · `linux` · `windows` | `runtime.GOOS` 원문. 서버가 enroll과 같은 규칙으로 정규화한다(§2) |
| `daemon.architecture` | 문자열(1~32자) | `runtime.GOARCH` 원문 |
| `daemon.run_id` | 문자열(16~64자, `[A-Za-z0-9_-]`) | 데몬 프로세스가 뜰 때 만든 무작위 식별자. 같은 프로세스는 같은 값을 보낸다. 누적 개수의 세대를 가른다 |
| `applied_config_revision` | 정수 ≥ 1 | 데몬이 **지금 집행하는** manifest의 `config_revision`. 벤더 설정과 상위 전달의 시그널·프라이버시 집행이 그 판으로 돌고 있다 |
| `collection.mode` | `local` · `direct` | `local`: 벤더 도구가 로컬 수신기로 보내고 데몬이 회사로 전달한다. `direct`: 벤더 설정이 회사를 직접 가리킨다([`telemetry-ingest.md`](telemetry-ingest.md) §6) — 데몬은 그 전송을 보지 못한다 |
| `collection.receiving_since` | 시각 또는 null | 이 프로세스의 로컬 수신기가 끊김 없이 듣기 시작한 시각. 수신기가 떠 있지 않으면 null |
| `collection.forwarding` | 불리언 | 상위 전달기가 떠 있다 |
| `collection.delivered` | 정수 ≥ 0 | 이 프로세스가 뜬 뒤로 회사가 2xx로 받은 페이로드 수 |
| `collection.lost` | 정수 ≥ 0 | 이 프로세스가 뜬 뒤로 회사로 보내야 했지만 전달하지 못하고 버린 페이로드 수(큐 포화, 정리 실패, 4xx 폐기, 재시도 소진). 회사 manifest가 끈 시그널은 세지 않는다 |
| `collection.pending` | 정수 ≥ 0 | 아직 보내는 중이거나 대기 중인 페이로드 수 |
| `collection.last_delivered_at` | 시각 또는 null | 회사가 마지막으로 2xx로 받은 시각. 이 프로세스가 뜬 뒤로 성공이 없으면 null |

- **시각**은 UTC RFC 3339(`Z`)다. `receiving_since`와 `last_delivered_at`은 `sent_at`보다 뒤일 수 없다. 어기면 400이다.
  데몬은 시계가 바뀌어도 이 관계가 깨지지 않게, 두 시각을 `sent_at`에서 단조 시계로 잰 경과 시간을 빼서 만든다.
- **서버는 데몬 시계의 절대값을 쓰지 않는다.** 생존 시각은 서버가 받은 시각이다. 본문의 시각은 (받은 시각 − `sent_at`)만큼 옮겨 쓴다.
- 개수는 페이로드(OTLP 요청 본문) 단위이고 **프로세스 단위 누적**이다. `run_id`가 바뀌면 0부터 다시 센다.
- `mode`가 `direct`이거나 `forwarding`이 false면 `delivered`·`lost`·`pending`은 0, `last_delivered_at`은 null이다.
- 서버는 모르는 키를 무시한다. 데몬은 자기가 알지 못하는 값을 추정해 채우지 않는다.

### 응답 (200)

`Cache-Control: no-store`.

```json
{
  "received_at": "2026-09-30T01:05:00.412Z",
  "expected_config_revision": 4,
  "acknowledged_config_revision": 3,
  "report_interval_seconds": 300
}
```

| 필드 | 타입 | 뜻 |
|---|---|---|
| `received_at` | 시각 | 서버가 이 보고를 받은 시각. 서버가 기록한 생존 시각이다 |
| `expected_config_revision` | 정수 ≥ 1 또는 null | 그 조직의 현재 활성 manifest 판. 활성 manifest가 없으면 null |
| `acknowledged_config_revision` | 정수 ≥ 1 또는 null | 서버가 **적용 확인으로 기록한** 판. 보고한 `applied_config_revision`을 서버가 그 조직의 판으로 갖고 있지 않으면 null |
| `report_interval_seconds` | 정수 60~3600 | 다음 보고까지의 주기 |

- 같은 상태를 여러 번 보고해도 결과가 같다. 적용 확인 시각은 서버가 그 판을 **처음** 보고받은 시각이다.
- `expected_config_revision` > `applied_config_revision`이면 데몬은 manifest를 재조회해야 한다. 재조회는 [`user-auth.md`](user-auth.md)의 `GET /v1/manifest`다. **이 응답은 manifest를 싣지 않는다.**
- `acknowledged_config_revision`이 null이어도 생존은 기록된다.
- 데몬은 모르는 키를 무시한다.

### 주기와 재시도

- 데몬은 기동 직후 한 번 보고하고, 그 뒤로는 마지막 응답의 `report_interval_seconds`마다 보고한다. 응답을 받기 전의 주기는 300초다.
- manifest를 적용한 직후에는 주기를 기다리지 않고 바로 보고한다.
- 주기마다 ±10% 범위의 무작위 지연을 둔다.
- 요청 한 번의 제한은 10초다. 리다이렉트를 따라가지 않는다. 응답 본문은 64 KiB를 넘지 않는다.
- **보고는 다시 보내지 않는다.** 실패한 본문을 보관하지 않고, 다음 시도는 그 시점의 상태로 새로 만든다.

| 상태 | `error` | 뜻 | 데몬의 처리 |
|---|---|---|---|
| 200 | — | 기록됨 | 응답의 주기를 따른다 |
| 400 | `invalid_request` | 본문이 계약을 어긴다 | 이번 보고를 버린다. 다음 주기에 새로 보낸다 |
| 401 | `unauthorized` | 자격증명이 없거나 무효다 | 이번 보고를 버린다. 주기를 3600초로 늘리고 "다시 등록해야 한다"를 로컬 상태에 표시한다 |
| 403 | `forbidden` | 경로의 설치가 자격증명의 설치와 다르다 | 401과 같다 |
| 403 | `installation_revoked` | 폐기된 설치다 | 401과 같다 |
| 503 + `Retry-After` | `heartbeat_unavailable` | 일시 장애(저장소 장애 포함) | `Retry-After`와 데몬 백오프 중 큰 값 뒤에 새 보고로 다시 시도한다. 대기는 보고 주기를 넘지 않는다 |

- 네트워크 오류와 그 밖의 5xx·408·429는 503과 같이 다룬다. 그 밖의 4xx는 400과 같이 다룬다.
- 401·403을 받아도 데몬은 자격증명과 로컬 데이터를 지우지 않는다. 이 응답을 이유로 `ptt_`를 버리거나 재발급하지 않는다.
- 서버는 저장소 장애를 401·403으로 돌리지 않는다.

### 이 보고가 말하지 않는 것

- 벤더 도구가 실제로 텔레메트리를 냈는지, 회사 직결 설치가 무엇을 보냈는지.
- 이전 데몬 프로세스가 종료하며 버린 것, 데몬이 돌지 않던 동안의 일.
- 서버가 받은 뒤의 적재 결과.

따라서 보고가 있었다는 사실만으로 그 기간의 수집이 완전하다고 판정하지 않는다. 판정 규칙은 backend가 정한다.

### 현재 상태

- `telemetryctl` 기본 브랜치에는 이 보고의 송신(클라이언트·주기 작업)이 없다. 서버 경로만 있다.
- 그래서 지금 배포된 데몬의 설치는 보고하지 않는다. 서버는 그 설치의 적용 판을 확인할 수 없고 "확인 불가"로 낸다.
- 응답이 알려 주는 새 판을 데몬이 받는 재조회([`user-auth.md`](user-auth.md)의 `GET /v1/manifest`)도 데몬 쪽 구현이 없다.
  이미 설치된 기기가 새 정책을 받는 방법은 다시 등록하는 것뿐이다.
