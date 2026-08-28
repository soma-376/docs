# 0003. 텔레메트리 파이프라인은 별도 레포로 유지하고 collector만 backend로 이관한다

## Status

Accepted — backend ADR-0006(파이프라인 전체 병합)을 이 허브 결정으로 대체한다.
backend 0006은 `Superseded by`로 닫는다(제안 자체는 기각).

## Context

걸리는 레포는 셋이다 — `ai-telemetry-pipeline`(병합 대상), `pulsemetry-backend`(병합 도착지 제안),
`infra`(레포 2개 전제 위의 배포 계약: ADR-0009·0007·0015·0024).

backend ADR-0006은 "같은 개념의 스키마가 두 저장소에 두 벌"이라는 문제를 근거로 파이프라인 전체를
Kotlin/Spring으로 재작성해 backend로 병합하고 ClickHouse 스키마 소유권까지 가져오는 것을 제안했다.
단일 레포 ADR로 있었지만 레포의 존속이 걸린 결정이라 스코프 규칙상 허브 소유다.

제안의 근거 두 가지가 그 사이 바뀌었다. RDB 스키마 두 벌 문제는 병합 없이 진실원 지정(backend Flyway,
backend ADR-0009)으로 이미 풀렸다. 재작성의 완화책이던 기존 테스트(약 1,000라인)는 PROJ-40(`0893613`)
에서 삭제돼, 재작성은 특성화 테스트 신규 확보를 전제해야 하는 상태였다.

결정하지 않으면 `ai-telemetry-pipeline`은 자기가 병합 대상인지 모른 채 투자를 계속하고,
infra는 레포 2개 전제의 결정들을 확정하지 못한다.

## Decision

- **파이프라인 전체 병합은 기각한다.** 레포 2개 체제와 Python 구현을 유지한다.
- **collector만 backend로 이관한다**(backend ADR-0007). 도착지는 `:apps:telemetry-ingest`이며,
  collector config의 소유권(현행 infra `config/otel-collector.yaml` + 파이프라인 in-repo 설정)이
  함께 이동한다. 인증 계층은 배포 단위가 아니라 `:libs:security` 횡단 라이브러리로 간다.
- **RDB(enrollment) 스키마 진실원은 backend Flyway**(병합과 무관하게 backend ADR-0009로 확정),
  **ClickHouse 스키마 소유권은 파이프라인에 남는다.**
- 전체 이관은 Python·Kotlin 성능 비교 목적의 재논의 여지만 남긴다.

## Alternatives Considered

- **전체 병합 (backend ADR-0006 원안)** — 동작하는 Python 약 3,800라인의 재작성이 필요하고, 회귀를
  막아 줄 테스트가 없어 특성화 테스트 확보가 선행 비용으로 추가된다. 병합이 풀려던 스키마 중복은
  진실원 지정으로 이미 풀렸다. 기각.
- **현상 유지(collector도 이관하지 않음)** — OTLP 토큰 검증(auth-proxy)이 발급자(backend)와 다른
  레포·언어에 남아 인증 이관([ADR 0001](0001-otlp-authentication-model.md))이 성립하지 않는다. 기각.

## Consequences/Tradeoffs

### Positive

- 레포 경계가 확정되어 infra ADR-0009(단일 인프라 레포)·0024(배포 역할)의 전제가 안정된다.
- 인증 검증이 발급자와 같은 코드베이스로 수렴할 경로가 열린다.

### Negative

- 스키마·collector 설정의 "두 벌" 구성은 남는다 — 자동화 장치 없이 진실원 명시와 규율로 관리한다
  (드리프트 재발 시 감지 장치를 재검토한다).
- collector 이관 전까지 auth-proxy(폐기 예정)가 과도기 구성으로 유지된다.

## Follow-up

- collector 이관 시: 배포 단위 `:apps:telemetry-ingest`와 ECR 레포·`DEPLOY_TARGETS` 항목 추가,
  collector config 소유권 이동, infra ADR-0017 재검토(infra ADR-0009 Follow-up의 트리거).
- Python·Kotlin 성능 비교가 실제로 논의되면 이 ADR을 개정한다.

## References

- backend ADR-0006(`Superseded by` 이 ADR — 제안 기각, 본문 이관) · ADR-0007 · ADR-0009
- infra ADR-0009 · [`../architecture/repos.md`](../architecture/repos.md)
