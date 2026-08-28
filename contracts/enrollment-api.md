# 계약 — enrollment API

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (Go 클라이언트) ↔ **`pulsemetry-backend`** (`apps/enrollment-api`) |
| 기계 판독 원본 | `telemetryctl/contracts/enrollment-manifest.schema.json`, `enrollment-envelope.schema.json` |
| 서버 측 상세 | `pulsemetry-backend/docs/enrollment-server-spec.md` |
| 관련 ADR | backend ADR-0003(계약과 2단 토큰), 0005(부트스트랩·바이너리 서빙), 0007(인증 계층), 0009(스키마 enum) |
| 상태 | 확정 |

관리자가 발급한 일회성 초대 코드를 검증·소비해 사용자 PC의 설치(installation)를 만들고,
그 설치에 귀속되는 자격증명과 회사 단위 OTel 설정(manifest)을 내려주는 계약이다.

## 1. 엔드포인트

| 메서드 | 경로 | 인증 | 성공 | 호출자 |
|---|---|---|---|---|
| POST | `/v1/enroll` | 없음 (초대 코드 자체가 자격) | 201 | telemetryctl |
| POST | `/v1/installations/telemetry-token` | `Authorization: Bearer <pit_…>` | 200 | telemetryctl (데몬) |
| POST | `/v1/invitations` | `X-Admin-Token` | 201 | 관리자 |
| POST | `/v1/invitations/{id}/revoke` | `X-Admin-Token` | 204 | 관리자 |
| GET | `/windows?code=…` / `/unix?code=…` | 없음 | 200 `text/plain` | 사용자 셸 |
| GET | `/bin/{filename}` | 없음 | 200 `application/octet-stream` | 부트스트랩 스크립트 |
| GET | `/v1/healthz` | 없음 | 200 | 운영 |

스크립트·바이너리 경로에 `/v1` 접두사가 없는 것은 의도다 — 사용자가 터미널에 붙여넣는 URL이라 짧아야 한다.

### 바이너리 산출물 이름 규칙 — 이 문서가 소유한다

`GET /bin/{filename}`이 서빙하는 파일명은 **정확히 여섯 개**다. 만드는 쪽(`telemetryctl` 릴리스)과
쓰는 쪽(backend `BinaryController` 화이트리스트, 부트스트랩 스크립트)이 서로 다른 레포라 이 목록이 계약이다.

`pulsemetry_windows_amd64.exe` · `pulsemetry_windows_arm64.exe` · `pulsemetry_darwin_amd64` ·
`pulsemetry_darwin_arm64` · `pulsemetry_linux_amd64` · `pulsemetry_linux_arm64`

규칙은 `pulsemetry_{os}_{arch}`(Windows만 `.exe`)다. **이 이름으로 산출물을 만드는 릴리스
파이프라인은 `telemetryctl`에 아직 없다**(backend ADR-0005 Follow-up의 서빙 전제 3건 중 하나).

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
| `installation_token` | `pit_` | OS 키링 | 이 설치의 장기 신원 | 하지 않는다 |
| `telemetry_token` | `ptt_` | OS 키링 (데몬이 상위 전송 시 `Authorization`에 주입) | 텔레메트리 전송 | 언제든 재발급 |

형식은 둘 다 `접두사 + base64url(32 랜덤 바이트, 패딩 없음)`.

**`ptt_`는 벤더 설정 파일로 나가지 않는다.** enroll이 로컬 파이프라인을 배선하면서 Codex/Claude 설정에는
**로컬 ingest 토큰**이 들어가고, 회사 `ptt_`는 OS 키링에 있다가 데몬이 상위 전송 시 헤더에 주입한다.

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
**양쪽이 같은 시크릿을 받아야 한다** — 운영에서는 `infra`의 `DevEdgeStack` CfnOutput `TokenHashSecretArn`
(Secrets Manager)이 원본이고, backend는 `PULSEMETRY_TOKEN_HASH_SECRET`, auth-proxy는 `TOKEN_HASH_SECRET`으로 받는다.

> **이 세 값이 갈라지면 발급된 모든 토큰이 401이 된다.** 한쪽만 고치는 PR을 열지 않는다.

해시가 결정론적이어야 유니크 인덱스 조회가 성립하므로 bcrypt·Argon2를 쓸 수 없다.
원본이 고엔트로피 난수(토큰 256비트, 초대 코드 60비트)라 사전 공격 대상이 아니라는 전제 위에 서 있으며,
**사람이 고른 비밀번호에는 이 방식을 쓸 수 없다.**

토큰과 초대 코드의 원본은 DB에 저장하지 않고 **로그·에러 응답에도 담지 않는다** —
파싱 실패 메시지에는 요청 본문 조각이 섞여 있어 그대로 흘리면 코드가 새어 나간다.

### 초대 코드 pepper — 해소됨(PROJ-80)

`ai-telemetry-pipeline`의 `sql/rds/seed.sql`이 `HMAC-SHA256('dev-only-invite-pepper', code)`를 전제해
backend의 무염 SHA-256과 어긋나 있었다. **시드는 dev 편의용이고 진실원은 backend다** — 시드를 backend
방식(무염 SHA-256)으로 맞췄다(pipeline `bd344a9`).

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
| M1 | **좁혀짐(PROJ-80)** — backend `LocalSeeder`의 endpoint가 설정값(`pulsemetry.local-seed.otlp-endpoint`, 기본 `:4316` 정상 경로)으로 분리돼(backend `f9d3762`) 두 시드가 정렬됐다. 남은 것은 로컬 compose가 두 시드에 **한 값을 주입**하는 배선(파이프라인 `seed.sql`의 주입형 전환 포함)이다 | (기존 위험이던 `:4318` 하드코딩 — auth-proxy 우회·자기참조 — 은 제거됨) |
| M7 | **계약 진화 취약** — 클라이언트 `DisallowUnknownFields` + 서버 `FAIL_ON_UNKNOWN_PROPERTIES`. 응답 필드 하나만 추가해도 배포된 전 클라이언트가 파괴된다 | 버저닝 또는 tolerant reader 정책이 필요하다. **현재는 필드 추가가 breaking change다** |
| M8 | enrollment HTTP 클라이언트에 **타임아웃이 없고** 3xx 리다이렉트를 따라가며 초대 코드를 재전송한다 | 무한 대기, 코드 유출 |
| M9 | manifest `protocol: "grpc"`는 서버 검증을 통과하지만 클라이언트가 상위 전송을 지원하지 않아 로컬 파이프라인 배선에서 제외된다(회사 직결 강등 — [`telemetry-ingest.md`](telemetry-ingest.md) §6). 현재 grpc 테넌트는 없다 | 서버에서 grpc를 막거나 클라이언트에 구현해야 한다 |
| M10 | heartbeat·config 재조회·토큰 회전 부재. `installations.last_seen_at`·`telemetry_tokens.last_used_at`은 죽은 컬럼이고, `config_revision`은 저장만 하고 비교하지 않는다 | **manifest 변경이 기존 설치에 전파될 경로가 없다** ([`../product/prd.md`](../product/prd.md) §8-1) |
| — | `/v1/enroll`에 rate limit이 없다 | 60비트 초대 코드의 유일한 브루트포스 표면이 무방비 |
| — | `POST /v1/invitations/{id}/revoke`에 테넌트 격리가 없다 | 정적 admin 키 보유자가 전 테넌트 revoke 가능 |
| — | `--force` 플래그가 받기만 하고 아무 동작도 하지 않는다(엔드포인트 충돌 감지 미구현) | 사용자 기대와 불일치 |
| — | **해소됨(PROJ-80)** — `privacy.collect_raw_api_bodies`를 `required`에 추가해 Go 구조체와 대칭이 됐다(telemetryctl `23ddca3`. backend 계약 테스트가 갱신된 스키마 원본으로 통과) | — |
