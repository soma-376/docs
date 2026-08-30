# 0004. 텔레메트리 파이프라인을 backend 레포로 병합한다

## Status

Accepted — [ADR 0003](0003-telemetry-pipeline-repo-boundary.md)(레포 2개 체제 유지, collector만 이관)을
대체한다. 0003은 `Superseded by`로 닫는다. backend ADR-0006(병합 원안)의 링크 stub은 이 ADR을
결정 원문으로 가리킨다.

## Context

걸리는 레포는 셋이다 — `ai-telemetry-pipeline`(병합 대상), `pulsemetry-backend`(도착지),
`infra`(레포 2개를 전제로 선 배포 계약: ADR-0017·0018·0022·0023). `telemetryctl`은 코드가 바뀌지
않지만 [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md)의 전송 계약이 보존
조건으로 걸린다.

ADR 0003은 두 근거로 전체 병합을 기각했다. 그중 하나가 무너졌다.

**"스키마 두 벌 문제는 진실원 지정으로 이미 풀렸다."** 진실원은 지정됐지만(backend Flyway,
backend ADR-0009) 규율은 유지되지 않았다. 세 지점에서 확인된다 — 공유 RDS 배선 단절
([`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §5 B3)은 E2E를 막은 채
미해결이고, 같은 계약 §4는 collector 설정 두 벌의 신원 전파 3요소를 "함께 바꾼다"고 규정하면서
자동 검증 장치를 두지 않기로 했으며 그 항목은 이미 한 번 어긋난 이력이 있다(§5 B4, 해소됨).
파이프라인의 dev DDL은 backend Flyway와 손으로 맞춘다. 0003은 이 구성을 "자동화 장치 없이 진실원
명시와 규율로 관리한다"고 적으면서 재검토 조건을 드리프트 재발로 걸어 두었다. 재발을 기다리는
대신, 규율이 유지되지 않는 현재 상태를 근거로 재검토한다.

**"재작성을 막아줄 테스트가 없다."** 이 근거는 여전히 유효하다. 반박하지 않고 비용으로 지불한다 —
특성화 테스트와 언어 중립 golden fixture 확보를 이식 착수의 선행 조건으로 둔다.

0003이 남긴 재개 트리거는 Python·Kotlin 성능 비교였다. 이번 재론의 근거는 성능이 아니라 진실원 규율이다.

결정하지 않으면 병합 작업은 유효한 ADR을 조용히 어기는 코드가 되고, 두 벌 구성의 드리프트는
감지 장치 없이 계속된다.

## Decision

- **파이프라인 로직을 `pulsemetry-backend`로 병합한다.** 이관이 끝나면 `ai-telemetry-pipeline`은
  아카이브한다. RDB(enrollment) 스키마의 진실원은 backend Flyway 그대로다.
- **배포 단위는 하나다.** 그 안의 계층 배치와 OTel Collector 존치 여부는
  [ADR 0005](0005-single-app-telemetry-topology.md)가 정한다.
- **OTLP 인증은 Spring Security가 맡는다.** [ADR 0001](0001-otlp-authentication-model.md)이 정한
  종착지에 도달하는 것이며, 불투명 토큰을 해시 조회로 검증하는 방식은 바뀌지 않는다.
  auth-proxy는 별도 배포 단위로 남지 않는다.
- **서버 마스킹을 생략하지 않는다.** 데몬의 1차 제거를 신뢰해 서버에서 건너뛰지 않는다.
  구현 위치는 ADR 0005가 정한다.
- **파이프라인 단계는 `:libs:` 모듈로 나누고 앱은 조립만 한다.** 경계 기준은 backend ADR-0008의
  쓰기 소유권·패키지 단일 공급·의존 방향 규칙이며, 모듈 목록은 backend `docs/module-map.md`가 소유한다.
- **ClickHouse 스키마 소유권을 backend로 옮긴다.** 0003이 파이프라인에 남겼던 것을 뒤집는다.

## Alternatives Considered

- **현행 유지(ADR 0003 — collector만 이관)** — 두 벌 구성과 B3가 그대로 남는다. 0003이 감수한
  "자동화 없이 규율로 관리한다"가 실제로 지켜지지 않았으므로 같은 선택을 한 번 더 할 근거가 없다. 기각.
- **레포는 병합하되 Python 구현을 유지** — 재작성 비용은 사라지지만 영속성 코드가 두 언어로 갈라진 채
  한 레포에 들어올 뿐이라 정의가 갈라지는 문제가 그대로 남는다. backend ADR-0006 원안이 같은 이유로
  Kotlin 재작성을 택했다. 기각.

## Consequences/Tradeoffs

### Positive

- 스키마·collector 설정·토큰 검증이 한 레포·한 언어로 모여, 드리프트를 규율이 아니라 빌드와 CI로 막을 수 있다.
- 토큰 발급자와 검증자가 `TOKEN_HASH_SECRET`을 나눠 갖는 구성이 사라진다
  ([ADR 0001](0001-otlp-authentication-model.md) Constraints).
- auth-proxy와 그 ECR 레포, infra ADR-0022의 잔여 항목이 함께 닫힌다.

### Negative

- 동작하는 Python 약 3,800라인을 재작성해야 한다. 특성화 테스트 확보가 선행 비용으로 남는다.
- 전환이 끝날 때까지 구 파이프라인과 새 앱이 병존하고, 문서는 두 시대를 함께 설명해야 한다.
- backend가 Postgres와 ClickHouse 두 저장소의 DDL을 관리한다. Flyway는 ClickHouse를 다루지 않으므로
  적용 경로를 따로 정해야 한다.
- 레포 경계를 전제로 확정했던 infra 결정들이 다시 열린다.

## Follow-up

- 특성화 테스트와 golden fixture 확보가 이식 착수의 전제다. 확보 전에는 이식을 시작하지 않는다.
- ClickHouse DDL의 적용 메커니즘을 backend에서 정한다. Flyway가 아니므로 진실원과 적용 경로를
  같은 결정으로 묶는다.
- infra ADR-0017 재검토, 0018 폐지(파생 DSN 시크릿이 불필요해진다), 0022의 4·8·10번 정리,
  0023 폐기, 새 배포 단위의 ECR 레포 신설.
- 계약 문서([`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) ·
  [`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §4)의 검증 주체 서술은 전환이
  실제로 끝난 시점에 고친다. 그전까지 현행 서술이 사실이다.

## References

- [ADR 0001](0001-otlp-authentication-model.md)(인증 종착지) ·
  [ADR 0003](0003-telemetry-pipeline-repo-boundary.md)(대체됨)
- backend ADR-0006(병합 원안) · ADR-0007(인증 계층) · ADR-0008(모듈 경계) · ADR-0009(Flyway 진실원)
- [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §4·§5 ·
  [`../architecture/repos.md`](../architecture/repos.md)
