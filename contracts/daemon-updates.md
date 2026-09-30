# 계약 — 데몬 업데이트 확인

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (데몬 `internal/updatecheck`, 빌드·릴리스 산출물) ↔ **`pulsemetry-backend`** (`apps/enrollment-api`) |
| 기계 판독 원본 | `telemetryctl/contracts/daemon-updates.schema.json` (`check_updates_response`, `release_metadata`) |
| 관련 ADR | 허브 [ADR 0011](../adr/0011-daemon-update-check-uses-served-binary-version.md) |
| 상태 | 변경 중 (PROJ-187) |

데몬이 등록한 서버에 "내 버전보다 새 데몬이 있는가"를 묻고 서버가 답하는 계약이다.
알려 주기만 한다. 내려받기와 설치는 이 계약에 없다.

## 1. 요청

`GET {server_url}/api/v1/check-updates?current_version=…&platform=…&architecture=…`

| 항목 | 값 |
|---|---|
| `{server_url}` | 등록할 때 쓴 서버 주소. base path가 있으면 그대로 앞에 붙는다 |
| 인증 | 없음. 데몬은 `Authorization`을 보내지 않고 서버는 요구하지 않는다 |
| `current_version` | 실행 중인 데몬의 버전. §4의 형식 |
| `platform` | 데몬의 `runtime.GOOS` — `darwin` · `linux` · `windows` |
| `architecture` | 데몬의 `runtime.GOARCH` — `amd64` · `arm64` |
| 헤더 | `Accept: application/json` |

세 쿼리는 모두 필수다. 서버는 그 밖의 쿼리를 무시한다.

## 2. 응답

**200** — JSON 문서 하나. 뒤에 다른 내용이 붙지 않는다.

```json
{ "latest_version": "0.2.0", "update_available": true }
```

| 키 | 타입 | 뜻 |
|---|---|---|
| `latest_version` | 비어 있지 않은 문자열, §4의 형식 | 이 서버가 그 플랫폼·아키텍처로 배포하는 데몬의 버전(§3) |
| `update_available` | 불리언 | `current_version`이 `latest_version`보다 **낮을 때만** true. 판정은 서버가 한다 |

- 헤더는 `Content-Type: application/json`과 `Cache-Control: no-store`다.
- 서버는 이 두 키만 보낸다. 데몬은 모르는 키를 무시한다.
- 서버는 **리다이렉트를 내지 않는다.** 데몬은 3xx를 따라가지 않고 실패로 처리한다.
- 본문은 64 KiB를 넘지 않는다. 넘으면 데몬이 무효로 처리한다.

| 상태 | 뜻 | 데몬의 표시 |
|---|---|---|
| 200 | 위 본문 | 최신 버전과 업데이트 여부 |
| 400 `invalid_request` | 세 쿼리 중 빠지거나 빈 것이 있다, `current_version`이 §4의 형식이 아니다 | 확인 실패 |
| 404 | 서버가 그 대상에 대해 답할 수 없다(§3) | 미지원 |
| 503 + `Retry-After` | 일시 장애 | 확인 실패 |

오류 본문은 `{"error": "…", "message": "…"}`다. 데몬은 상태 코드만 본다.
데몬은 실패해도 곧바로 다시 묻지 않는다. 다음 주기에 다시 묻고, 그동안 마지막으로 성공한 결과를 확인 시각과 함께 유지한다.
**실패와 미지원을 "최신 버전"으로 표시하지 않는다.**

## 3. 최신 버전 — 서버가 배포하는 바이너리의 판

최신 버전은 그 서버가 `GET /bin/{filename}`([`enrollment-api.md`](enrollment-api.md) §1)으로 배포하는 바이너리의 버전이다.
서버는 외부 릴리스 목록을 보지 않는다.

버전은 **릴리스 메타데이터 파일**이 말한다. 바이너리를 만드는 쪽이 여섯 바이너리와 같은 디렉터리에 `pulsemetry_release.json`을 둔다.

```json
{
  "version": "0.2.0",
  "sha256": {
    "pulsemetry_darwin_amd64": "<소문자 hex 64자>",
    "pulsemetry_darwin_arm64": "<소문자 hex 64자>",
    "pulsemetry_linux_amd64": "<소문자 hex 64자>",
    "pulsemetry_linux_arm64": "<소문자 hex 64자>",
    "pulsemetry_windows_amd64.exe": "<소문자 hex 64자>",
    "pulsemetry_windows_arm64.exe": "<소문자 hex 64자>"
  }
}
```

- `version`은 그 바이너리들에 주입한 버전과 같다(§4의 형식).
- `sha256`의 키는 바이너리 파일명이다. 만들지 않은 대상은 키를 뺀다. 여섯 이름 밖의 키는 쓰지 않는다.
- 이 파일은 `/bin/{filename}`으로 서빙하지 않는다.
- 서버는 모르는 키를 무시한다.

서버는 요청의 `platform`·`architecture`로 파일명(`pulsemetry_{platform}_{architecture}`, Windows만 `.exe`)을 정하고,
**그 파일이 있고 SHA-256이 메타데이터와 같을 때만** `version`을 `latest_version`으로 답한다.

다음은 모두 **404**다. 서버가 주지 못하는 버전을 최신이라고 답하지 않는다.

- 업데이트 확인이 꺼져 있다.
- 메타데이터 파일이 없거나 형식이 틀렸다.
- `platform`·`architecture`가 여섯 파일명 중 하나로 이어지지 않는다(§1의 값이 아니다).
- 그 대상의 바이너리가 없거나 메타데이터에 그 파일명이 없다.
- 바이너리의 해시가 메타데이터와 다르다.

## 4. 버전 표기와 비교

- 형식은 [SemVer 2.0.0](https://semver.org/spec/v2.0.0.html)이다: `X.Y.Z`, 사전 릴리스는 `X.Y.Z-rc.1`. **`v`를 붙이지 않는다.**
- 순서는 SemVer의 우선순위 규칙이다.
  - 주·부·수 번호를 숫자로 비교한다.
  - 사전 릴리스는 같은 번호의 정식 판보다 낮다(`0.2.0-rc.1` < `0.2.0`). 사전 릴리스끼리는 점으로 나눈 식별자를 차례로 비교한다.
  - 빌드 메타데이터(`+…`)는 순서에 영향을 주지 않는다.
- `update_available`은 `current_version` < `latest_version`일 때만 true다. 같으면 false, 데몬이 더 높아도 false다.
- 버전을 주입하지 않은 빌드는 `0.1.0`으로 보고한다. 서버는 이를 다른 버전과 똑같이 비교한다.
- 형식이 틀린 `current_version`(예: `v0.2.0`, `dev`, 빈 값)은 400이다. 순서를 알 수 없는 버전에 "업데이트 없음"이라고 답하지 않는다.

## 5. 데몬의 동작

- 기동 직후와 24시간마다 묻는다. 서버 주소가 없으면 묻지 않는다.
- 요청 한 번의 제한은 10초다. 종료할 때는 진행 중인 요청을 취소한다.
- 서버의 `update_available`을 그대로 쓴다. 버전을 다시 비교하지 않는다.
- 결과는 메모리에만 둔다. 로컬 API와 CLI `status`가 같은 결과를 읽는다.

## 6. 미해결

| 항목 | 영향 |
|---|---|
| 릴리스 워크플로가 `pulsemetry_release.json`을 만들지 않는다 | 메타데이터를 따로 만들어 놓기 전까지 서버는 404로 답한다 |
| 릴리스 태그 검사(`vX.Y.Z[-…]`)가 SemVer보다 느슨하다 | SemVer가 아닌 태그로 만든 바이너리는 업데이트 확인 대상이 되지 못한다 |
| 내려받기·설치·서명 검증이 없다 | 사용자가 직접 다시 설치해야 한다 |
| 배포 채널(정식·시험판) 구분이 없다 | 서버에 놓인 판이 곧 그 서버의 최신이다 |
