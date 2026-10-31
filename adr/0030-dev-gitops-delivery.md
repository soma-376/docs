# 0030. dev Kubernetes는 검증한 이미지 묶음과 고정 values를 GitOps로 배포한다

## Status

Accepted — PROJ-273의 사용자 승인 설계다. PROJ-280·281·286은 문서 작업이며 컨트롤러·workflow·chart 구현이나 실제 GitOps 인수를 완료했다는 뜻이 아니다.
[generated 사본 ADR 0026](0026-generated-values-private-sync-copy.md)의 private 사본·infra 원본 소유·역동기화 금지를 계승한다. generated만 보관·모든 직접 수정 금지·사본 공급과 수동 배포에 한정한 범위는 네 image.ref 예외·완료 manifest·V/P 선택으로 부분 대체·확장한다.
[Kafka ADR 0026](0026-dev-telemetry-kafka-strimzi.md)의 Kafka 인증서 metadata revision을 수동 Helm 배포로 반영하는 실행 주체만 GitOps 인수 후 이 결정으로 대체한다. Strimzi의 인증서 소유권·mTLS·명시적 앱 재배포·수집 성공 의미는 유지한다.

## Context

backend의 네 앱 이미지, infra의 chart·runtime 입력·생성기, private 저장소의 실제 values는 서로 다른 revision으로 움직인다. 이미지 tag 네 개를 순차 갱신하면 그 중간에 서로 다른 실행의 이미지를 읽을 수 있다. generated를 재생성하는 CD와 이미지를 갱신하는 컨트롤러가 같은 파일을 쓰면 검토한 이미지가 되돌아갈 수도 있다.

이 결정은 `pulsemetry-backend`, `infra`, `infra-generated-values`와 계약 소유자인 `docs`에 적용한다. dev의 검증된 Strimzi·ClickHouse·앱·관측 자원을 기존 Helm 배포에서 인계하되, 데이터와 자격 공급 책임을 유지해야 한다.

## Decision

### 1. 소유권과 범위

- backend는 같은 backend commit과 같은 workflow run/attempt에서 성공한 enrollment-api·telemetry-ingest·telemetry-gateway·dashboard-api의 ARM64 image digest 네 개를 하나의 완료 manifest로 게시한다. Gateway가 빠지면 완료 묶음이 아니다.
- infra는 AWS/CDK·Kubernetes chart·runtime·생성기·검증 및 승격 정책을 소유한다. backend CI에는 Kubernetes 적용이나 infra/private desired 변경 권한을 주지 않는다.
- `infra-generated-values`는 private으로 유지한다. 비이미지 generated 값은 infra의 사본이며, Image Updater가 비배포 candidate의 네 앱 `image.ref`를 수정하는 경우만 직접 수정의 예외다. 사람이 generated를 고치거나 원본으로 역동기화하지 않는다.
- Argo CD와 Image Updater 자체는 운영자가 고정된 Helm release로 설치·갱신한다. 자체 관리 Application으로 옮기지 않는다. 기존 Helm 소유권을 인계한 dev 앱·Kafka·ClickHouse·platform·관측 Application은 모두 automated sync와 self-heal을 사용하고 prune은 끈다.
- prod, frontend ECS와 격리 perf는 범위 밖이다. perf 비교 중 이미지·노드·계측 고정은 [ADR 0029](0029-perf-strimzi-readiness-and-path-comparison.md)를 따른다.

### 2. 후보와 배포 선택의 분리

- Image Updater는 `ImageUpdater` CRD로 구성한다. 네 앱 모두 기존 CI의 `dev` tag를 사용하며 Gateway에도 같은 tag 공급을 추가한다. native updater는 `digest` updateStrategy로 단일 mutable `dev` tag를 추적하고 platform은 `linux/arm64`로 제한한다. tag는 후보 발견에만 쓰고 native Helm image 갱신 대상은 digest가 포함된 전체 참조 `image.ref`다. 별도 image overlay는 만들지 않는다.
- Updater는 private `candidate/dev` branch에 Git write-back한다. source/write branch는 `main:candidate/dev`로 구분한다. candidate는 Argo 배포 source가 아니며 중간의 혼합된 이미지 조합을 배포하지 않는다. Argo CD의 Git 자격은 읽기 전용, Updater의 쓰기 경로는 candidate로 제한한다.
- infra 승격기는 네 digest가 한 완료 manifest에 속하는지, ARM64·이미지 내용·chart/runtime과의 호환성·기준 revision·허용 파일/필드·비밀 원문 부재를 검증한다. 실패·부분 게시·오래된 실행·출처 불일치는 승격하지 않는다.
- `C`는 chart·생성기·추적 입력의 infra commit, `B`는 backend 완료 manifest의 바이트 해시다. 검증 결과를 보호된 private `main`에 불변 values snapshot commit `V`로 먼저 append한다. 이때 기존 Application pin은 유지하고, 별도 desired 선택 commit `P`가 `dev/desired.yaml`과 영향을 받는 모든 Application을 함께 바꿔 `C`·`V`·`B`를 선택한다. `V`가 자신의 commit SHA를 담는 순환 구조를 만들지 않는다.
- Argo root는 protected `main`의 `dev/applications/`를 추적하고 child Application의 실제 source는 `C`와 `V`의 full SHA로 고정한다. `infra-commit.txt`를 저장한 사실만으로 source가 고정됐다고 보지 않는다. 각 실행의 실제 resolved revision과 기대 repository/digest, 그 digest가 선택하는 ARM64 manifest를 실제 Pod imageID에 대조한다. index digest와 ARM64 child manifest digest의 단순 문자열 일치를 요구하지 않는다.
- 정상 후보는 검증 성공 후 자동 승격한다. 매 이미지 묶음마다 추가 수동 승인을 요구하지 않는다. 같은 backend SHA라도 digest가 달라지면 새 `B`로 전체 검증한다. breaking/schema/runtime/key 변경은 승격 전에 명시적으로 조정한 호환성·배포 계획을 반영한다.

### 3. 배포 순서와 운영

- 네 앱은 `enrollment-api(Flyway 성공) → telemetry-ingest → telemetry-gateway → dashboard-api` 순서로 갱신한다. 기존 image가 실행 중인 동안의 신구 혼합 구간도 호환되어야 한다. 완료 manifest가 Kubernetes의 원자적 네 앱 교체를 뜻하지 않는다.
- parent app-of-apps의 sync wave와 child Application custom health로 순서를 강제한다. 선행 child의 목표 source revision 일치, `Synced`, `Healthy` 및 migration 성공을 확인하기 전 다음 단계를 열지 않는다.
- Strimzi Operator/CRD와 Kafka·KafkaNodePool·KafkaTopic·KafkaUser 선언을 분리하고 Operator Ready 뒤 CR을 동기화한다. 기존 Strimzi Operator post-render 변환은 Operator Application의 전용 CMP에서 유지하고 같은 입력의 최종 manifest를 대조한다. Kafka CR 등 다른 Application은 native Helm rendering을 사용하며 CMP와 child health의 구현 세부는 infra ADR 0054가 소유한다. Strimzi가 생성한 Pod·Service·PVC·인증서 Secret은 operator가 소유한다.
- 토픽 생성·설정은 KafkaTopic/Topic Operator만 담당한다. offset reset은 별도 수동 운영 workflow의 group 확인·중지·dry-run·실행·재조회 절차로 수행한다. 일반 sync로 Job을 반복 실행하지 않는다.
- runtime·JWT/OIDC·Kafka 인증서 변경은 명시적인 입력·metadata revision과 재배포로 적용한다. 기존 AWS Secret의 CSI 공급과 Strimzi 인증서 Secret 예외를 유지하며 Secret 원문은 어떤 Git branch에도 저장하지 않는다.
- 유지보수와 복구는 대상·사유·현재 `P/C/V/B`·복귀값을 기록하고 승격기·Updater·해당 parent/child의 auto sync/self-heal을 정해진 범위에서 중지한다. parent가 child의 중지를 즉시 복구하지 않도록 인계한다. 작업 뒤 desired와 실제 상태를 일치시키고 검증한 순서로 재개한다.
- prune을 끈 상태의 삭제·퇴역은 별도 검토·운영 절차다. Namespace·PVC·CRD·Kafka CR·기존 Helm release를 인수 과정에서 삭제·재생성하지 않는다.

### 4. 실패와 증거

- 게시와 승격은 환경별로 직렬화하며 `P` 갱신 직전 현재 기준을 다시 비교한다. 다른 writer가 바꾼 상태에 덮어쓰거나 force-push하지 않는다. 충돌은 새 기준에서 전체 검증한 뒤 재시도한다.
- 실패한 candidate는 현재 desired를 바꾸지 않는다. desired 적용 실패는 rollout 실패로 기록하고 다음 자동 승격을 중지한다. 복구는 현재 비이미지 설정을 보존하고, 현재 데이터/schema/runtime/key와 호환되는 이전 정상 네 앱 묶음 `B`로 새 `V`를 생성·검증한 뒤 새 `P`에서 선택하는 것을 기본으로 한다. 과거 `C/V/B` 전체 복원은 현재 상태와의 호환성을 별도 검증했을 때만 허용하며 임의의 YAML 일부를 섞지 않는다.
- Git 되돌리기는 DB migration·Kafka offset·PVC·데이터 삭제의 역연산이 아니다. 되돌릴 수 없는 변경은 별도 복구 계획을 따른다. 재개 전에 실패 이미지가 다시 후보가 되지 않도록 추적 tag와 제외 상태를 대조한다.
- 이미지 게시, candidate 검증, `V` 게시, desired 선택, Argo sync, health, 실제 기능 검증을 구분해 기록한다. Git push나 `Synced` 하나만으로 배포 성공을 선언하지 않는다.

## Alternatives Considered

- generated 전체를 Updater가 직접 덮어쓴다: 생성기와 입력·runtime 소유권이 충돌하므로 네 앱의 image 필드 예외로 제한한다.
- 별도 이미지 overlay를 만든다: writer를 나눌 수 있으나 사용자가 선택한 generated 직접 갱신 방식 대신 두 번째 값 합성 규칙을 만든다.
- Updater의 Application parameter 변경만 사용한다: Git에서 선택·복구할 수 있는 불변 묶음을 남기지 못한다.
- ECR의 순차 `dev` tag 갱신을 완료로 취급한다: 일부 앱 게시 실패와 서로 다른 실행의 혼합을 구별하지 못한다.
- 모든 후보에 사람 승인을 추가한다: 이번에 선택한 검증 후 자동 승격 정책과 다르다.

## Consequences/Tradeoffs

### Positive

앱 게시와 인프라 적용 권한을 분리하면서 실제 실행 조합을 `P/C/V/B`로 추적한다. 후보 단계에서 이미지 혼합과 생성기 충돌을 잡고 기존 상태를 보존해 인수한다.

### Negative

완료 manifest·candidate 검증·불변 snapshot·desired 선택의 여러 단계가 필요하다. 네 앱 중 하나가 실패하면 전체 후보가 지연된다. self-heal 때문에 수동 운영은 명시적 중지·복귀가 필요하며 prune을 끈 자원의 삭제는 자동화되지 않는다. replica 1/Recreate의 기존 중단 가능성은 남는다.

## Follow-up

- 구현은 PROJ-273의 후속 구현 작업과 backend 후속 작업에서 수행한다. controller 버전/CRD, 권한, 완료 manifest·candidate 승격기, `image.ref`, CMP, child health, 관측 및 복구를 계약 수락 조건으로 검증한다.
- 최초 활성화 전에 네 앱의 `dev` tracking tag를 검증한 현재 digest로 초기화한다. Gateway도 필수이며 누락 시 자동 승격을 켜지 않는다. 완료 B가 없는 기존 이미지의 수동 인수는 고정 C와 별도 handoff snapshot H로 직접 관리하고 V/P·root 자동화와 분리한다. 최초 자동 승격은 새 정상 B를 검증한 뒤 시작한다.
- [PR #16](https://github.com/soma-376/docs/pull/16)의 PROJ-276 선행 브랜치에 최신 main을 병합한 이력을 기반으로 이 결정을 쌓는다. 기존 사본 원칙을 계승하고 단일 [공급 계약](../contracts/generated-values-sync.md)을 갱신한다. 로컬 이력의 계승과 원격 PR 갱신·병합·실제 인수는 구분한다.
- [PR #11](https://github.com/soma-376/docs/pull/11)의 미병합 dev EKS 수동 배포 ADR을 현행 채택 기록으로 취급하지 않는다. 현재 main의 ADR 0023은 과거 시드 결정이며 이 GitOps ADR의 전신이 아니다.

## Acceptance Criteria

[공급 계약](../contracts/generated-values-sync.md)의 수락 사례가 양쪽 레포 문서와 일치해야 한다. 문서 검증은 링크·ADR 인덱스·대체 관계·소유권·실패 및 재개 조건의 정합성을 확인한다. 실제 후속 검증에서는 부분/혼합/오래된 후보 거부, 허용 필드 외 변경 거부, C/V 불일치·P 동시 변경 거부, 순서와 health 대기, 유지보수 및 실패 복구, 기존 데이터·인증서·PVC 보존을 확인한다.

## References

- [PROJ-273](https://team376.atlassian.net/browse/PROJ-273), [PROJ-280](https://team376.atlassian.net/browse/PROJ-280), [PROJ-281](https://team376.atlassian.net/browse/PROJ-281), [PROJ-286](https://team376.atlassian.net/browse/PROJ-286)
- [생성된 values와 GitOps 공급 계약](../contracts/generated-values-sync.md)
- [infra ADR 0054 — dev Argo CD 운영](https://github.com/soma-376/infra/blob/develop/docs/adr/0054-dev-argocd-gitops-operations.md)
- [backend ADR 0075 — 검증한 이미지 묶음 인계](https://github.com/soma-376/pulsemetry-backend/blob/develop/docs/adr/0075-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)
- [Image Updater Git write-back](https://argocd-image-updater.readthedocs.io/en/stable/basics/update-methods/)
