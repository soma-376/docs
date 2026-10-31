# 계약 — dev generated values와 검증한 GitOps 배포 묶음

| 항목 | 내용 |
|---|---|
| 당사자 | `pulsemetry-backend` → `infra` → private `infra-generated-values`; 계약은 `docs` 소유 |
| 사본 저장소 | `soma-376/infra-generated-values`, private, desired 기본 브랜치 `main` |
| 관련 결정 | [generated 사본 ADR 0026](../adr/0026-generated-values-private-sync-copy.md)을 계승·부분 대체하는 [ADR 0030 — 검증한 묶음의 GitOps 배포](../adr/0030-dev-gitops-delivery.md) |
| 상태 | 변경 중 — 설계 Accepted, 관련 문서·구현·실제 GitOps 인수는 별도 검증 |

## 1. 소유권과 적용 범위

backend는 네 앱 이미지와 완료 manifest를 게시한다. infra는 chart·입력·생성기·검증·승격·배포·복구를 소유한다. private 저장소는 실제 ARN·내부 endpoint를 포함할 수 있는 generated 사본, 완료 manifest 사본과 배포 선택 metadata를 보관한다. 앱 CI가 Kubernetes나 desired를 변경하지 않는다.

Argo CD는 Git을 읽고 선택된 선언을 적용한다. Image Updater는 `ImageUpdater` CRD의 native Helm 대상 `image.ref`만 `candidate/dev` branch에서 수정한다. candidate는 배포 source가 아니다. 사람이 generated를 편집하거나 infra로 역동기화하지 않는다. 별도 이미지 overlay는 사용하지 않는다.

대상은 dev EKS 앱·Kafka·ClickHouse·platform·관측이다. Argo CD·Image Updater 자체는 수동 Helm 관리다. prod, frontend ECS와 perf는 대상에 포함하지 않는다. [perf ADR 0029](../adr/0029-perf-strimzi-readiness-and-path-comparison.md)의 이미지·노드·계측 고정은 유지한다.

## 2. 식별자와 backend 완료 manifest

| 식별자 | 의미 |
|---|---|
| `C` | chart·생성기·추적된 runtime 입력의 infra 전체 commit SHA |
| `B` | backend 완료 manifest 원본 UTF-8 바이트의 SHA-256. 내용이 달라지면 다른 묶음 |
| `V` | 검증된 generated와 manifest 사본이 담긴 private 불변 snapshot commit SHA |
| `P` | 보호된 private `main`에서 C·V·B를 함께 선택하는 별도 desired commit SHA |

완료 manifest는 네 앱의 게시와 검증이 모두 성공한 뒤 한 번 게시하며, 다음 정보를 필수로 제공한다.

| 필드 | 요구사항 |
|---|---|
| `schemaVersion` | `1`. 미지원 버전은 추측하지 않고 거부 |
| `repository`, `backendCommit` | `soma-376/pulsemetry-backend`와 전체 backend commit SHA |
| `workflowRef`, `workflowRunId`, `workflowRunAttempt` | 검증한 게시 workflow와 동일 실행·시도 식별자 |
| `status` | 네 앱 전체 완료를 나타내는 `complete`. 부분 성공은 게시 대상 아님 |
| `images` | 아래 정확히 네 앱을 key로 하고 각 `repository`, `digest`, `platform`을 기록 |
| `enrollmentAssets` | enrollment 이미지에 포함한 고정 CLI·GUI 릴리스/자산 lock과 checksum 검증 결과를 식별 |
| 검증 증거 참조 | 실행 결과·빌드 출력 digest와 ARM64 manifest 검사 결과를 원본 실행에 연결 |

앱 key는 `enrollment-api`, `telemetry-ingest`, `telemetry-gateway`, `dashboard-api`다. 각 repository는 해당 앱의 기존 `soma-376/` ECR repository로 제한하고 platform은 `linux/arm64`, digest는 실제 게시한 `sha256` digest다. tag 문자열이나 실행 시점에 다시 조회한 tag만으로 build output digest를 대체하지 않는다.

`B`는 manifest 바이트 바깥에서 계산·기록하므로 manifest가 자기 해시를 포함하는 순환을 만들지 않는다. 사본을 전달할 때 원본 바이트를 유지하며 다시 직렬화한 내용을 같은 `B`라고 표시하지 않는다. manifest·이미지 출처와 성공한 workflow 실행을 대조하고 이름이 같은 임의 파일은 신뢰하지 않는다.

네 이미지는 같은 `backendCommit`, `workflowRunId`, `workflowRunAttempt`에서 나온 것이어야 한다. 같은 SHA의 재실행으로 digest가 달라져도 덮어쓰기나 동일 묶음 생략을 하지 않고 새 `B`로 다시 검증한다. 네 개의 `dev` tag를 순차 갱신한 사실은 완료 manifest를 대신하지 않는다. 동일 source·digest로 재실행해 새 B가 생긴 경우에도 새 실행의 출처를 재검증하되, 내용이 같은 Pod의 불필요한 재시작을 요구하지 않는다. 최초 자동화 활성화에는 Gateway를 포함한 네 앱의 `dev` tracking tag와 현재 정상 digest를 먼저 대조한다. 기존 CI의 `dev` 공급을 Gateway까지 확장하며 다른 tag 선택을 후속 구현의 재량으로 남기지 않는다.

## 3. private 파일과 수정 경계

| 경로 | 의미와 writer |
|---|---|
| `dev/generated/*.yaml` | infra 생성 사본. Updater의 네 image.ref 예외만 허용 |
| `dev/infra-commit.txt` | 생성 원본 C 전체 SHA 한 줄. infra 작성 |
| `dev/bundle.json` | backend 완료 manifest의 원본 바이트 사본. infra 작성, 해시가 B |
| `dev/desired.yaml` | 선택한 C·V·B. infra 승격기만 작성 |
| `dev/applications/*.yaml` | C의 infra 템플릿에서 생성한 child Application. infra 승격기만 작성 |
| 루트 `README.md`, `AGENTS.md` | 소유권·허용 파일·담당자와 이 계약의 안내 |

`dev/desired.yaml`은 `schemaVersion: 1`, `infraCommit: C`, `valuesCommit: V`,
`bundleManifestSha256: B`를 필수로 가진다. C와 V는 full SHA이며 이를 참조한 P 자체의 SHA는
자기 파일에 쓰지 않고 Git 이력에서 얻는다. private 파일에 chart·생성기 소스를 복제하지 않는다.

`dev/generated/`의 비이미지 값은 `C`의 생성기·추적 입력·검증한 외부 참조로 재생성한다. `dev/infra-commit.txt`에는 `C` 한 줄을 기록한다. 같은 생성 실행의 파일을 묶어 공급하며 파일 누락·서로 다른 실행의 혼합·다른 C의 chart 조합을 거부한다.

현행 Strimzi 기본 생성 목록은 `base.yaml`, `lbc.yaml`, `runtime.yaml`, `enrollment-api.yaml`, `telemetry-ingest.yaml`, `telemetry-gateway.yaml`, `telemetry-kafka-admin.yaml`, `dashboard-api.yaml`, `clickhouse.yaml`, `fluent-bit.yaml`, `ascp.yaml`, `observability.yaml`, `kafka-strimzi.yaml`이다. `telemetry-kafka-admin.yaml` 공급은 Job 실행 요청이 아니다. 정적 chart·Loki/Tempo/otel-platform 설정과 lock의 원본은 infra에 남는다.

허용목록은 현재 generator의 Strimzi 출력과 대조한다. 새 출력·관측 입력이 필요하면 생성기와 계약을 함께 변경하며 디렉터리 전체를 무조건 복사하지 않는다. MSK bootstrap·broker 목록을 현행 Strimzi 입력으로 되살리지 않는다.

| writer | 허용 수정 |
|---|---|
| infra 생성·승격기 | generated 묶음, 대응 C, 원본 완료 manifest 사본, 검증 provenance와 desired 선택 metadata |
| Image Updater | 비배포 candidate의 네 앱 YAML에 있는 `image.ref`만 |
| Argo CD | Git 읽기만. private source를 수정하지 않음 |
| 운영자 | infra 원본 변경 또는 정해진 복구·승격 절차. generated 임의 편집은 금지 |

`image.ref`는 `<허용 ECR repository>[:tag]@sha256:<64자리 16진수>`인 전체 참조이며 digest를 필수로 한다. native updater가 만드는 `:dev@sha256:...`도 허용하되 선택·검증의 기준은 digest다. chart는 이 전체 참조를 그대로 렌더해야 한다. 기존 `image.repository`·`image.digest` 조합과 `image.ref`를 동시에 입력해 불명확한 우선순위를 만들지 않는다. 해당 chart·생성기 변경은 후속 구현이며 첫 활성화 전에 같은 C에서 함께 검증한다.

허용 예외는 image 필드뿐이다. candidate의 runtime·replica·ServiceAccount·Secret 참조·annotation·C·완료 manifest·다른 YAML 변경은 거부한다. 재생성은 최신 생성 결과에서 시작해 검증한 B의 네 image ref를 적용한다. 이전 candidate의 비이미지 변경을 병합하거나 마지막 writer의 값을 채택하지 않는다.

Secret 원문·token·개인키·비밀번호, kubeconfig, AWS 조회 원본, 로그·개인 경로를 넣지 않는다. Kafka TLS 입력은 Secret 이름과 UID/resourceVersion에서 유래한 metadata revision만 허용한다. 실제 ARN·endpoint는 private 사본 안에서만 다루고 공개 PR·문서로 복사하지 않는다. `0600`은 Git이 보존하지 않으므로 로컬 작업 사본의 권한은 별도로 적용한다.

## 4. candidate에서 desired까지

1. backend는 네 앱 완료 manifest와 이미지의 immutable 참조를 게시한다. infra가 출처와 manifest 해시 B를 검증한다.
2. infra는 현재 선택된 V/C를 기준으로 candidate/dev를 재생성한다. 새 기준은 정상 commit으로 기록하며 이전 candidate의 혼합 image나 비이미지 변경을 이어받지 않는다. Updater는 `main:candidate/dev`로 source와 write branch를 분리한다. native `digest` updateStrategy로 네 앱 각각의 단일 mutable `dev` tracking tag를 추적하고 platform은 `linux/arm64`로 제한해 후보 digest를 쓴다. tag 갱신 도중 혼합 상태가 생겨도 배포하지 않는다.
3. infra 승격기는 후보 네 digest와 하나의 완료 B를 대조한다. 다른 SHA·run/attempt·부분 목록·잘못된 repository/platform·누락 image는 거부한다.
4. 고정 C의 생성 결과와 후보를 비교해 image.ref 이외의 변경을 거부한다. chart·runtime·DB migration·envelope·key의 호환성, Secret 원문 부재, 파일 허용목록과 렌더 결과를 검사한다. Strimzi는 같은 post-render 결과까지 비교한다.
5. 성공한 generated·C와 B의 원본 사본을 보호된 main에 먼저 append한다. 이 commit을 snapshot V로 식별한다. 이 단계에서는 `dev/desired.yaml`과 Application의 기존 V pin을 유지하므로 새 snapshot이 곧 배포되지 않는다. V를 force-push하거나 Git 이력에서 제거하지 않고 rollback 가능 기간 동안 보존한다. 별도의 snapshot branch는 만들지 않는다.
6. 환경별 단일 publisher의 승격 잠금 안에서 최신 source·현재 desired를 다시 대조한다. C·V·B와 검증 결과가 일치하면 별도 P commit 하나에서 `dev/desired.yaml`과 영향을 받는 모든 `dev/applications/*.yaml`의 source C/V를 함께 갱신한다. snapshot 게시와 desired 선택은 별도 커밋이다.
7. Argo root는 보호된 main의 `dev/applications/`를 읽는다. P가 선택한 각 child source는 C와 V의 full SHA로 pin한다. candidate·움직이는 ECR tag·임의 latest chart를 배포 source로 쓰지 않는다.

정상 후보는 검증 성공 뒤 자동 승격한다. 별도의 매번 수동 승인 단계를 추가하지 않는다. 보호된 main에는 검증된 승격 경로만 쓰도록 권한과 보호 규칙을 설정한다. GitHub 저장소별 concurrency만으로 backend·Updater·infra·운영 작업의 상호 배제가 성립한다고 보지 않는다.

P 갱신은 V 게시 이후 확인한 main HEAD를 expected ref로 compare-and-swap하고, 이전에 선택된 desired P와 source 기준도 함께 대조한다. V append로 main HEAD가 전진한 사실을 이전 desired P의 변경으로 혼동하지 않는다. 다른 실행이 main 또는 desired를 바꾸거나 source가 오래되면 중단하고 최신 기준에서 전체 검증한다. 오래된 실행의 재실행이 더 최신 desired를 덮어쓰지 못하게 한다. manifest B만 같고 C 또는 runtime이 바뀐 경우에도 기존 검증 결과를 재사용하지 않는다.

V 게시 뒤 P 게시가 실패하면 기존 배포 선택을 유지한다. V commit 자체는 main에 남겨 두고 같은 검증 입력과 현재 기준이 여전히 맞는지 확인한 뒤 재시도한다. P 게시 응답이 유실되면 실제 main의 선택을 읽어 성공 여부를 판정하고 중복 선택을 만들지 않는다. 생성/게시/승격 단계별 완료표시를 분리한다.

## 5. 적용 순서와 성공 판정

V/P와 root로의 인계가 완료된 Application은 모두 automated sync·self-heal을 사용하고 prune은 끈다. 최초 인수는 동일 release 이름·namespace·resource identity·PVC·runtime·digest를 보존해 diff를 확인한다. Helm과 Argo가 같은 리소스를 계속 동시에 갱신하지 않는다.

기존 이미지가 완료 manifest 도입 전의 것이고 같은 SHA/run 증명이 없으면 관리자는 고정 C와 별도 handoff values commit H로 child Application을 직접 생성해 수동 인수한다. H는 이 계약의 V가 아니며 인수 기록은 별도 운영 증거로 보존한다. 이 단계는 `dev/desired.yaml`의 P 및 root 자동관리 대상에 아직 연결하지 않고 기존 이미지를 완료 B로 위장하지 않는다. 새 정상 B가 검증되면 V/P를 생성하고 root에 인계해 자동화를 활성화한다. 최초 자동 이미지 승격부터는 B·V·P의 필수 계약을 모두 적용한다.

platform·controller·runtime·데이터 의존성을 먼저 준비한다. Strimzi Operator/CRD Application의 Ready 뒤 Kafka CR을 적용하며, operator가 생성한 리소스를 Argo manifest로 별도 소유하지 않는다. Strimzi Operator Application만 기존 post-render를 유지하는 전용 CMP를 사용하며 Kafka CR 등 다른 Application은 native Helm rendering을 사용한다. CMP와 child health의 구현 세부는 infra ADR 0054가 소유한다.

앱 순서는 enrollment(Flyway 성공) → ingest → Gateway → dashboard다. parent의 child Application wave와 custom health에서 목표 revision 일치·Synced·Healthy를 함께 검사한다. 기존 healthy 상태만으로 새 revision 완료를 인정하지 않는다. enrollment의 migration 성공 확인 없이 consumer를 갱신하지 않는다. 선행 앱이 실패하면 뒤 단계를 열지 않으며 신구 앱의 공존을 호환성 검증에 포함한다.

배포 성공은 다음 증거를 함께 요구한다.

- 실제 resolved P/C/V/B의 일치. 네 앱의 기대 repository/digest와 그 digest가 선택하는 ARM64 manifest를 실제 Pod imageID에 대조하며, index digest와 ARM64 child manifest digest의 단순 문자열 일치를 요구하지 않는다.
- 목표 revision에서 Synced·Healthy와 migration 성공, 필요한 service/target 연결 확인.
- enrollment·dashboard 핵심 기능, Gateway ACK→Kafka consume/commit→S3/ClickHouse의 해당 변경에 필요한 검증.
- 기존 PVC·Kafka offset·인증서·Secret 공급과 주요 관측 경로의 보존.

이미지 게시 성공, P 선택 성공, Synced, Healthy, 기능 검증은 서로 다른 상태다. Kafka Gateway ACK가 최종 저장 완료를 뜻하지 않는 기존 계약도 유지한다.

## 6. 유지보수·복구·삭제

offset reset, 인증서/JWT 교체, 수동 데이터 작업, rollback은 대상과 현재 P/C/V/B·복귀값을 먼저 기록한다. 승격기와 Updater, 해당 parent/child의 auto sync·self-heal을 조정해 의도한 중지·replica 변경을 즉시 되돌리지 않도록 한다. 부모가 자식의 중지 설정을 복구하는 경로도 함께 다룬다. 전체 dev를 불필요하게 멈추지 않으며 대상별 중지·재개 순서는 infra 런북이 소유한다.

breaking schema/runtime/key 변경은 후보를 승격하기 전에 명시적으로 조정한다. 자동화가 DB migration을 되돌리거나 현재 runtime과 맞지 않는 과거 image를 선택하지 않는다. 사용자 JWT의 verifier→signer 순서와 구 키 겹침 시간은 해당 키 계약을 유지한다. Kafka 인증서는 metadata revision과 명시적 재배포로 새 파일을 읽게 한다.

적용 실패는 뒤 rollout과 다음 승격을 멈추고 원인을 기록한다. 복구는 현재 비이미지 설정을 보존하고 현재 데이터/schema/runtime/key와 호환되는 이전 정상 네 앱 B를 선택해, 새 V를 생성·검증한 뒤 새 P로 승격하는 것을 기본으로 한다. 과거 C/V/B 전체 복원은 현재 상태와의 호환성을 별도 검증했을 때만 허용한다. 만료된 key·인증서·과거 MSK 설정을 되살리지 않으며 임의의 YAML 일부와 다른 revision의 image·chart를 섞지 않는다. Git rollback은 DB migration·Kafka offset·PVC·업무 데이터의 역연산이 아니다.

복구 중 Updater가 실패 image를 다시 선택하지 않도록 후보·tracking tag·제외 상태를 확인한다. desired와 실제 상태 및 기능이 일치하면 parent/child와 승격기·Updater를 정해진 순서로 재개하고 다음 정상 후보가 적용되는지 확인한다. 중지 상태를 운영자 기억에만 남기지 않는다.

prune off는 자원 삭제 승인이나 보존 보장의 대체가 아니다. Namespace·CRD·Kafka CR·PVC·Application·Helm release 삭제는 별도 영향 확인과 명시적 운영 절차로 수행한다. Strimzi가 생성한 자원의 수명은 Strimzi 선언과 기존 보존 정책이 소유한다. 토픽 생성 Job을 추가하지 않으며 offset reset Job을 일반 sync hook으로 실행하지 않는다.

## 7. 수락 사례

| 사례 | 요구 결과 |
|---|---|
| Gateway 누락·네 앱 중 한 빌드 실패 | 완료 B·승격 없음, 현재 P 유지 |
| 같은 SHA이나 다른 run/attempt의 digest 혼합 | 거부 |
| 같은 SHA의 정상 재실행으로 digest 변경 | 새 B로 전체 검증 후 자동 승격 가능 |
| 동일 digest의 정상 재실행 | 새 B 출처를 검증하되 불필요한 Pod 재시작은 요구하지 않음 |
| enrollment CLI·GUI 자산 lock/checksum 근거 누락 | 완료 manifest 검증·승격 거부 |
| candidate에서 image 외 필드 변경 | 거부, 원본 입력으로 수정하도록 인계 |
| generated 재생성과 Updater 충돌 | 새 C의 생성 결과와 B로 다시 검증, 마지막 writer 우선 금지 |
| C/V 불일치·누락 파일·불변 snapshot 미존재 | P 게시 거부 |
| 오래된 실행·P 동시 변경 | compare-and-swap 실패 후 재검증, force-push 금지 |
| snapshot 게시 뒤 desired 실패·응답 유실 | 현재 P 조회로 상태 확인, 중복 적용/조합 혼합 금지 |
| enrollment migration 또는 선행 child 실패 | 다음 앱 wave 및 다음 승격 정지 |
| 유지보수 중 ingest 중지·offset reset | self-heal과 경쟁하지 않으며 재개 뒤 정상 처리 확인 |
| rollback 직후 updater 재실행 | 실패 image 재승격 방지와 정상 다음 후보 적용 확인 |
| 기존 Helm 인수 | resource identity·PVC·offset·Secret 공급 보존, 중복 writer 없음 |

## 8. 현재 상태와 인계

PROJ-280·281·286은 승인 설계를 문서화한다. 완료 manifest, image.ref chart 입력, candidate write-back, 승격기, snapshot/desired 선택, 권한·branch 보호, Application/CMP/custom health와 실제 인수는 후속 구현·검증 대상이다. 기존 Strimzi 배포 검증 완료와 GitOps 자동화 완료를 혼동하지 않는다.

이번 변경은 [PR #16](https://github.com/soma-376/docs/pull/16)의 PROJ-276 선행 브랜치에 최신 main을 병합한 이력을 기반으로 한다. 기존 [generated 사본 ADR 0026](../adr/0026-generated-values-private-sync-copy.md)의 private 사본·infra 원본 소유·역동기화 금지 원칙을 계승하고, Updater image 필드 예외·완료 manifest·V/P 선택을 같은 계약에 반영한다. generated 사본 ADR 0026과 Kafka ADR 0026은 기존 파일명·링크를 보존하며 번호만으로 서로를 지칭하지 않는다. 로컬 계승은 원격 PR 병합이나 GitOps 인수 완료를 뜻하지 않는다.

기존 수동 사본 공급·사용 절차는 Argo 인수 전의 Strimzi 13개 generated 파일과 대응 C에만 적용한다. 대상 저장소에 `dev/bundle.json`·`dev/desired.yaml`·`dev/applications/`가 있으면 해당 수동 절차를 중단하며 인수 후에는 이 계약의 publisher 경로만 사용한다. H 수동 인수의 기록과 V/P 자동화 상태를 일반 사본 복사로 덮어쓰지 않는다.

구현·운영은 [infra ADR 0054](https://github.com/soma-376/infra/blob/develop/docs/adr/0054-dev-argocd-gitops-operations.md), 이미지 게시 의무는 [backend ADR 0075](https://github.com/soma-376/pulsemetry-backend/blob/develop/docs/adr/0075-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)를 따른다. 상대 레포 문서·구현이 모두 반영될 때까지 이 계약은 변경 중으로 둔다.
