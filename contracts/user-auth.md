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

backend CI는 telemetryctl 계약 커밋 e730bbfa08657a9887a87cc1d1e07095e9cca305를 고정한다.
해당 커밋이 현재 로컬에만 있으므로 계약 브랜치를 먼저 게시해야 원격 CI가 이를 읽을 수 있다.
실제 클라이언트 변경은 이 스키마를 구현하는 후속 티켓으로 분리한다.
