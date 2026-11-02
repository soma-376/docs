# 0031. dev Kubernetes는 workflow가 선택한 이미지 묶음과 고정 values를 GitOps로 배포한다

## Status

Accepted — PROJ-281 개정으로 **Image Updater 도입을 철회**하고 backend workflow의 image 게시·rollback과
infra 설정 publisher를 공통 Git 잠금으로 조정한다. PROJ-280·281·286은 설계 문서이며 workflow·controller·
chart 구현이나 실제 GitOps 인수 완료를 뜻하지 않는다. Argo CD 채택은 유지한다.

[ADR 0030](0030-generated-values-private-sync-copy.md)의 private 보관·infra 입력/생성기/chart 소유권·
역동기화 금지를 계승한다. 인계 전의 사본만 보관·직접 편집 금지·infra 운영자 수동 공급 범위는 인계 후
backend의 제한된 image 변경과 완료 manifest·V/P·delivery state 게시로 부분 대체·확장한다.
[Kafka ADR 0026](0026-dev-telemetry-kafka-strimzi.md)의 인증서 metadata revision을 반영하는 실행 책임만
GitOps 인수 후 이 결정으로 바꾼다. Strimzi 소유권·mTLS·명시적 재배포·수집 성공 의미는 유지한다.

## Context

backend는 네 앱의 정확한 build digest와 원본 실행을 이미 알고 있다. registry를 다시 관찰하는 Updater는
네 이미지가 같은 SHA/run/attempt의 완성 묶음임을 보장하지 못하므로 별도 검증이 필요하다. 후보 branch·
추적 tag·image.ref 전환을 더해도 이 검증은 줄지 않는다. 원본 workflow의 성공 결과를 별도 승격 workflow가
직접 받아 Git에 기록하는 방식으로 공급 경로를 단순화한다.

`pulsemetry-backend`, `infra`, private `infra-generated-values`와 계약 소유자인 `docs`가 당사자다.
이미지와 설정을 서로 다른 publisher가 바꾸므로 repository별 workflow concurrency만으로는 충돌을 막지
못한다. 단순 Git revert는 migration·현재 runtime·키/TLS 호환성을 보장하지 않고 실패 전 생성된 실행이
늦게 끝나면 rollback 직후 같은 장애를 재배포할 수 있다. 공통 잠금과 영속적인 실패·재개 경계가 필요하다.

## Decision

### 1. 범위와 소유권

- backend는 같은 SHA/run/attempt에서 검증·게시한 네 ARM64 이미지의 완료 manifest를 만든다. 원본 게시
  workflow가 최종 성공으로 **종료된 후 별도 승격 workflow**가 private image values·선택을 갱신한다.
  rollback을 시작해도 원본 manifest의 producer run/attempt를 바꾸지 않는다.
- infra는 AWS/CDK·chart·runtime 입력·생성기·설정 검증과 config publisher를 소유한다. backend image
  publisher는 현재 C와 generated의 비이미지 바이트를 보존하고, infra config publisher는 현재 선택된
  B·원본 manifest·네 digest를 보존한다. 모두 같은 잠금·허용목록·호환성 검증을 통과해야 한다.
- Image Updater·candidate branch·registry polling·Updater Git write-back을 도입하지 않는다. `dev` tracking
  tag의 존재·최신성을 배포 입력으로 쓰지 않는다. 현행 `image.repository` + bare `image.digest`를 유지하고
  image.ref 전환과 별도 image overlay는 도입하지 않는다.
- private 저장소의 임의 수동 편집·원본 역동기화는 금지한다. backend의 image 예외는 정해진 workflow
  경로에서만 허용한다. 앱 workflow에 CDK·Helm·Kubernetes 쓰기 권한을 주지 않는다.
- Argo CD 자체는 고정한 수동 Helm release로 관리한다. 인계한 dev 앱·Kafka·ClickHouse·platform·관측은
  Argo가 적용하고 자동 sync·self-heal을 사용하며 prune·allowEmpty는 끈다. prod·frontend ECS·perf는 제외한다.

### 2. 불변 묶음과 두 단계 게시

- `C`는 infra/chart/runtime commit, `B`는 완료 manifest 원본 바이트 해시, `V`는 generated와 B의 불변
  snapshot commit, `P`는 desired와 Application source를 함께 바꾼 선택 commit이다. source와 artifact
  출처·ARM64·네 repository/digest·현재 schema/runtime/key/TLS 호환성을 검증한다.
- 잠금 소유 publisher는 V를 먼저 append한다. 이때 Application은 이전 V를 유지한다. 별도 P에서
  desired와 영향을 받는 모든 Application의 C/V full SHA를 함께 바꾼다. V나 P 자신의 SHA를 자기 파일에
  기록하지 않는다. 상태 기록 commit은 P가 아니며 main HEAD와 현재 desired P를 따로 비교한다.
- 정상 image 후보는 검증 뒤 자동 승격하며 매번 수동 승인을 더하지 않는다. 원본 workflow 실패·부분 목록·
  혼합 실행·과거 source·허용 범위 밖 변경은 거부한다. 같은 SHA 재실행의 다른 digest는 새 B로 검증한다.
  Git 게시 성공은 배포·기능 성공을 뜻하지 않는다.

### 3. 공통 잠금과 상태

private main의 `dev/delivery-state.json`을 양 publisher가 공유한다. schemaVersion 1의 필드와 수락 조건은
[단일 공급 계약](../contracts/generated-values-sync.md#4-공통-delivery-state와-잠금)이 소유한다. 상태는
`idle`, `deploying`, `rolling_back`, `blocked`이며 owner의 repository/workflow/run/attempt와 증가하는
`generation`, `expectedDesiredP`, 작업 종류, lastKnownGood, 실패 B·canonical 이미지 벡터와 resumeFence를
기록한다. 실제 state 파일·잠금 구현은 이번 문서 변경에 포함하지 않는다.

잠금 획득은 최신 main HEAD에 대한 CAS·정상 fast-forward이며 같은 시점에 한 owner/generation만 성공한다.
V/P 이후에도 배포·기능 검증·rollback까지 잠금을 보유한다. 모든 mutation 전에 owner/generation·main HEAD·
현재 P를 다시 대조하고, force push·마지막 writer 우선·timeout만으로 잠금 탈취를 금지한다. 중단된 runner와
불명확한 실제 상태는 blocked에서 운영자가 확인한다. 소유권을 잃은 이전 실행은 차단 상태 기록을 포함해
추가 쓰기를 하지 않는다. 수동 소유권 인계도 실제 실행 종료·Argo operation 종료 확인과 CAS가 필요하다.

### 4. 적용과 복구

platform·runtime·데이터 준비 뒤 enrollment Flyway 성공 → ingest → Gateway → dashboard 순서를 유지한다.
parent wave·child health는 목표 C/V의 resolved revision·Synced·Healthy·migration 및 operation 종료를
함께 확인한다. Git 선택이 네 Deployment의 동시 교체를 보장하지 않으며 혼합 버전 구간도 호환되어야 한다.
Strimzi Operator의 고정 vendor/Helm/post-render CMP, CRD/CR Ready·observedGeneration, 인증서 metadata
재배포를 유지한다. offset reset을 일반 sync hook이나 반복 Job으로 옮기지 않는다.

backend image 배포의 **현재 목표 revision 실패가 확정**되고 진행 중 operation이 없으면, 같은 owner가
실패 B·전체 이미지 벡터·source와 fence를 먼저 영속화한다. 배포 직전 lastKnownGood B가 migration이
적용된 실제 현재 schema와 C/runtime·Kafka backlog·key/TLS에 호환되는지 재검증한다. 근거가 충분한
경우에만 rolling_back으로 전환해 자동 rollback을 **한 번** 시도하며 이전 성공만으로 판단하지 않는다. 현재 비이미지 바이트를 보존한 새 V/P로 복구하고 Argo auto
sync/self-heal은 유지한다. Argo 이력 rollback·Helm rollback·DB downgrade·PVC/offset 복원은 실행하지 않는다.

unknown·infra config 실패·호환성 근거 없음·정상 복구 B 없음·소유권 유실·rollback 실패는 blocked이며
자동으로 반복 복구하지 않는다. 현재 revision·기능 검증까지 성공한 rollback만 잠금을 해제한다. 실패 B와
동일한 네 repository/digest 벡터는 manifest를 재포장해도 거부한다. 새 정상 자동 승격은 실패 source의
strict descendant인 backend develop commit이며 **원본 게시 workflow run 생성 시각이 영속 failure fence
이후**인 경우만 허용한다. 오래된 run의 늦은 완료·재시도나 승격 workflow 시작 시각은 이 조건을 대신하지
못한다. 잠금 해제 때 이미 완료된 적격 후보도 다시 조회한다. infra config는 현재 B와 실패 이력·fence를
보존하며 image 전용 descendant/run 시각 조건으로 설정 게시를 막지 않는다. config로 blocked를 우회하지 않는다.

offset reset·key/인증서 교체·수동 데이터 작업은 일반 image rollback과 분리한다. 공통 잠금·자동 승격 보류를
확보하고 필요한 parent/child의 sync·self-heal을 함께 중지한 뒤 운영한다. 대상·현재 선택·복귀값과 Git 정합·
기능 검증을 남기고 재개한다. prune off가 데이터 삭제·보존 검토를 대체하지 않는다.

## Alternatives Considered

- Image Updater 유지: backend가 이미 아는 digest를 다시 발견하면서 별도 묶음 검증·승격·복구가 필요해 철회한다.
- backend가 모든 설정을 소유: chart/runtime·외부 metadata 책임까지 이동하므로 image와 config 변경 경계를 분리한다.
- publisher마다 GitHub concurrency만 사용: 저장소 간 잠금이 아니므로 private main CAS·owner/generation을 사용한다.
- 별도 image overlay 또는 image.ref 전환: 현행 repository+digest로 충분하고 두 번째 합성·입력 규칙을 추가하지 않는다.
- 실패 시 무조건 이전 Git commit으로 복귀: 현재 schema/runtime/key 및 데이터 호환성을 보장하지 못한다.
- rollback 후 다음 webhook만 대기: 이미 완료된 정상 후보를 놓치므로 잠금 해제 시 적격 후보도 다시 조회한다.

## Consequences/Tradeoffs

### Positive

원본 빌드 결과에서 배포 선택까지 직접 추적하고 Updater 설치·후보 branch·tag 관찰을 제거한다. backend는
image·rollback을, infra는 설정을 소유하면서 공통 상태로 동시 변경을 차단한다. 실패 전 실행과 재포장된
실패 이미지를 차단하고 호환 가능한 정상 묶음의 자동 복구·재개 조건을 검증할 수 있다.

### Negative

공통 CAS·소유권·상태 확인·실패 fence·현재 schema 호환성 판정은 구현 부담으로 남는다. 배포·rollback 동안
다른 게시가 대기하며, 불명확한 장애나 config 실패는 사람이 복구해야 한다. 네 앱 중 하나가 실패하면 묶음이
지연된다. replica 1/Recreate의 중단 가능성과 별도 데이터 복구 책임은 유지된다.

## Follow-up

- backend의 완료 manifest·별도 승격/rollback과 infra 설정 publisher, 공통 Git 상태·권한·보호 규칙을 구현한다.
- Application/CMP·health·현재 revision/기능 검증·호환성 판정기를 검증한다. 구현되지 않은 gate로 자동화를 켜지 않는다.
- 최초 수동 인수는 고정 C와 별도 H로 보존한다. 과거 이미지를 새 B로 위장하지 않으며 새 정상 B·V/P·공통 상태가
  준비된 뒤 자동화한다. Argo 인계 전 수동 사본과 인계 후 publisher를 동시에 사용하지 않는다.

## Acceptance Criteria

[공급 계약의 수락 사례](../contracts/generated-values-sync.md#9-수락-사례)를 양 레포에 동일하게 적용한다.
부분·혼합·stale 후보, 비이미지 변경, cross-repo CAS 경합, 늦은 owner, state/P 구별, migration 이후 비호환,
unknown/config 실패, 한 번의 rollback, 실패 벡터 재포장, 오래된 원본 run 재시도, strict descendant·fence 이후
정상 후보의 재개와 잠금 해제 재조회를 검증한다. 문서 검증과 workflow/Argo 설치·실제 인수 증거는 구분한다.

## References

- [generated values 공급 계약](../contracts/generated-values-sync.md)
- [infra ADR 0056 — dev Argo CD 운영](https://github.com/soma-376/infra/blob/docs/PROJ-280-dev-argocd-adr/docs/adr/0056-dev-argocd-gitops-operations.md)
- [backend ADR 0076 — 검증한 이미지 묶음 인계](https://github.com/soma-376/pulsemetry-backend/blob/docs/PROJ-286-backend-image-delivery-adr/docs/adr/0076-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)
