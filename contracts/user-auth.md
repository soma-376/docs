# 계약 — 사용자 인증

당사자: backend ↔ telemetryctl·frontend. 상태: 변경 중, PROJ-107 로컬 구현·상대 레포 리뷰 대기.
기계 판독 원본은 telemetryctl/contracts/user-auth.schema.json이다. 기존 enroll 봉투와 독립이다.

## API
- POST /v1/auth/signup: code, email, password → 201 (빈 본문).
- POST /v1/auth/login: tenant_id, email, password → 200 사용자 토큰 봉투.
- POST /v1/auth/cli/authorize: 로그인 필드 + redirect_uri, state, code_challenge, code_challenge_method=S256 → 200 {callback_url}.
- POST /v1/auth/cli/token: code, redirect_uri, code_verifier → 200 사용자 토큰 봉투.
- POST /v1/auth/refresh: refresh_token → 200 사용자 토큰 봉투.
- POST /v1/auth/logout: refresh_token → 204.

봉투: access_token, refresh_token, token_type=Bearer, expires_in=300.
RT는 urt_ 접두사와 32바이트 base64url 난수다. CLI code는 uac_ 접두사를 쓴다.
인증 결과와 무관하게 Cache-Control: no-store를 사용한다. 토큰·코드는 URL/로그에 넣지 않는다(콜백의 일회용 code만 예외).
웹은 JSON API만 제공한다. 쿠키 세션을 만들지 않는다. 허용 Origin은 서버 설정 allowlist로 한정한다.

## 가입과 회전
같은 초대의 가입·설치 소비는 독립이다. 이메일은 trim+소문자로 정규화한다.
가입은 12 코드포인트 이상·UTF-8 72바이트 이하 비밀번호를 받는다. 비밀번호 재설정은 지원하지 않는다.
일반 refresh는 manifest revision을 변경하지 않는다. RT를 직렬 소비해야 하며 재사용은 세션 전체를 폐기한다.
세션 절대 만료는 최초 발급 후 30일이다. 응답 유실로 RT를 잃으면 재로그인한다.

## 오류
400 invalid_request, 401 invalid_credentials, 409 signup_unavailable 또는 manifest_not_configured,
429 rate_limited + Retry-After, 503 auth_unavailable + Retry-After.
JWT 실패는 invalid_credentials로 통일한다. DB 장애나 서명 실패를 인증 실패로 숨기지 않는다.

## 현재 상태
제품 웹 화면·CLI 로그인·키링 저장은 미구현이다. 서버와 모의 CLI E2E만 이 티켓의 대상이다.
관리자 role 인가 전환은 PROJ-109이며 owner/admin 대시보드 제한은 그 티켓이 담당한다.

## 검증·게시 순서

backend CI는 telemetryctl 계약 커밋 b190344d24a7d3e6f22241a0e2f0ef06390d72cf를 고정한다.
해당 커밋이 현재 로컬에만 있으므로 계약 브랜치를 먼저 게시해야 원격 CI가 이를 읽을 수 있다.
실제 클라이언트 변경은 이 스키마를 구현하는 후속 티켓으로 분리한다.

## manifest 재동기화 (PROJ-108, 변경 중)

GET /v1/manifest는 Authorization: Bearer <사용자 RT>로 호출한다. AT·pit_·ptt_는 거부한다.
응답은 manifest와 사용자 토큰 봉투 4키를 합친 5키다. 원본은 telemetryctl/contracts/manifest-resync.schema.json이다.
JWT 클레임은 user-auth.schema.json의 access_token_claims를 따른다.

세션 잠금, 구성원/조직 검사, 활성 manifest 읽기·검증, RT 소비, 세션 revision 변경과 새 AT/RT 저장이
한 서버 트랜잭션이다. 응답 manifest.config_revision과 JWT.manifest_revision은 같은 DB version이다.
manifest 부재/계약 위반은 409 manifest_not_configured다. 실패 시 소비 전 RT를 재사용할 수 있다.

GET이 상태를 변경한다. Cache-Control: no-store와 Pragma: no-cache를 사용하고 304를 반환하지 않는다.
프리페치·자동 재시도·캐시를 금지한다. 커밋 후 응답이 유실되면 재로그인한다.
같은 RT로 동시 refresh/resync하면 재사용 탐지가 세션 전체를 폐기한다.

클라이언트에 실제로 정책이 적용됐는지는 증명하지 않는다. 서버 원자성은 로컬 키링/설정 쓰기와 별개다.
일반 refresh는 기존 revision을 유지한다. 관리자 경로의 낡은 AT는 PROJ-109가 검증 코어를 연결한 뒤
409 manifest_revision_mismatch로 재동기화를 요청한다. OTLP ptt_ 경로에는 적용하지 않는다.

재동기화 서버는 build에서 원본 manifest JSON Schema를 jar에 포함하여 원문을 검증한다. DTO 기본값으로 필수 정책 누락을 보충하지 않는다.

가입·설치 소비 분리의 폐기 규칙: 한쪽 권한이라도 미소비이면 초대를 폐기할 수 있다(204).
양쪽 모두 소비한 초대는 409 invitation_used, 이미 폐기된 초대는 409 invitation_revoked다.
초대 폐기는 남은 권한만 끊으며 이미 만든 계정·설치 자격증명을 폐기하지 않는다.
