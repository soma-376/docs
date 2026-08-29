# 0002. manifest 배정 단위는 tenant다

## Status

Accepted

## Context

걸리는 레포는 셋이다 — `pulsemetry-backend`(저장·배포), `telemetryctl`(계약 스키마·적용),
그리고 이 허브의 계약 문서([`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §5).

backend ADR-0007의 Context는 부서 단위 manifest 배정(인사이동 시나리오)을 전제로 쓰였다.
그러나 실제 계약과 구현은 전부 tenant 단위였다 — backend 명세 §5.1과 부분 유니크 인덱스
`ux_manifests_tenant_active`(tenant당 활성 manifest 최대 1), 허브 계약 §5,
`enrollment-manifest.schema.json`(team/부서 축이 없다). backend ADR-0008 Follow-up이 이 충돌을
"가정이 아니라 현재 상태"로 기록했고, 결정 없이는 어느 쪽이 이기는지 아무 문서도 답하지 못했다.

## Decision

- **manifest 배정 단위는 tenant다.** tenant당 활성 manifest는 최대 하나이며
  `ux_manifests_tenant_active`가 DB 불변식으로 강제한다.
- 계약·마이그레이션·manifest 스키마는 바꾸지 않는다 — 이미 tenant 단위로 정합한 쪽이 확정이다.
- backend ADR-0007의 Context는 tenant 내 manifest 개정(revision) 시나리오로 다시 쓴다(반영됨).

## Alternatives Considered

- **부서 단위 배정** — `manifests`에 `team_id`를 추가하거나 `installation_manifest_assignments`를
  배정 진실원으로 승격해야 하고, `ux_manifests_tenant_active` 대체 마이그레이션 + 허브 계약 §5 +
  manifest 스키마가 한 묶음으로 바뀐다. 현재 `teams`·`team_memberships`에는 manifest 배정을 위한
  **쓰기(write)** 코드·관리자 API가 없어 부서 단위 배정을 집행할 수단이 없다(읽기 소비자인 파이프라인
  org provider의 as-of 조인은 별개 — [`../contracts/data-model.md`](../contracts/data-model.md) D-3). 택하지 않았다.

## Consequences/Tradeoffs

### Positive

- 세 레포의 계약·구현·문서가 같은 답을 갖는다. 계약 변경 비용이 0이다.

### Negative

- 부서별로 다른 프라이버시 정책을 줄 수 없다. tenant 안에서는 가장 엄격한 요구에 맞춘 단일 정책이 된다.

## Follow-up

- `teams`가 실제로 쓰이게 되면(엔티티·관리자 API 구현) 부서 단위 배정을 **별도 개정 ADR**로 다시 연다.

## References

- backend ADR-0007(revision 대조) · ADR-0008 Follow-up(충돌 기록, 해소됨) · ADR-0004(부분 유니크 인덱스)
- [`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §5
