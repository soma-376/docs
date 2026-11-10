# 0035. dev Kubernetes는 workflow가 선택한 이미지 묶음과 고정 values를 GitOps로 배포한다

## Status

Accepted — PROJ-281 개정으로 **Image Updater 도입을 철회**하고 backend workflow의 image 게시·rollback과
infra 설정 publisher를 공통 Git 잠금으로 조정한다. PROJ-280·281·286은 설계 문서이며 workflow·controller·
chart 구현이나 실제 GitOps 인수 완료를 뜻하지 않는다. PROJ-311은 불변 artifact·Argo 상태 기반 성공 판정·단일 이미지 복구 경계를 구체화한다. 기존 PostSync Job과 실환경 호환성 검사를 제거하고 기능 미탐지·비호환 복구 위험을 수용한다. Argo CD 채택은 유지한다.

[ADR 0034](0034-generated-values-private-sync-copy.md)의 private 보관·infra 입력/생성기/chart 소유권·
역동기화 금지를 계승한다. 인계 전의 사본만 보관·직접 편집 금지·infra CD 사본 공급 범위는 인계 후
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
못한다. 이전 이미지 복구는 migration·현재 runtime·키/TLS 호환성을 보장하지 않고 실패 전 생성된 실행이
늦게 끝나면 rollback 직후 같은 장애를 재배포할 수 있다. 공통 잠금과 영속적인 실패·재개 경계가 필요하다. PostSync 검증 Job·synthetic 계정·DB reader 운영 비용을 줄이기 위해 실제 기능·DB 호환성 검사를 제거하고 Argo 상태를 배포 판정으로 사용한다.

## Decision

### 0. 인계 전 CD 사본과 최초 인수 경계

PROJ-279는 성공한 infra CDK 배포 뒤 수동 Helm용 generated YAML 13개와 C만 private main에 공급한다.
최초 사본부터 CD가 생성·검증·게시하며 plan에 고정한 backend 성공 run/attempt·SHA·네 ARM64 digest와
로그 해시는 임시 입력이다. 이 입력으로 완료 manifest B를 꾸미거나 정식 GitOps 승격을 시작하지 않는다.
승인 시 활성 여부·이미지 입력 고정, metadata-only 조회, 배포 receipt와 사본 전용 재시도는
[공급 계약 §8](../contracts/generated-values-sync.md#8-유지보수최초-인수현재-상태)을 따른다.

GitOps 표식이 있는 저장소에는 이 writer를 실행하지 않는다. Argo 인계 전에 writer를 중지하고 고정 C와
실제 실행 중 이미지의 별도 handoff snapshot H를 검증한다. 사본 이미지와 현재 실행 이미지가 같다고
가정하지 않는다. 인계 이후에는 아래 publisher·완료 manifest·V/P·공통 state를 사용하며 두 경로를
동시에 운영하지 않는다. 사본 게시 성공은 Helm 적용·GitOps 인수·기능 성공 증거가 아니다.

### 1. 범위와 소유권

- backend는 같은 SHA/run/attempt에서 검증·게시한 네 ARM64 이미지의 완료 manifest를 만든다. 원본 게시
  workflow가 최종 성공으로 **종료된 후 별도 승격 workflow**가 private image values·선택을 갱신한다.
  rollback을 시작해도 원본 manifest의 producer run/attempt를 바꾸지 않는다.
- infra는 AWS/CDK·chart·runtime 입력·생성기·설정 검증과 config publisher를 소유한다. backend image
  publisher는 현재 C와 generated의 비이미지 바이트를 보존한다. generated 비이미지 바이트의 예외는 없으며 nonce는 Git 상태·증거에서만 사용한다. infra config publisher는 현재 선택된
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
  출처·ARM64·네 repository/digest·packaged contract의 원본 대응을 검증한다. 실환경 schema/runtime/key/TLS 호환성은 검사하지 않는다.
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
기록한다. PROJ-311의 공통 구현과 별도 infra 설정 publisher는 같은 파일·CAS 계약을 사용한다. 초기화와 활성화는 운영 수락이 필요하다.

잠금 획득은 최신 main HEAD에 대한 CAS·정상 fast-forward이며 같은 시점에 한 owner/generation만 성공한다.
V/P 이후에도 Argo 상태 판정·rollback까지 잠금을 보유한다. 모든 mutation 전에 owner/generation·main HEAD·
현재 P를 다시 대조하고, force push·마지막 writer 우선·timeout만으로 잠금 탈취를 금지한다. 중단된 runner와
불명확한 실제 상태는 blocked에서 운영자가 확인한다. 소유권을 잃은 이전 실행은 차단 상태 기록을 포함해
추가 쓰기를 하지 않는다. 수동 소유권 인계도 실제 실행 종료·Argo operation 종료 확인과 CAS가 필요하다.

### 4. 적용과 복구

platform·runtime·데이터 준비 뒤 enrollment → ingest → Gateway → dashboard 순서를 유지한다.
parent wave·child health는 목표 C/V의 resolved revision·Synced·Healthy 및 진행/현재 revision 실패 operation 부재를
함께 확인한다. Git 선택이 네 Deployment의 동시 교체를 보장하지 않으며 혼합 버전 구간도 호환되어야 한다.
Strimzi Operator의 고정 vendor/Helm/post-render CMP, CRD/CR Ready·observedGeneration, 인증서 metadata
재배포를 유지한다. offset reset을 일반 sync hook이나 반복 Job으로 옮기지 않는다.

backend image 배포에서 목표 C/V와 spec·comparedTo·resolved revision 및 완료 operation revision이 일치하고
현재 operation이 `Failed` 또는 `Error`일 때만 확정 실패로 판정한다. root와 모든 child의 진행 중 operation이
없으면 같은 owner가 실패 B·이미지 벡터·source와 fence를 영속화한 뒤 마지막 Argo 정상 B로 **한 번** 복구한다.
현재 C와 비이미지 설정을 보존한 새 V/P를 사용하며 Argo auto sync/self-heal은 유지한다. 실제 schema/migration·
키·TLS·Kafka backlog 호환성은 검사하지 않는다. migration 이후 이전 이미지가 비호환일 수 있는 위험을 수용한다.

unknown·infra config 실패·정상 B 부재·소유권 유실·rollback 실패는 blocked다. 과거 revision·Degraded 단독·
timeout은 확정 실패가 아니다. 복구도 현재 C/V와 root P 이력·Synced·Healthy·진행/현재 실패 operation 부재이 확인돼야
잠금을 해제한다. 실패 B와 동일한 이미지 벡터를 차단하며 실패 source의 strict descendant이자 failure fence
이후 새로 생성된 원본 생산 run의 정상 묶음부터 재개한다. unlock 때 이미 완료된 후보도 다시 조회한다.
DB downgrade·PVC·Kafka offset 복원은 자동화하지 않는다.

offset reset·key/인증서 교체·수동 데이터 작업은 일반 image rollback과 분리한다. 공통 잠금·자동 승격 보류를
확보하고 필요한 parent/child의 sync·self-heal을 함께 중지한 뒤 운영한다. 대상·현재 선택·복귀값과 Git 정합·
기능 검증을 남기고 재개한다. prune off가 데이터 삭제·보존 검토를 대체하지 않는다.

### 5. PROJ-311의 검증 경계

권한 있는 승격은 default branch의 검토된 workflow 코드로 실행한다. 생산 `develop`의 SHA·원본 run/attempt·
성공·artifact를 API로 대조하고 생산 artifact의 코드를 실행하지 않는다. 공개 manifest는 account 없는
repository·네 ARM64 digest·이미지별 배포 계약 해시를 담는다. B는 원본 바이트 해시이며 private 복사도
같은 바이트다. 승격용 `dev-gitops` Environment 역할은 기존 ECR push 역할과 분리한다.

PostSync 검증 Job·검증기 이미지·전용 synthetic identity·DB reader·probe Secret은 제거한다. runner는 기존
SSM EKS API relay와 전용 read-only 역할로 root·18개 child Application만 읽는다. Job/Pod·DB·Prometheus·
S3·Secret 조회는 하지 않는다. producer 이미지의 packaged contract와 B 원본 계약 hash 검증은 유지한다.

배포 성공은 child의 현재 spec·comparedTo·resolved C/V, `Synced`·`Healthy`, 진행 중 operation과 현재 revision에 귀속된 실패 operation 부재를 요구한다. root도 `Synced`·`Healthy`이며 resolved revision의
Git 이력에 현재 P가 포함돼야 한다. root와 child의 `retry.limit: 0`은 terminal 결과 판정을 위해 유지한다.
nonce는 Git 상태·증거에만 남고 generated values나 앱 Pod template을 바꾸지 않는다.

`lastKnownGood`은 Argo 상태 기준의 정상 묶음이며 실제 로그인·조회·수집·저장 성공을 뜻하지 않는다.
private `dev/evidence/`에는 결과·reason·Application UID·revision·operation 시각과 해시를 남긴다.
실환경 fingerprint는 제거하고 generated 비이미지 바이트의 `configurationFingerprint`만 유지한다.
PROJ-310의 실제 인계·최초 정상 B·별도 infra 설정 publisher/CAS 수락 전에는 자동화를 활성화하지 않는다.

failure fence는 unsealed 실패 기록의 CAS 성공을 확인한 뒤 새 시각으로 sealed CAS를 남긴다. 미확정
fence는 자동 재개를 막고 Git 이력의 가장 늦은 실패 시각을 보존한다. P 선택 전 최신 정상 생산 run으로
대체된 후보만 owner·기준 P를 확인한 뒤 superseded/idle로 반환한다. 다른 불명확한 실패는 blocked다.

같은 manifest의 새 V나 main 상태-only commit에서는 새 sync가 없을 수 있다. 성공은 현재 spec·comparedTo·
resolved revision과 Synced·Healthy·진행 중/현재 실패 operation 부재로 판정한다. operationState 부재나 과거
terminal operation은 성공을 막지 않는다. 현재 revision에 귀속된 완료 Failed/Error만 확정 실패로 우선한다.
root는 P ancestry와 해당 resolved revision의 desired/Application 파일이 현재 선택과 같은지도 확인한다.

## Alternatives Considered

- Image Updater 유지: backend가 이미 아는 digest를 다시 발견하면서 별도 묶음 검증·승격·복구가 필요해 철회한다.
- backend가 모든 설정을 소유: chart/runtime·외부 metadata 책임까지 이동하므로 image와 config 변경 경계를 분리한다.
- publisher마다 GitHub concurrency만 사용: 저장소 간 잠금이 아니므로 private main CAS·owner/generation을 사용한다.
- 별도 image overlay 또는 image.ref 전환: 현행 repository+digest로 충분하고 두 번째 합성·입력 규칙을 추가하지 않는다.
- 과거 values 전체 복귀: 현재 C와 비이미지 설정을 바꾸므로 채택하지 않는다. 이전 이미지만 한 번 복구하며 실환경 호환성 위험은 수용한다.
- PostSync 기능·호환성 검사 유지: 계정·검증 이미지·권한 운영 부담 때문에 제거한다.
- rollback 후 다음 webhook만 대기: 이미 완료된 정상 후보를 놓치므로 잠금 해제 시 적격 후보도 다시 조회한다.

## Consequences/Tradeoffs

### Positive

원본 빌드 결과에서 배포 선택까지 직접 추적하고 Updater 설치·후보 branch·tag 관찰을 제거한다. backend는
image·rollback을, infra는 설정을 소유하면서 공통 상태로 동시 변경을 차단한다. 실패 전 실행과 재포장된
실패 이미지를 차단하고 마지막 Argo 정상 묶음의 자동 복구·재개 조건을 검증할 수 있다.

### Negative

공통 CAS·소유권·상태 확인·실패 fence의 구현 부담은 남는다. Argo Healthy여도 기능 장애가 남을 수 있고, 실제 schema·키·TLS 호환성을 검사하지 않는 이미지 복구가 장애를 악화시킬 수 있다. 배포·rollback 동안
다른 게시가 대기하며, 불명확한 장애나 config 실패는 사람이 복구해야 한다. 네 앱 중 하나가 실패하면 묶음이
지연된다. replica 1/Recreate의 중단 가능성과 별도 데이터 복구 책임은 유지된다.

## Follow-up

- backend image 경로와 독립적인 infra 설정 publisher를 완성하고 공통 CAS 상호 운용·권한·보호 규칙을 운영 검증한다.
- Application/CMP·health·현재 revision과 operation 판정·단일 rollback을 검증한다. 기능·실환경 호환성 검사는 자동 배포 수락 조건에서 제거한다.
- 최초 수동 인수는 고정 C와 별도 H로 보존한다. 과거 이미지를 새 B로 위장하지 않으며 새 정상 B·V/P·공통 상태가
  준비된 뒤 자동화한다. Argo 인계 전 CD 사본 writer와 인계 후 publisher를 동시에 사용하지 않는다.

## Acceptance Criteria

[공급 계약의 수락 사례](../contracts/generated-values-sync.md#9-수락-사례)를 양 레포에 동일하게 적용한다.
부분·혼합·stale 후보, 비이미지 변경, cross-repo CAS 경합, 늦은 owner, state/P 구별, stale revision·진행 중 operation,
unknown/config 실패, 한 번의 rollback, 실패 벡터 재포장, 오래된 원본 run 재시도, strict descendant·fence 이후
정상 후보의 재개와 잠금 해제 재조회를 검증한다. 문서 검증과 workflow/Argo 설치·실제 인수 증거는 구분한다.

## References

- [generated values 공급 계약](../contracts/generated-values-sync.md)
- [infra ADR 0056 — dev Argo CD 운영](https://github.com/soma-376/infra/blob/feature/PROJ-311-gitops-delivery-support/docs/adr/0056-dev-argocd-gitops-operations.md)
- [backend ADR 0079 — 검증한 이미지 묶음 인계](https://github.com/soma-376/pulsemetry-backend/blob/feature/PROJ-311-backend-image-promotion/docs/adr/0079-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)
