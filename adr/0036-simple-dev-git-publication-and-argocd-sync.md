# 0036. dev 배포는 짧은 Git 게시와 Argo CD 자동 동기화로 연결한다

## Status

Accepted — PROJ-334·335에서 채택한다. 기존 [ADR 0035의 고정 원본](https://github.com/soma-376/docs/blob/c8c99f90851615a2d95871f61a9063cb8361188d/adr/0035-dev-gitops-delivery.md)에 정의된 장기 배포 잠금·자동 이미지 rollback·실패 fence·초기화 readiness·CI observer 요구를 대체한다. 원본 이미지 검증, 고정 infra·values 참조, 이미지/설정 소유권과 Argo CD 채택은 유지한다. 구현·운영 적용 상태는 공급 계약의 현재 상태에 따로 기록한다.

## Context

`pulsemetry-backend`는 네 앱의 검증한 이미지를 생산하고, `infra`는 chart·runtime 입력·generated values를 관리한다. private `infra-generated-values`는 두 저장소가 선택한 배포 선언을 보관하며 Argo CD가 읽는다. 이 계약은 세 저장소와 문서 허브에 걸친다.

기존 설계는 Git 게시부터 Argo 상태 관측과 자동 rollback까지 하나의 잠금으로 묶었다. 최초 초기화에도 publisher 수락과 실행 identity를 연결해야 했으므로, 이미 수동 인계한 Application을 auto sync로 전환하는 데 별도 배포 제어 시스템이 필요했다.

공유 dev의 요구는 검토된 설정과 검증된 이미지를 Git에 게시하고 Argo CD가 최신 선언으로 수렴하게 하는 것이다. Git 충돌 방지와 실제 배포 성공 판정은 분리할 수 있다.

## Decision

### 1. 소유권과 고정 참조를 유지한다

- backend는 원본 성공 run/attempt의 네 ARM64 이미지와 완료 manifest를 검증하고 이미지 digest만 바꾼다. 현재 infra commit과 generated의 비이미지 바이트를 보존한다.
- infra는 같은 commit의 공식 profile/runtime·생성기와 현재 AWS output·Strimzi metadata로 13개 values를 생성한다. 현재 선택된 네 이미지와 원본 bundle·producer 증거를 보존한다.
- root만 private `main`의 Application 경로를 추적한다. 하위 18개 Application은 검증한 infra commit과 values snapshot commit을 고정한다. `image.repository` + `image.digest`를 유지하며 태그 감시·별도 image overlay는 도입하지 않는다.
- 원본 manifest 바이트의 hash와 생산 출처 검증을 유지한다. 후보 이미지를 검증 프로세스로 실행하지 않는다.

### 2. 게시할 때만 Git 충돌을 확인한다

현재 private `main`을 읽고 values snapshot commit을 로컬에 만든 뒤, 그 SHA를 참조하는 Application·desired commit을 만든다. 마지막 commit까지 정상 fast-forward push 한 번으로 게시한다. 자기 commit SHA를 자기 파일에 넣지 않는다.

예상 HEAD가 달라지면 최신 선택을 다시 읽고 반대 영역을 보존한 후보를 재생성한다. 시도는 최초 실행을 포함해 최대 3회이며 실패하면 workflow를 실패시킨다. force push·자동 rebase·시간 기반 잠금 탈취를 사용하지 않는다. push 응답이 유실되면 원격 이력에서 정확한 게시 commit을 확인해 중복 게시를 막는다.

현재 선택은 `dev/desired.yaml`과 그 파일이 가리키는 snapshot에서 읽는다. `dev/delivery-state.json`, `dev/activation.json`, 배포 nonce, `dev/evidence/` 또는 lastKnownGood을 새로 생성하지 않는다. 이전 상태 파일이 있으면 자동 변환하거나 삭제하지 않고 명시적 전환을 요구한다.

### 3. Argo CD가 최신 선언으로 수렴한다

root는 Application 선언만 관리하며 운영자가 한 번 등록한다. root와 하위 Application은 auto sync·self-heal을 사용한다. prune·allowEmpty·연쇄 삭제 finalizer는 끄고 retry limit은 0을 유지한다. 현재 Application health 평가와 sync wave를 재사용한다.

게시기는 Argo rollout 완료를 기다리지 않는다. Git 게시 성공과 실제 배포 성공은 별개이며, 연속 게시에서는 중간 선택이 적용되지 않고 최신 선택으로 수렴할 수 있다. Argo 실패는 자동 rollback이나 다음 게시의 전역 차단을 만들지 않는다. 운영자가 Argo 상태를 확인하고 원인을 수정하거나 검토한 선언으로 수동 복구한다.

이후 publisher는 Git에 있는 sync 정책을 보존한다. 유지보수 때 Git에서 대상 child의 자동화를 끄고 root 반영과 진행 중 operation 종료를 확인한다. root 자체의 변경도 필요하면 운영자 경로로 중지한다.

### 4. 최초 연결은 현재 배포를 보존한다

기존 18개 Application의 실제 이름·namespace·releaseName·infra 참조·values·이미지를 확인해 초기 Git 선언을 만든다. 서로 다른 values snapshot에서 사용하던 파일을 합칠 때도 기존 바이트와 렌더 결과를 보존한다. 새로운 chart나 이미지로 바꾸는 작업은 초기 연결과 구분한다.

초기 선언을 게시한 뒤 기존 관리 RBAC·AppProject·private Git reader·CMP를 사용하는 root를 운영자가 적용한다. 별도 readiness true, 가상 owner/run ID, 정상 배포 상태 파일을 만들지 않는다. backend와 infra에는 이 작업을 위한 Kubernetes 쓰기 권한을 추가하지 않는다.

## Alternatives Considered

- 장기 CAS 잠금과 자동 rollback 유지: 배포 순서와 복구를 중앙에서 통제하지만 초기화·owner 회복·호환성 위험에 비해 공유 dev 운영 부담이 크다.
- child가 각각 가변 branch 추적: 게시기는 단순하지만 함께 검증하지 않은 chart·values 조합을 읽을 수 있어 선택하지 않는다.
- workflow가 Kubernetes Application을 직접 patch: Git만 변경하는 경계를 깨고 CI에 클러스터 쓰기 권한이 필요하므로 선택하지 않는다.

## Consequences/Tradeoffs

### Positive

- 두 publisher는 작은 공통 Git 게시 규칙을 사용하고 Argo CD가 배포·drift 복구를 담당한다.
- 고정 SHA와 원본 이미지 증거를 유지하며 CI observer·자동 rollback·장기 잠금 회복 절차를 줄인다.
- root를 등록한 뒤 Application 참조를 매번 수동 수정할 필요가 없다.

### Negative

- 연속 게시의 모든 중간 버전이 실제 배포되지는 않는다. 실패한 rollout 이후에도 새 Git 선택이 게시될 수 있다.
- Git 게시 성공은 로그인·조회·수집·저장 성공을 보장하지 않는다. 실제 상태 확인과 복구는 운영자 책임이다.
- repository 단위 쓰기 권한은 YAML 필드별 권한이 아니므로 허용 파일·반대 영역 보존 검증을 유지해야 한다.
- prune 비활성화로 삭제된 선언의 리소스가 남을 수 있으며, 데이터 삭제·축소 위험은 별도 검토해야 한다.

## Follow-up

- PROJ-334는 infra·backend 게시기와 문서 계약을 정합화한다.
- PROJ-335는 현재 배포의 보존을 확인하고 root·auto sync·self-heal을 실제 dev에 연결한다.
- 완료 기록에는 코드 검증, Git 게시, Argo 적용, 실제 기능 검증을 구분한다.

## Acceptance Criteria

- 동일 내용 no-op, 동시 push, 예상 HEAD 변경, push 응답 유실에서 혼합 선택이나 중복 게시가 생기지 않는다.
- infra 설정 게시는 현재 이미지·bundle을, backend 이미지 게시는 현재 infra와 비이미지 바이트를 보존한다.
- root와 18개 child가 목표 revision의 Synced/Healthy에 도달하고 초기 child·PVC·이미지·mount가 보존된다.
- 검토한 Git 변경이 workflow → root → child로 적용되고, 비파괴 drift가 self-heal로 복구된다.

## References

- [generated values 공급 계약](../contracts/generated-values-sync.md)
- [Argo CD 자동 sync](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/)
- [Argo CD App of Apps](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
