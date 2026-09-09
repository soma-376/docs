# 0007. 사용자 로그인은 PKCE 코드 교환을 쓰고 가입과 설치의 초대 소비를 분리한다

## Status
Accepted — 사용자 승인 구현 계획. 상대 레포 리뷰와 계약 배포는 미완료다.

## Context
backend·telemetryctl·frontend가 사용자 인증 계약의 당사자다.
telemetryctl develop 659a909에는 사용자 로그인 구현이 없다. 기존 설치 코드는 enroll에서 한 번 소비된다.

## Decision
- 회원가입과 설치는 같은 초대 코드를 각각 한 번 소비한다. used_at은 설치 전용, signup_used_at은 가입 전용이다.
- 만료·폐기는 양쪽에 적용한다. 가입은 대상 이메일 일치와 password_hash IS NULL을 요구하며 role을 요청으로 받지 않는다.
- 사용자 로그인 식별자는 tenant_id와 정규화 이메일이다. signup은 토큰을 발급하지 않는다.
- 웹·CLI 진입점은 동일 발급 코어를 사용한다. CLI는 S256 PKCE와 60초 일회용 코드를 사용한다.
- 콜백은 http://127.0.0.1:{port}/callback 또는 http://[::1]:{port}/callback만 허용한다. userinfo·query·fragment를 받지 않는다.
- callback URL에는 code와 state만 담고 클라이언트는 state를 비교한다. 서버는 callback 주소에 네트워크 요청을 하지 않는다.
- AT는 iss·aud·sub(member UUID)·tenant_id·role·manifest_revision·sid·iat·exp·jti를 갖는다. 최초 revision은 발급 시 선택된 서버 정책이며 로컬 적용 증명이 아니다.
- 사용자 세션 봉투는 기존 enroll 4키와 별개 스키마다. 설치 pit_/ptt_의 의미와 응답은 유지한다.

## Alternatives Considered
- callback에 AT/RT 직접 전달: URL 기록에 장기 비밀이 노출되므로 배제한다.
- 초대 소비 하나 공유: 가입/설치 중 먼저 실행한 쪽이 다른 쪽을 막으므로 배제한다.

## Consequences/Tradeoffs
### Positive
기존 CLI 설치를 깨지 않고 가입을 도입한다. 코드 탈취만으로는 토큰을 교환할 수 없다.
### Negative
초대는 만료 전 설치와 가입의 두 권한을 가진다. 클라이언트와 웹 화면은 별도 구현이 필요하다.

## Follow-up
실제 telemetryctl 로그인·콜백·키링 저장과 frontend 로그인/가입 화면은 연결 티켓 초안으로 분리한다.

## References
- https://www.rfc-editor.org/rfc/rfc8252
- https://www.rfc-editor.org/rfc/rfc7636
