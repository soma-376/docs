# 0008. manifest 재동기화는 서버 원자성만 보장하고 OTLP는 설치 토큰을 유지한다

## Status
Accepted — 사용자 승인 계획. 계약 스키마 게시·상대 레포 리뷰·파급 티켓 등록은 미완료다.

## Context
backend·telemetryctl·infra에 영향을 주는 결정이다. backend ADR 0007은 한 응답의 새 토큰을
로컬 manifest 적용 완료의 증거로 설명했으나 서버 응답과 클라이언트 키링/설정 쓰기는 다른 트랜잭션이다.
PROJ-102와 최신 telemetryctl은 설치 귀속 ptt_로 OTLP를 인증한다.

## Decision
- GET /v1/manifest의 Authorization Bearer는 사용자 RT다. AT/pit_/ptt_는 받지 않는다.
- 세션을 잠그고 구성원·tenant 상태를 확인한다. 활성 manifest 행을 공유 잠금으로 읽고 스키마를 검증한다.
- 기존 RT 소비·세션 revision 변경·새 RT 저장·AT 서명을 한 서버 트랜잭션으로 수행한다.
- 응답의 manifest.config_revision과 AT.manifest_revision은 같은 DB version이다. 실패는 전부 롤백한다.
- 원자성은 서버까지만이다. 클라이언트 적용 완료나 OTLP 정책 최신성을 보장하지 않는다.
- 일반 refresh는 revision을 올리지 않는다. 사용자 AT의 정책 불일치 코어는 409 manifest_revision_mismatch다.
- 상태를 바꾸는 GET을 기존 티켓 계약대로 유지하되 no-store/Pragma no-cache를 사용하고 304·캐시·자동 재시도를 금지한다.
- 커밋 뒤 응답 유실은 롤백할 수 없다. 사용한 RT 재시도는 세션을 폐기하므로 재로그인한다.
- OTLP는 installation 귀속 ptt_를 유지한다. PROJ-102 검증기와 설치별 폐기·401/403 복구를 유지하고 AT/revision 검사는 추가하지 않는다.

## Alternatives Considered
- 사용자 AT로 OTLP 전환: 설치 격리·재시도 의미를 바꾸므로 채택하지 않는다.
- 클라이언트 원자 적용까지 보장: 적용 확인 프로토콜과 로컬 복구 구현이 별도로 필요해 이번 서버 보장에 포함하지 않는다.
- POST 재동기화: 의미상 자연스럽지만 이번 GET 계약을 변경하지 않는다. 일반 목적 프록시/클라이언트는 캐시 금지 조건을 따라야 한다.

## Consequences/Tradeoffs
### Positive
기존 OTLP 인증 경로를 유지하면서 사용자 정책과 세션의 서버 일관성을 검증할 수 있다.
### Negative
서버는 로컬 정책 적용을 알지 못한다. 응답 유실과 정상 중복 RT 소비도 재로그인을 요구한다.
공유 잠금은 정책 활성화 트랜잭션을 짧게 지연시킨다.

## Follow-up
telemetryctl 설정 적용·키링 복구와 PROJ-109 관리자 인가 연결을 외부 티켓으로 분리한다.
`contracts/user-auth-followups.md`는 미등록 티켓 초안이다.
