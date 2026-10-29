# 계약 — enrollment API

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (Go 클라이언트) ↔ **`pulsemetry-backend`** (`apps/enrollment-api`) |
| 기계 판독 원본 | `telemetryctl/contracts/enrollment-manifest.schema.json`, `enrollment-envelope.schema.json` |
| 서버 측 상세 | `pulsemetry-backend/docs/api/enrollment.md` |
| 관련 ADR | backend ADR-0003(계약과 2단 토큰), 0005(부트스트랩·바이너리 서빙), 0007(인증 계층), 0009(스키마 enum), [허브 ADR 0025](../adr/0025-invitation-installation-delivery.md) |
| 상태 | 확정 — 초대 설치 동선과 사전 검증은 ADR 0025를 따른다 |

관리자가 발급한 일회성 초대 코드를 검증·소비해 사용자 PC의 설치(installation)를 만들고,
그 설치에 귀속되는 자격증명과 회사 단위 OTel 설정(manifest)을 내려주는 계약이다.

## 1. 엔드포인트

| 메서드 | 경로 | 인증 | 성공 | 호출자 |
|---|---|---|---|---|
| POST | `/api/v1/enroll` | 없음 (초대 코드 자체가 자격) | 201 | telemetryctl |
| POST | `/api/v1/installations/telemetry-token` | `Authorization: Bearer <pit_…>` | 200 | telemetryctl (데몬) |
| POST | `/api/v1/invitations` | `X-Admin-Token` | 201 | 관리자 |
| POST | `/api/v1/invitations/{id}/revoke` | `X-Admin-Token` | 204 | 관리자 |
| POST | `/api/v1/organizations/{organizationId}/invitations/batch` | 사용자 access token, owner/admin | 200 | 관리자 웹 UI |
| GET | `/windows?code=…` / `/unix?code=…` | 없음 | 200 `text/plain` | 사용자 셸 |
| GET | `/bin/{filename}` | 없음 | 200 `application/octet-stream` | 부트스트랩 스크립트 |
| GET | `/api/v1/healthz` | 없음 | 200 | 운영 |

스크립트·바이너리 경로에 `/api/v1` 접두사가 없는 것은 의도다 — 사용자가 터미널에 붙여넣는 URL이라 짧아야 한다.

### 공개 설치 안내와 메일

`pulsemetry-frontend`의 `/install?code=…`는 인증 없이 설치 명령을 제공하는 페이지다.
backend의 `pulsemetry.management.installation-page-url`은 dev에서
`https://dev.pulsemetry.co.kr/install`이며, 관리·메일 활성화 시 유효한 URL이 필수다.
설치 가능한 일반 초대·설치 전용 초대·재발급 메일에는 정규화한 코드의 설치 링크와 기존 원시
설치 명령을 함께 넣는다. 설치 권한을 소진한 초대에는 둘 다 넣지 않는다.
`invitation-accept-url`은 코드 없는 SSO 로그인 주소로 유지한다.

메일은 암호화 outbox로 보내며 초대별 중복 방지·미발송 취소·재시도 계약을 유지한다.
정상 메일 발급 동선은 사용자 인증을 거친 조직의 일괄 초대·재발급·설치 전용 초대 API다.
정적 관리자 키의 `POST /api/v1/invitations`는 메일을 적재하지 않으므로 실제 발송 검증을
그 엔드포인트로 대체하지 않는다.
재발급은 새 코드와 링크를 사용한다. 동일 멱등 요청은 초대·메일을 중복 생성하지 않는다.
query 코드 전달은 설치 페이지와 부트스트랩에 한정하고 제목·일반 로그·오류에 코드를 남기지
않는다. 페이지는 `no-store`·`no-referrer`를 사용하며 페이지 열기로 코드를 소비하지 않는다.
프론트의 `NEXT_PUBLIC_ENROLLMENT_API_URL`은 backend 공개 API 주소와 일치해야 한다.

### 부트스트랩 코드 사전 검증

`/unix`·`/windows`는 `InvitationCode` 규칙으로 정규화·형식 검사한 뒤 SHA-256 해시로
초대 상태를 읽는다. 유효한 코드만 다운로드 스크립트를 받는다. 누락·형식 오류·미존재·만료·
사용·취소는 사유별 고정 안내를 stderr에 출력하고 실패 종료하는 스크립트를 HTTP 200
`text/plain`으로 반환한다. 오류 스크립트는 바이너리 요청·파일 생성·enroll을 실행하지 않으며
입력 코드를 재출력하지 않는다. 모든 스크립트는 `Cache-Control: no-store`를 사용한다.
DB 장애는 서버 오류로 처리하고 코드 무효나 정상 다운로드로 바꾸지 않는다.

사전 조회는 `used_at`, 설치, 구성원 상태를 바꾸지 않는다. 최종 enroll은 반드시 상태를 다시
검사하고 조건부 UPDATE로 소비한다. 조회 후 취소·만료·다른 요청의 소비는 최종 enroll을
거부한다. 사전 조회 결과는 예약이나 최종 허가가 아니다.

부트스트랩 URI는 한 번 URL decode한 뒤 `/unix`·`/windows`와 정확히 일치시키며,
`/api/v1/enroll`을 포함한 기존 등록 API는 raw URI의 정확 일치를 유지한다. 이 경로들은
출발지 IP별 합산 300회/300초 WAF 예산을 공유한다. 차단은 HTTP 429와 `Retry-After: 60`이며
전달 헤더를 집계 키로 신뢰하지 않는다. dev WAF 정책 배포·검증 후 사전 조회 기능을 공개한다.
HTTP 오류에서 셸 파이프 전체의 종료 코드 보장은 이 계약에 포함하지 않는다.

### 바이너리 산출물 이름 규칙

공개 `GET /bin/{filename}`과 원격 릴리스 파일 이름은 다음처럼 매핑한다.

| 공개 경로 파일명 | GitHub Releases CLI 자산 |
|---|---|
| `pulsemetry_windows_amd64.exe` | `pulsemetry_cli_windows_amd64.exe` |
| `pulsemetry_windows_arm64.exe` | `pulsemetry_cli_windows_arm64.exe` |
| `pulsemetry_darwin_amd64` | `pulsemetry_cli_darwin_amd64` |
| `pulsemetry_darwin_arm64` | `pulsemetry_cli_darwin_arm64` |
| `pulsemetry_linux_amd64` | `pulsemetry_cli_linux_amd64` |
| `pulsemetry_linux_arm64` | `pulsemetry_cli_linux_arm64` |

공급 태그는 **`v0.1.0-rc.3`**로 고정한다. prerelease를 명시적으로 선택하며 `latest`나 로컬
재빌드로 대체하지 않는다. 원격 CLI 6종·GUI 6종과 `SHA256SUMS`를 받아 필수 파일 존재·목록 포함·실제
SHA-256 일치를 모두 검증한다. GUI도 같은 고정 해시 검증을 거쳐 공급한다. 조회 실패·파일
누락·해시 불일치는 이미지 빌드를 실패시킨다.

검증한 게시 파일은 enrollment 이미지의 `/app/binaries/v0.1.0-rc.2/`에 실행 사용자가 읽을 수
있도록 포함하고 `PULSEMETRY_BINARIES_DIR=/app/binaries`로 연결한다. 기존 `BinaryController`와
`DaemonRelease`의 versioned 매핑·해시 검증·업데이트 인식 동작을 사용한다. 릴리스 변경 시 태그,
필수 자산 해시, 배포 이미지 digest를 기록한다. 실제 dev 공개 6종의 해시가 게시 자산과 같아야
하며 이전 이미지 digest와 설정을 지정해 복구한다. 이미지 게시와 실제 EKS 배포·검증은 별도다.

GUI는 `pulsemetry_gui_windows_{amd64|arm64}.exe`, `pulsemetry_gui_darwin_{amd64|arm64}.dmg`,
`pulsemetry_gui_linux_{amd64|arm64}.AppImage`다. 다운로드 표시 이름은 각각 `Pulsemetry.exe`,
`Pulsemetry.dmg`, `Pulsemetry.AppImage`다. 릴리스 자산은 같은 이름과 `SHA256SUMS`로 검증한다.
릴리스 디렉터리가 없는 개발 환경은 허용 이름 그대로의 평면 파일을 제공한다.

부트스트랩은 등록 성공 후 `pulsemetry gui install`을 실행한다. GUI 설치·실행 실패는
등록 성공을 취소하지 않는다. 최초 실행은 데몬 준비 확인 뒤에만 한다([ADR 0027](../adr/0027-install-desktop-and-report-policy-status.md)).

### 설치 보고와 GUI 정책 상태

기존 heartbeat 응답의 `expected_config_revision`을 적용 판과 비교한다. GUI는 데몬의 로컬 API에서
적용 판·최신 확인 판·확인 시각·확인 실패를 읽는다. 이 조회는 정책 자동 적용을 하지 않는다.
응답의 판이 null이거나 조회 실패·결과 만료인 경우 최신 정책 적용 완료로 표시하지 않는다.


## 2. `POST /api/v1/enroll`

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

소비 권한을 `SELECT` 결과만으로 결정한 뒤 무조건 `UPDATE`하지 않는다. 사전 조회와 별개로 위 조건부 UPDATE가 최종 소비를 결정한다. 동시 요청 N개가 같은 코드로 들어와도 **정확히 하나만 201**을 받고
나머지는 409 `invitation_used`다.

**enroll 성공은 대상 멤버의 `invited → active` 전환 이벤트다.** OTLP 경로의 auth-proxy가
`invited` 멤버의 토큰을 거부하므로, 이 전환 없이는 발급된 telemetry token이 전부 401이 된다
([`telemetry-ingest.md`](telemetry-ingest.md) §3). 재발급(`POST /api/v1/installations/telemetry-token`)도
같은 전환을 보정한다 — `pit_` 인증이 과거 enroll 완료의 증명이기 때문이다.
전환은 `invited`에서만 일어난다. **`suspended`는 어느 경로도 건드리지 않는다** — 정지 해제는 관리자의 결정이지 설치의 부수효과가 아니다.

## 3. 2단 토큰 모델

| 토큰 | 접두사 | 저장 위치 | 용도 | 교체 |
|---|---|---|---|---|
| `installation_token` | `pit_` | OS 키링 | 이 설치의 장기 신원 | 하지 않는다 |
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

PROJ-105에서 **dev 시드 자체가 backend로 모였다.** 이제 `apps/enrollment-api`의 `LocalSeeder`
(`local` 프로파일)가 tenant·member·팀·소속·manifest·초대를 넣는다. 시드가 두 벌이면 어느 쪽이
사실인지가 실행 순서로 정해지고, SQL 시드는 backend Flyway보다 먼저 돌아 스키마가 없는 시점에
적용된다. 해시 방식이 갈릴 여지도 함께 사라졌다 — 시더가 서버와 같은 `Sha256.hex`를 부른다.

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
| ~~M1~~ | **해소됨(PROJ-105)** — 시드가 backend `LocalSeeder` 한 벌로 모였다. 주입할 두 번째 시드가 없으므로 값을 맞출 배선도 필요 없다. endpoint 기본값 `:4316`은 이제 `:apps:telemetry-ingest`가 로컬에서 듣는 포트이며, 구 auth-proxy의 자리를 물려받은 것이라 데몬 설정을 바꾸지 않는다 | (기존 위험이던 `:4318` 하드코딩 — auth-proxy 우회·자기참조 — 은 제거됨) |
| M7 | **계약 진화 취약** — 클라이언트 `DisallowUnknownFields` + 서버 `FAIL_ON_UNKNOWN_PROPERTIES`. 응답 필드 하나만 추가해도 배포된 전 클라이언트가 파괴된다 | 버저닝 또는 tolerant reader 정책이 필요하다. **현재는 필드 추가가 breaking change다** |
| M8 | enrollment HTTP 클라이언트에 **타임아웃이 없고** 3xx 리다이렉트를 따라가며 초대 코드를 재전송한다 | 무한 대기, 코드 유출 |
| M9 | manifest `protocol: "grpc"`는 서버 검증을 통과하지만 클라이언트가 상위 전송을 지원하지 않아 로컬 파이프라인 배선에서 제외된다(회사 직결 강등 — [`telemetry-ingest.md`](telemetry-ingest.md) §6). 강등 상태에서는 포워더 `Scrub`이 경로 밖이라 **manifest `privacy` 집행에 공백이 생긴다**(같은 문서 §5 M13). 현재 grpc 테넌트는 없다 | 서버에서 grpc를 막거나 클라이언트에 구현해야 한다 |
| M10 | heartbeat는 시작 직후·매분 보고하며 GUI가 서버 최신 판과 실제 적용 판을 비교한다. manifest 재조회·정책 자동 적용·토큰 자동 회전은 미구현이다 | 정책 업데이트 대기는 표시하며 적용은 재설치가 필요하다 |
| — | **해소됨(PROJ-269·268)** — WAF 등록 그룹 확대와 사전 조회를 dev에 배포하고 실제 차단·복구, 조회 비소비, 상태 변경 후 거부와 동시 단일 성공을 검증했다 | [ADR 0025 수용 결과](../adr/0025-invitation-installation-delivery.md#수용-결과와-범위) 참조. 기존 제한 M7~M10은 별도다 |
| — | `POST /api/v1/invitations/{id}/revoke`에 테넌트 격리가 없다 | 정적 admin 키 보유자가 전 테넌트 revoke 가능 |
| — | `--force` 플래그가 받기만 하고 아무 동작도 하지 않는다(엔드포인트 충돌 감지 미구현) | 사용자 기대와 불일치 |
| — | **해소됨(PROJ-79)** — `privacy.collect_raw_api_bodies`를 `required`에 추가해 Go 구조체와 대칭이 됐다(telemetryctl `8268a3a`. backend 계약 테스트가 갱신된 스키마 원본으로 통과). **기존 저장 manifest 주의** — `required` 추가라 이 필드가 없는 기존 v1 manifest는 서버 검증에서 409 `manifest_not_configured`가 된다. 테넌트 온보딩 전 수동 INSERT 점검이 필요하다 | — |
