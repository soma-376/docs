# 0016. 애플리케이션 HTTP API는 /api/v1을 정식 경로로 사용한다

## Status

Accepted

## Context

백엔드의 조직·분석 API는 `/api/v1`이지만 인증·설치·초대·헬스 API는 `/v1`이다. 프론트엔드와 telemetryctl은 두 접두사를 모두 사용한다. ALB도 `/v1` 일부만 라우팅하므로 호출 주소를 예측하기 어렵다. 이 결정은 docs, pulsemetry-backend, pulsemetry-frontend, telemetryctl, infra에 걸린다.

## Decision

- Pulsemetry 애플리케이션 HTTP API의 정식 접두사는 `/api/v1`이다. 인증, 설치, 초대, manifest, heartbeat, inquiry, health를 포함한다.
- 기존 `/v1` 애플리케이션 경로는 호환 별칭 없이 제거한다. 호출자는 서버 배포와 함께 새 경로로 전환한다.
- OTLP/HTTP 표준 경로 `/v1/logs`, `/v1/metrics`, `/v1/traces`는 유지한다. 데몬의 로컬 루프백 API, 외부 벤더 API, 설치 스크립트·바이너리 URL, 프론트엔드의 동일 출처 `/api/bff`는 이 결정의 대상이 아니다.
- 인프라 라우팅과 헬스 체크는 새 애플리케이션 경로를 가리킨다. OTLP 규칙은 별도로 유지한다.

## Alternatives Considered

### 기존 경로를 호환 별칭으로 유지

- 장점: 배포 순서의 제약이 줄어든다.
- 단점: 두 URL이 계속 공개 계약으로 남아 경로 혼란과 테스트 부담이 지속된다.
- 탈락 이유: 기존 경로를 제거하라는 사용자 결정에 따른다.

## Consequences/Tradeoffs

### Positive

- 애플리케이션 API 호출 규칙과 ALB 라우팅 기준이 일치한다.

### Negative

- 구버전 프론트엔드·데몬과 구 헬스 체크는 서버 전환 후 실패한다. 배포 전 호출자·인프라를 함께 전환하고 롤백 시 같은 버전 묶음으로 되돌려야 한다.

## Follow-up

- infra의 ALB 경로 규칙과 대상 그룹 헬스 체크를 변경하고 서버·프론트엔드·데몬의 배포 순서를 정한다.

## Acceptance Criteria

- 애플리케이션 엔드포인트는 `/api/v1`에서 응답하고 과거 `/v1` 주소에서는 응답하지 않는다.
- OTLP 세 경로는 `/v1`에서 계속 동작한다.

## References

- [`../contracts/enrollment-api.md`](../contracts/enrollment-api.md)
- [`../contracts/dashboard-api.md`](../contracts/dashboard-api.md)
- [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md)
