# 계약 — dev generated values와 workflow 기반 GitOps 배포

| 항목 | 내용 |
|---|---|
| 당사자 | `pulsemetry-backend` 이미지·rollback publisher, `infra` 설정 publisher → private `infra-generated-values`; 계약은 `docs` 소유 |
| 사본 저장소 | `soma-376/infra-generated-values`, private, 배포 선택 기본 브랜치 `main` |
| 관련 결정 | [ADR 0034](../adr/0034-generated-values-private-sync-copy.md)의 인계 전 사본 원칙을 계승·부분 대체하는 [ADR 0035](../adr/0035-dev-gitops-delivery.md) |
| 상태 | 변경 중 — 인계 전 CD 사본 공급과 인계 후 workflow 게시를 분리; 구현·실제 GitOps 인수는 별도 검증 |

## 1. 소유권과 적용 범위

backend는 네 앱 이미지와 완료 manifest를 게시하고, **성공한 이미지 게시 workflow가 종료된 뒤 별도
승격 workflow**로 이미지 values 갱신과 이미지 rollback을 수행한다. infra는 AWS/CDK·chart·runtime 입력·
생성기·설정 검증 및 설정 publisher를 소유한다. 두 publisher는 아래 공통 Git 잠금과 같은 검증 규칙을
사용한다. 한 환경에 동시에 활성화되는 writer는 하나다. backend에 CDK·Helm·Kubernetes 쓰기 권한을
추가하지 않으며 실제 적용·상태 복구는 Argo CD가 맡는다.

Image Updater는 도입하지 않는다. `candidate/dev` branch, registry polling, Updater Git write-back,
`image.ref` 전환을 요구하지 않는다. 현재 `image.repository`와 bare `image.digest`를 사용하고 별도 image
overlay는 두지 않는다. mutable `dev` tag는 배포 후보 발견·승격의 입력이 아니며 Gateway를 포함한 tracking
tag 초기화를 GitOps 활성화 조건으로 삼지 않는다. 기존 tag의 존치·삭제는 이 문서 변경으로 실행하지 않는다.

private 저장소는 generated 사본·완료 manifest·배포 선택·공통 운영 상태를 보관한다. 사람이 generated를
임의 편집하거나 infra로 역동기화하지 않는다. 입력·생성기·chart 소유권은 infra에 남으며 backend의 허용
image 변경은 이 계약의 자동 게시·복구 경로에 한정된다. 대상은 dev EKS 앱·Kafka·ClickHouse·platform·
관측이다. Argo CD 자체는 수동 Helm 관리이며 prod, frontend ECS, 격리 perf는 범위 밖이다.

## 2. 식별자와 backend 완료 manifest

| 식별자 | 의미 |
|---|---|
| `C` | chart·생성기·추적 runtime 입력의 infra 전체 commit SHA |
| `B` | backend 완료 manifest 원본 UTF-8 바이트의 SHA-256 |
| `V` | 검증된 generated와 manifest 사본이 담긴 private 불변 snapshot commit SHA |
| `P` | 보호된 private `main`에서 C·V·B와 Application 참조를 함께 선택하는 desired commit SHA |

완료 manifest는 정확히 네 앱의 게시·검증이 모두 성공한 뒤 한 번 발행한다.

| 필드 | 요구사항 |
|---|---|
| `schemaVersion` | `1`; 미지원 버전은 거부 |
| `repository`, `backendCommit` | `soma-376/pulsemetry-backend`와 전체 backend commit SHA |
| `workflowRef`, `workflowRunId`, `workflowRunAttempt` | 네 이미지를 실제 만든 허용 **원본 게시 workflow**의 동일 실행·시도 |
| `status` | 네 앱 전체 완료를 나타내는 `complete` |
| `images` | `enrollment-api`, `telemetry-ingest`, `telemetry-gateway`, `dashboard-api`의 정확한 네 key; 각각 `repository`, `digest`, `arm64ManifestDigest`, `platform`, `deploymentContractSha256` |
| `deploymentContracts` | 동일한 네 key; 각각 이미지 계약의 원본 UTF-8 `content`와 `sha256` |
| `enrollmentAssets` | enrollment에 포함한 고정 CLI·GUI 릴리스/자산 lock과 checksum 검증 결과 |
| 검증 증거 참조 | 원본 실행 결과·build output digest·ARM64 manifest 검사 결과 |

각 repository는 해당 앱의 기존 `soma-376/` ECR repository로 제한한다. platform은 `linux/arm64`이고,
digest는 실제 게시한 root `sha256` digest이며 arm64ManifestDigest는 그 immutable root에서 확인한 ARM64 child digest다. 단일 ARM64 manifest면 두 값이 같다. 다시 조회한 mutable tag로 build output digest를 대체하지 않는다.
Gateway 누락·부분 성공·다른 SHA 또는 run/attempt 조합은 완료 묶음이 아니다. 같은 SHA 재실행에서 digest가
달라지면 새 B로 다시 검증한다. 동일 digest라도 다른 실행의 B이면 출처를 재검증하되 불필요한 Pod 재시작은
요구하지 않는다. manifest의 status만 믿지 않고 원본 게시 workflow 전체의 최종 성공을 확인한다.

B는 manifest 바이트 바깥에서 계산한다. 사본은 원본 바이트를 유지하며 다시 직렬화한 내용을 같은 B라고
표시하지 않는다. 승격·재실행·rollback workflow의 run/attempt를 원본 manifest에 덮어쓰지 않는다. 각
publisher 실행의 출처는 delivery state와 별도 검증 기록으로 연결한다. 별도 승격 workflow가 원본 실행의
종료를 확인하므로 같은 실행이 자기 성공을 기다리는 순환을 만들지 않는다.

### 2.1 생산 실행과 권한 있는 승격 실행의 경계

생산 완료 artifact는 원본 run/attempt마다 불변 이름으로 발행하고 90일 보존을 요청한다. 공개 artifact의
repository는 `soma-376/<app>`이고 AWS 계정·전체 ECR registry URL·ARN·내부 endpoint를 넣지 않는다.
승격기는 검증한 실행 계정과 현재 V의 앱별 repository를 대조해 private repository를 결정한다.
B에 각 이미지의 배포 계약 해시를 포함한다. 이미지에는 Flyway migration·envelope·고정 설치 자산의
배포 계약을 넣고, 신뢰한 검증기가 immutable digest로부터 그 작은 계약 파일을 읽는다. 후보 이미지를
인증정보가 있는 initContainer나 Job으로 실행하지 않는다.

`workflow_run: completed`의 workflow 파일과 `GITHUB_SHA`/`GITHUB_REF`는 default branch에 속한다.
이를 producer의 `develop` SHA로 해석하지 않는다. 승격 코드는 검토된 default-branch revision에서만
실행하고 event·GitHub API로 생산 workflow ID/path·repository/head repository·event·develop 이력·
SHA·run·attempt·최종 success와 정확한 artifact를 대조한다. artifact와 producer checkout의 스크립트를
승격 자격으로 실행하지 않는다. infra 공통 tooling도 desired C와 독립적인 검토된 전체 SHA로 고정한다.

생산의 기존 ECR push 역할과 승격의 `dev-gitops` Environment/OIDC 역할을 분리한다. default branch에
승격 workflow 정의가 있어야 이벤트를 받을 수 있다. 자동 스위치는 기본 비활성이며 수동 실행과 예약
재평가도 같은 provenance·잠금·fence 검사를 통과한다. schedule은 30분 간격의 후보 재평가이고 배포가
진행 중이면 새 owner로 덮지 않는다. repo concurrency는 `cancel-in-progress: false`다.

## 3. private 파일과 수정 경계

| 경로 | 의미와 writer |
|---|---|
| `dev/generated/*.yaml` | infra 생성 사본; backend는 아래 image·검증 metadata 변경만 허용, infra는 설정 재생성 |
| `dev/infra-commit.txt` | C 전체 SHA 한 줄; infra 설정 publisher만 변경 |
| `dev/bundle.json` | 원본 완료 manifest의 바이트 사본; backend image/rollback publisher가 선택 |
| `dev/desired.yaml` | `schemaVersion: 1`, `infraCommit: C`, `valuesCommit: V`, `bundleManifestSha256: B`, `operationNonce`; 잠금 소유 publisher만 변경 |
| `dev/applications/*.yaml` | C의 infra 템플릿에서 생성한 child Application; 잠금 소유 publisher가 desired와 함께 C/V 참조 변경 |
| `dev/root.yaml` | 최초 정상 B·운영 수락 뒤 생성하는 root Application 선언; infra 초기화 경로 소유 |
| `dev/activation.json` | 최초 인수의 운영 수락 및 검증 설정; image publisher는 변경하지 않음 |
| `dev/evidence/<operationNonce>.json` | 현재 owner가 게시하는 정제된 기능·호환성 검증 증거; 원문 로그·자격증명 제외 |
| `dev/delivery-state.json` | §4의 공통 잠금·진행·실패·재개 상태; 양 publisher가 동일 CAS 계약으로 변경 |
| 루트 `README.md`, `AGENTS.md` | 허용 파일·소유권과 이 계약 안내 |

`image.repository`는 허용 repository, `image.digest`는 `sha256:<64자리 소문자 16진수>`다. chart는 두 값을
`repository@digest`로 렌더한다. `image.ref`나 tag-only 입력을 함께 두지 않는다. initContainer와 앱의 같은
이미지 사용 및 ARM64 검증을 보존한다.

- **backend image/rollback:** 현재 선택된 V에서 시작한다. 네 앱의 digest와 원본 bundle·필수 provenance를
  갱신하고 C는 보존한다. 아래 검증 metadata 예외를 제외한 generated의 비이미지 바이트와 나머지 파일은 그대로 유지한다. repository는
  앱별 허용값과 일치해야 하며 임의 repository 교체를 이미지 갱신으로 취급하지 않는다. Application은
  새 V 참조만 바꾸고 C·비이미지 spec을 보존한다. 호환 C가 없거나 새 runtime이 필요하면 먼저 infra 설정
  경로로 조정하며 image publisher가 설정을 몰래 고치지 않는다.
- **infra config:** 고정 C의 생성기·추적 입력·검증한 외부 metadata로 전체 generated를 재생성한다. 현재
  선택된 B와 `dev/bundle.json` 원본 바이트 및 네 repository/digest를 보존한다. 최신 ECR tag나 임의 다른
  B를 선택하지 않는다. C와 Application 템플릿 변경은 이 경로에서 검증한다.
- 두 경로 모두 파일 허용목록·render·현재 schema/runtime/key/TLS·앱 계약 호환성을 검사한다. 준비 중 현재
  P/C/V/B가 바뀌면 최신 기준에서 다시 검증하며 서로 다른 실행의 YAML을 섞지 않는다.

검증 metadata의 예외는 네 앱의 `deliveryVerification.operationNonce`, `infraCommit`,
`bundleManifestSha256`, `imageDigest`, `contractSha256`, `contract`의 정확한 여섯 leaf다. 여기서
`infraCommit` 이하도 모두 `deliveryVerification` 아래이며 `contract`는 manifest의 공개 계약 원본 바이트를
JSON 문자열로 보존한다. 부모 객체 전체나 임의 하위 필드는 허용하지 않는다. 검증기 이미지·ServiceAccount·Secret
참조·endpoint·권한·timeout은 infra 설정이며 backend가 바꾸지 못한다. 같은 앱 digest를 유지해도 새 V와
rollback V마다 새 nonce로 네 gate를 모두 실행한다. nonce는 Job에만 반영하고 Deployment Pod template을
바꾸지 않는다. 그 외의 비이미지 바이트는 파싱 후 재직렬화하지 않고 보존한다.

`dev/evidence/`는 감사 로그나 원문 저장소가 아니다. 허용 schema의 결과·reason code·revision·Job UID·
검사 시각·비밀이 아닌 fingerprint와 해시만 저장한다. state는 증거 경로와 원본 바이트 해시를 연결한다.
root 증거는 uid·revision·startedAt·finishedAt만 선택적으로 추가하고 실제 resolved revision의 Git 이력에 P가 포함되는지 확인한다.
URL query·HTTP body·SQL 결과 원문·Prometheus 응답 전체·S3 object 본문·사용자 데이터·token은 저장하지
않는다. 공개 Actions artifact에는 private evidence와 generated values를 업로드하지 않는다.

Strimzi 기본 생성 목록은 `base.yaml`, `lbc.yaml`, `runtime.yaml`, `enrollment-api.yaml`,
`telemetry-ingest.yaml`, `telemetry-gateway.yaml`, `telemetry-kafka-admin.yaml`, `dashboard-api.yaml`,
`clickhouse.yaml`, `fluent-bit.yaml`, `ascp.yaml`, `observability.yaml`, `kafka-strimzi.yaml`의 13개다.
목록을 바꾸면 생성기·계약·사본 안내를 함께 바꾼다. telemetry-kafka-admin 사본은 Job 실행 요청이 아니다.
static chart·Loki/Tempo/otel-platform 설정과 lock은 infra에 남고 MSK 입력을 되살리지 않는다.

Secret 원문·token·개인키·비밀번호, kubeconfig, AWS 조회 원본, 로그·개인 경로를 Git에 넣지 않는다.
Kafka TLS 입력에는 Secret 이름과 UID/resourceVersion에서 유래한 metadata revision만 허용한다. 실제
ARN·endpoint는 private 사본 안에서만 다룬다. Git 자격 공급은 앱 Secret과 분리하며 Git·검증 기록에
원문을 남기지 않는다. 필드 허용목록 검증은 Git의 repository 단위 쓰기 권한을 필드 권한으로 바꾸지 않는다.

## 4. 공통 delivery state와 잠금

아래는 `dev/delivery-state.json`의 schemaVersion 1 계약이다. PROJ-311의 공통 구현과 별도 infra 설정 publisher는 이 계약을 함께 사용한다. 운영 저장소의 초기화·활성화는 §8의 수락 절차다.

| 필드 | 의미 |
|---|---|
| `schemaVersion` | `1`; 미지원·손상 상태는 자동 진행 거부 |
| `state` | `idle`, `deploying`, `rolling_back`, `blocked` 중 하나 |
| `generation` | 잠금 획득 또는 명시적인 운영자 소유권 인계마다 증가하는 단조 증가 정수 |
| `owner` | `repository`, `workflowRef`, `workflowRunId`, `workflowRunAttempt`; `idle`에서는 null |
| `expectedDesiredP` | 작업이 비교해야 할 현재 desired P; 실제 선택이 바뀌면 소유자가 CAS로 갱신 |
| `operation.kind` | `image`, `config`, `rollback`; idle에서는 operation이 null |
| `currentSelection` | 현재 `infraCommit`, `valuesCommit`, `bundleManifestSha256`, `desiredCommit`, `operationNonce`; desiredCommit은 expectedDesiredP와 같음 |
| `lastKnownGood` | 기능 검증이 끝난 currentSelection 필드와 `evidenceSha256`, `configurationFingerprint`, `fingerprints` |
| `failedReleases` | 누적 실패의 `bundleManifestSha256`, `backendCommit`, `imageVectorSha256`, `failureRecordedAt`, 신규 기록의 `fenceSealed`; 전체 Git 이력과 대조 |
| `failedRelease` | 최신 failedReleases 원소의 선택적 요약; 누적 실패 기록을 대체하지 않음 |
| `resumeFence` | `active`, `failedBackendCommit`, `recordedAt`; 실패 뒤 자동 이미지 승격의 재개 경계 |

이미지 벡터는 `enrollment-api`, `telemetry-ingest`, `telemetry-gateway`, `dashboard-api` 순서의
`<app>=<repository>@<digest>\n`을 연결한 UTF-8 바이트의 SHA-256이다. JSON key 순서는 영향을 주지 않는다.
`fingerprints`는 `schemaFingerprint`, `runtimeFingerprint`, `keyTlsFingerprint`, `envelopeFingerprint`의
SHA-256을 포함한다. `configurationFingerprint`는 허용 image·검증 metadata를 제외한 generated 13개 바이트의 지문으로, 실제 workload에서 읽은 runtimeFingerprint와 구분한다. 원문 secret 대신 현재 버전·metadata와 계약을 검증한 결과를 해시한다.
platform·source 증거는 B와 대조한다. 실패 B와 **네 앱 repository/digest 전체 벡터**를 모두 거부 목록에
남긴다. manifest 재포장이나 같은 이미지의 다른 run/attempt로 B만 바꿔도 실패 조합을 다시 승인하지 않는다.
publisher는 최신 failedRelease와 보존된 Git 상태 파일의 실패 이력에서 모든 실패 B·벡터를 조회해 누적
거부 목록을 구성한다. 새 실패로 최신 객체가 바뀌어도 과거 거부를 지우지 않는다. shallow history나 조회
권한 부족으로 전체 거부 이력을 확인할 수 없으면 승격을 거부한다. Git 실패 이력의 보존·조회 권한과
누적 목록 재구성은 구현 수락 검증에 포함한다.

잠금은 두 저장소에 공통인 private `main`에서 획득한다. publisher는 최신 main HEAD·현재 desired P와
`idle` 상태를 읽고 owner·증가한 generation·예상 P·작업 종류를 기록한 commit을 **expected main HEAD의
CAS와 정상 fast-forward**로 게시한다. HEAD가 바뀌면 실패하며 최신 상태를 다시 읽는다. 저장소별 GitHub
`concurrency`는 이 공통 잠금을 대체하지 않는다. force push나 마지막 writer 우선 덮어쓰기는 금지한다.

잠금은 V/P 게시 이후에도 배포·기능 확인·필요한 rollback이 끝날 때까지 유지한다. V·P·상태·유지보수
선택 등 모든 변경 직전에 원격 owner와 generation, expected P 및 main HEAD를 다시 비교한다. 소유권이
바뀌면 이전 실행은 쓰기를 중단한다. state commit이나 V append로 main HEAD가 전진한 사실을 P 변경과
혼동하지 않는다. P는 desired와 Application 선택을 함께 바꾼 commit이고 상태 기록 commit은 P가 아니다.

P는 자신의 SHA를 포함할 수 없으므로 P 게시 직후에는 state의 expectedDesiredP가 이전 P를 가리킬 수
있다. 다음 state 갱신에만 제한된 전환을 허용한다: 같은 owner/generation이 기존 expectedDesiredP에서
자신이 게시한 정확한 P임을 실행 근거·desired·Application의 C/V/B로 확인하고, 최신 main HEAD CAS로
expectedDesiredP를 새 P로 바꾼다. 다른 실행이 만든 선택이나 임의 최신 P를 승인하는 예외가 아니다.
이 구간의 중단·응답 유실은 원격 Git과 실행 근거로 확인하며 입증할 수 없으면 blocked로 인계한다.
다음 image/config 게시나 잠금 해제 전에 이 전환이 완료돼야 한다.

`blocked`에서는 자동 승격과 자동 잠금 인계를 금지한다. 시간 경과·runner 종료 추정만으로 만료 잠금을
탈취하지 않는다. 운영자가 원래 workflow 종료, 진행 중 Argo operation 부재, 실제 실행 revision과 기능을
확인한 뒤 근거를 남기고 CAS로 generation을 증가시켜 소유권을 인계·복구한다. 잠금 유실 때문에 blocked를
쓸 권한도 없어진 이전 owner는 상태를 덮어쓰지 않고 오류를 보고한다. 상태를 보유한 경로에서 차단 상태를
확정하고 수동 확인한다. 정상 publisher는 unknown·손상·부재 상태를 idle로 가정하지 않는다.

## 5. 게시와 배포 선택

1. backend는 원본 게시 workflow의 최종 성공, B와 네 build digest·ARM64를 검증한 뒤 별도 image 승격을
   시작한다. infra는 승인된 설정 변경을 고정 C와 현재 B로 준비한다. 미리 준비한 결과는 잠금 획득 뒤
   현재 P/C/V/B에 다시 대조한다. 오래된 image 실행이 더 최신 desired를 덮어쓰지 못하게 한다.
2. §4의 공통 잠금을 획득하고 `deploying`으로 기록한다. 현재 lastKnownGood을 작업 시작 복구 기준으로
   고정한다. image 승격은 실패 이력·resumeFence가 있으면 §7의 재개 조건도 만족해야 한다. config
   게시에는 새 image의 strict-descendant/run 시각 조건을 적용하지 않고 현재 B를 보존한다. config
   경로로 blocked 상태를 우회하거나 실패 이력·resumeFence를 해제해서는 안 된다.
3. 허용된 변경·호환성·렌더 검증이 끝난 generated와 원본 bundle을 snapshot V로 먼저 append한다.
   이때 desired와 Application은 기존 V를 가리킨다. V 자체에 자신의 commit SHA를 쓰지 않는다.
4. owner/generation과 expected main HEAD·현재 P를 다시 확인한 뒤 별도 P commit에서 desired와 영향을
   받는 모든 child Application C/V 참조를 함께 변경한다. desired 파일에 P 자신의 SHA를 쓰지 않는다.
5. Argo root는 protected main의 `dev/applications/`를 읽고 child는 C/V full SHA를 고정한다.
   선택 이후에도 잠금을 보유하고 §6의 현재 목표 revision·health·기능 결과를 확인한다.
6. 정상 완료 때 lastKnownGood을 검증한 C/V/B/P로 갱신하고 idle로 잠금을 해제한다. 실패/불명확한 결과는
   §7을 따른다. V·이미지·원본 manifest와 검증 근거는 rollback 가능 기간 동안 보존한다.

V 이후 P 이전 실패는 기존 desired를 유지한다. 게시 응답 유실은 원격 main과 선택을 읽어 실제 성공 여부를
확인하고 중복 선택을 만들지 않는다. 배포 여부가 불명확하면 자동 rollback하지 않고 blocked로 둔다.
정상 image 후보는 검증 후 자동 승격하며 매번 수동 승인을 추가하지 않는다. config·key·schema 변경은
각 소유 계약의 명시적인 검토·운영 절차를 유지한다.

## 6. 적용 순서와 성공 판정

인계 후 Application은 automated sync·self-heal을 사용하고 prune·allowEmpty는 끈다. 기존 releaseName·
namespace·resource identity·PVC를 보존하며 같은 자원을 Helm과 Argo가 상시 동시에 쓰지 않는다.
platform·controller·runtime·데이터 준비 뒤 enrollment의 Flyway 성공 → ingest → Gateway → dashboard
순서로 적용한다. parent sync wave와 child custom health는 목표 C/V의 실제 resolved revision,
`Synced`·`Healthy`, 진행 중/실패 operation 부재와 migration 성공을 함께 검사한다. 이전 revision의
Healthy로 다음 앱을 열지 않는다. Git 선택의 원자성이 네 앱의 동시 교체를 뜻하지 않는다.

Strimzi Operator Application은 같은 C의 vendor·checksum·Helm·post-render를 보존하는 CMP로 렌더하며
Operator Ready·CRD Established 뒤 Kafka CR의 Ready·observedGeneration을 확인한다. Strimzi가 만든
Pod·Service·PVC·인증서는 중복 관리하지 않는다. offset reset Job을 일반 sync hook으로 실행하지 않는다.

성공 근거는 실제 P/C/V/B, 네 repository/digest와 ARM64 manifest에 대응하는 Pod imageID, 현재 revision의
Synced·Healthy·migration, 필요한 target/service 연결, enrollment·dashboard 핵심 기능과
Gateway ACK → Kafka consume/commit → S3/ClickHouse 적재를 포함한다. index digest와 ARM64 child
manifest digest에 단순 문자열 일치를 요구하지 않는다. Git 게시, Synced, Healthy와 기능 검증은 별도
상태로 기록하고 기존 PVC·offset·인증서·Secret 공급 및 관측 경로 보존을 확인한다.

### 6.1 PostSync 기능 gate와 관측 판정

활성화한 root와 18개 child는 `syncPolicy.retry.limit: 0`을 명시해 같은 게시 operation의 자동 재시도와
PostSync 중복 검증을 막는다. 재검증은 새 nonce를 가진 다음 게시나 명시적 운영 절차로 수행한다.
각 앱의 fixed-name PostSync Job은 `BeforeHookCreation`, `backoffLimit: 0`을 사용한다. 성공·실패 직후
삭제하거나 TTL로 증거를 지우지 않는다. Job은 자신의 Application operation 완료를 기다리지 않고
workload·DB·실제 기능만 확인한다. Job 종료 뒤 runner와 parent custom health가 현재 operation의
성공과 C/V를 결합한다. selective sync와 `ApplyOutOfSyncOnly`로 hook을 건너뛰지 않는다.

| 단계 | 필요한 기능 증거 |
|---|---|
| enrollment | `enrollment.flyway_schema_history`와 `telemetry_ops.flyway_schema_history`의 성공·checksum, 설치 bootstrap 자산, 전용 installation PIT의 telemetry token 발급 |
| ingest | 목표 workload와 저장 경로 준비, 실제 schema·envelope·TLS 계약 |
| Gateway | 고유 OTLP 요청 한 번의 ACK, ClickHouse의 기대 관측 한 행, 그 행의 archive reference에 대응하는 S3 metadata, Kafka commit watermark |
| dashboard | `dashboard_cache.flyway_schema_history`, 전용 사용자의 실행마다 새 OIDC 로그인과 실제 대표 조회 |

전용 synthetic tenant/member/installation은 운영 준비 단계에서 따로 만든다. 서명키·자체 발행 user JWT·
DB session INSERT·반복 사용하는 refresh token으로 로그인 검증을 대체하지 않는다. 발급받은 AT/RT와
telemetry JWT는 메모리에서만 사용한다. ACK 응답이 불명확하면 동일 요청을 재송신하지 않는다.

Kafka는 기존 Prometheus의 신선한 exporter UP·topic/group·전체 partition 집합을 검증한다. ACK 뒤
캡처한 모든 partition의 end offset 벡터를 이후의 신선한 committed offset 벡터가 이상인지 확인한다.
lag 0, consume 로그, commit 이전 처리 로그만으로 성공을 선언하지 않는다. 누락 partition·미commit·
오래된 sample·미도달 watermark는 성공 증거가 없다. S3는 기대 archive key를 정확한 Prefix로 하는
`ListObjectsV2(MaxKeys=1)`의 Key·Size·LastModified를 확인하고 object 본문을 읽지 않는다.

Job 결과는 4KiB 미만의 고정 schema termination message다. runner는 nonce·C·B·앱 digest·contract hash,
실제 Job UID·Pod imageID·현재 Argo operation·V/P를 함께 결합한다. V/P를 V 안에 써서 자기 참조를
만들지 않는다. 결측·잘림·예상하지 않은 exit·deadline·IdP 또는 조회 장애는 `unknown`이며 Job `Failed`
자체를 이미지의 확정 실패로 바꾸지 않는다.

key/TLS 검증은 현재 CSI mount와 다음 Pod가 선택할 버전을 함께 대조한다. verifier와 runner DB reader는
허용한 앱 Secret 원본 ARN의 `DescribeSecret` metadata로 SPC의 고정 `objectVersion` 또는 미고정 입력의
`AWSCURRENT` VersionId를 확인한다. 선택한 VersionId와 현재 CSI mount의 버전이 모두 같아야 하며 선택
metadata도 fingerprint에 포함한다. `AWSCURRENT`가 이동했지만 기존 Pod가 아직 이전 키를 mount한 상태는
`unknown`으로 보류한다. 앱 Secret 원문의 GetSecretValue 권한이나 조회는 추가하지 않는다.

## 7. 실패·자동 rollback·재개

backend image 배포에서 **현재 선택한 목표 revision의 실패가 확정**되고 진행 중 Argo operation이 없을
때만, 같은 owner/generation이 자동 rollback 후보를 검토한다. timeout·연결 끊김·이전 revision의 실패·
아직 실행 중인 operation은 확정 실패가 아니다. 먼저 잠금을 유지한 채 실패 B·전체 이미지 벡터·source와
failure fence를 공통 state에 CAS로 영속화한다. 아래 현재 호환성 검사가 통과한 뒤에만 `rolling_back`으로
전환하고 자동 rollback을 한 번 시도한다. 근거가 없거나 비호환이면 rollback을 시작하지 않고 blocked다.
rollback workflow가 producer B를 새로 만들지 않는다.

PROJ-311의 자동 복구는 마지막 정상 증거와 실제 schema·runtime·key/TLS가 **모두 같은 경우**만 지원한다. 변경된 schema에 대한 하위 호환성 추정으로 복구하지 않는다.

대상은 배포 직전 lastKnownGood B다. **이미 적용된 migration을 포함한 실제 현재 schema**, 현재 C/runtime,
Kafka backlog·envelope, key/TLS와 이전 이미지의 호환성을 다시 검증한다. 이전 성공 기록만으로 호환성을
추정하지 않는다. 호환되면 현재 비이미지 바이트와 C를 보존한 새 V/P를 만들고 같은 순서·현재 revision·
기능 검증을 수행한다. 일반 image rollback에서는 Argo automated sync·self-heal을 유지하여 새 Git 선택을
적용한다. Argo 이력 rollback이나 `helm rollback`을 호출하지 않는다.

다음은 자동 복구하지 않고 `blocked`로 둔다: 상태/결과 unknown, infra config 배포 실패, 호환성 근거 없음,
lastKnownGood 없음, 소유권 유실, rollback 실패 또는 두 번째 rollback 필요. 소유권 유실 시 이전 owner는
추가 Git 쓰기를 하지 않는다. 자동 DB downgrade·PVC 복원·Kafka offset reset·MSK 복구는 수행하지 않는다.
과거 C/V/B 전체 복원은 운영자가 실제 데이터 호환성과 범위를 별도로 검증한 수동 복구다.

rollback의 현재 revision·기능 검증까지 성공하면 복구된 조합을 lastKnownGood으로 남기고 잠금을 해제한다.
실패 B·전체 이미지 벡터 거부와 `resumeFence`는 보존한다. 자동 이미지 승격은 다음을 **모두** 만족할 때만
재개한다.

- 원본 backend commit은 실패 source의 **strict descendant**이며 허용된 현재 `develop` 이력에 속한다.
- **원본 이미지 게시 workflow run의 생성 시각**이 영속화한 failure fence의 `recordedAt`보다 늦다.
  재시도/승격 workflow 시작 시각으로 대신하지 않는다. 실패 전에 생성된 run의 늦은 완료·재실행은 제외한다.
- 원본 게시 실행·시도가 성공했고 B·ARM64·현재 C/runtime 호환성을 새로 검증했다.
- 실패 B 또는 실패한 canonical 네 이미지 벡터와 일치하지 않는다. 새 manifest로 재포장해도 거부한다.

신규 failure fence는 두 CAS로 확정한다. 먼저 `fenceSealed: false`인 실패 B·이미지 벡터를 영속화한다.
그 게시 성공 응답을 확인한 뒤의 시각을 failureRecordedAt/recordedAt으로 삼아 `fenceSealed: true`를
별도 CAS로 기록한다. 두 번째 CAS가 없거나 unsealed 실패 이력만 있으면 자동 재개하지 않는다.
전체 Git 이력에서 같은 실패 B/벡터의 가장 늦은 시각을 선택하며 오래된 기록으로 fence를 낮추지 않는다.
기존 schemaVersion 1 기록의 누락 필드와 구분하되 새 실패를 unsealed 상태로 확정된 것으로 간주하지 않는다.
fence commit ID는 Git 이력에서 얻으며 자기 SHA를 같은 파일에 쓰지 않는다. 잠금 해제 시 이미 완료된 적격 후보도 다시 조회해 재개하며 새 webhook만 기다리지
않는다. 최신성·실패 이력·fence 검증은 실제 잠금 획득·P 게시 전에 반복한다.
P를 바꾸기 전 더 새로운 성공 생산 run을 발견한 후보는 `superseded`다. 현재 owner·HEAD·기준 P를
다시 확인해 잠금을 idle로 해제하고 최신 후보를 재평가한다. V가 이미 append됐어도 선택되지 않은
snapshot으로 남으며 기존 P를 바꾸지 않는다. 이 정상적인 후보 교체와 게시·관측 실패는 구분한다. 새 정상 배포 뒤에도 실패 기록을
지우지 않는다. 수동 rollback·차단 해제는 같은 잠금·호환성·검증 경로를 사용하고 자동 재개 경계를 임의로
우회하지 않는다.

## 8. 유지보수·최초 인수·현재 상태

offset reset, key/인증서 교체와 수동 데이터 작업은 일반 image rollback과 다르다. 공통 잠금과 자동 승격
보류를 먼저 확보하고 해당 parent/child의 auto sync·self-heal을 필요한 범위에서 중지한다. parent가 child의
중지를 복구하지 않도록 선언도 맞춘다. 대상·사유·현재 P/C/V/B·복귀값과 실제 종료를 확인한 뒤 운영하고,
Git 정합·기능 확인 후 재개한다. key verifier→signer 순서와 인증서 metadata 기반 명시적 재배포는 유지한다.
prune off는 데이터 보존 보장의 대체가 아니며 Namespace·CRD·Kafka CR·PVC 삭제는 별도 운영 절차다.

### 인계 전 CD 사본 공급

PROJ-279는 성공한 infra CDK 배포 뒤 **Strimzi 13개 YAML과 `dev/infra-commit.txt`만** private main에
공급한다. 최초 실제 사본도 CD가 생성·검증·게시하며 로컬에서 실제 dev 값을 생성·수동 게시하지 않는다.
generated 직접 편집·역동기화는 금지하고 사본 commit과 대응 C의 chart·정적 values를 함께 사용한다.
적용은 기존 수동 Helm 절차이며 CDK 성공·사본 게시·Helm 적용·기능 검증을 별도 상태로 기록한다.

plan 시작 시 최신 성공 backend `develop` 원본 run의 정확한 attempt를 선택하고 네 ARM64 build digest·
backend SHA·로그 해시를 승인 artifact에 고정한다. 누락·중복·혼합·잘못된 digest/platform·형식 변경·
만료된 로그는 거부하며 mutable tag나 다른 run으로 대체하지 않는다. 이 정제된 로그 입력은 완료 manifest
B가 아니고 정식 GitOps 승격 증거도 아니다. backend manifest 전환은 PROJ-311의 정식 이미지 공급 경로다.

승인 plan에는 사본 활성 여부도 포함한다. 승인 후 Variable 변경으로 게시를 활성화하지 않는다.
CDK 성공 후 같은 C의 생성기·추적 runtime으로 Outputs와 Strimzi metadata를 새로 조회해 전체 사본을
생성·Helm 렌더 검증한다. 전용 EKS reader는 기존 infra deploy 역할만 AssumeRole하고, 지정 Secret 5개
`get` RBAC와 기존 SSM relay·TLS hostname·CA 검증을 사용한다. metadata-only 응답을 요청해 이름·
namespace·UID·resourceVersion만 보존하며 전체 Secret 응답으로 fallback하지 않는다. RBAC의 `get`은
필드별 권한이 아니므로 응답 형식도 검증한다. 원문·kubeconfig·실제 endpoint는 추가 공개 artifact에 넣지 않는다.

private main 시작 HEAD에서 14개 파일만 한 commit으로 정상 push한다. 같은 내용은 no-op이며 HEAD
경쟁 또는 게시 직전 metadata 변경은 중단한다. force push·자동 rebase는 하지 않는다. 게시 응답 유실은
원격 commit·내용을 확인해 성공 여부를 판정하고 중복 게시하지 않는다. 자동 게시 제목은
`auto(dev): values 동기화 by infra@<SHA 12자리> (run <ID>/<attempt>)`이며 원본 infra·backend 출처를 남긴다.

성공 deploy receipt는 원본 infra SHA·run·attempt, plan artifact ID/hash와 배포 후 baseline 해시를 보존한다.
사본 job은 기존 배포 Environment·workflow concurrency 안에서 원본 plan·deploy job 성공과 artifact 무결성·
현재 AWS baseline을 검증한다. publisher 재시도 attempt와 원본 승인·배포 attempt는 분리한다. 전체 run이
사본 실패로 실패했더라도 원본 plan·deploy 성공 증거가 유효하면 사본만 재시도한다. 고정 이미지는 유지하고
Outputs·metadata만 재수집하며 증거 만료·누락·baseline 변경은 새 plan을 요구한다. CDK deploy-only
재실행의 기존 attempt 차단을 완화하지 않는다.

초기에는 `GENERATED_VALUES_SYNC_ENABLED`를 비활성으로 두고 IAM·RBAC·GitHub App 자격을 준비한 뒤
실제 최초 게시·no-op·실패 시 기존 사본 보존을 검증한다. `dev/bundle.json`, `dev/desired.yaml`,
`dev/applications/`, `dev/delivery-state.json` 중 하나라도 있으면 이 사본 공급을 중단한다.

### 최초 GitOps 인수

Argo 인계 전에 CD 사본 writer를 먼저 중지한다. 완료 manifest 도입 전의 기존 이미지는 같은 SHA/run을
꾸며 B로 만들지 않는다. 사본의 이미지가 실제 실행 중 이미지라고 가정하지 않고 고정 C와 실제 실행 중
이미지의 handoff snapshot H를 별도 검증해 수동 인수한다. 검증한 새 B·V/P·공통 state가 준비된 뒤
정식 자동화를 활성화하며 인계 전 사본 writer와 동시에 운영하지 않는다.

`dev/activation.json`은 PROJ-310의 실제 수동 인수, 별도 infra 설정 publisher/CAS 수락, contract를 포함한
최초 정상 B의 기능 검증을 별도로 증명한다. 이 세 조건과 검증기 ARM64 image digest·전용 조회 권한·
synthetic identity·Git 보호 설정을 준비하기 전에는 자동화를 켜지 않는다. PROJ-310의 H 생성기는 root·
bundle·desired·delivery state를 만들지 않는다. 별도 초기화 경로에서 정상 B와 증거를 검증한 뒤에만
처음 V/P·공통 상태와 root 연결을 준비한다. 검토 출력의 root는 운영자가 보호된 context에서 먼저 수동 적용하고 publisher는 Kubernetes에 쓰지 않는다. 초기 검증이 blocked이면 같은 owner/generation·현재 P/V의 재관측으로만 수락을 재시도하며 V/P를 다시 게시하지 않는다. 최초 게시도 원본 성공 run/attempt와 artifact ID/archive hash·manifest 원본 바이트·immutable 이미지의 packaged contract를 신뢰한 코드로 재검증한다. 운영 수락 hash만으로 출처를 인증하지 않는다. `activation.json`의 true 값을 직접 넣는 것만으로 운영 수락을
대체하지 않는다. backend image publisher는 이 파일을 수정하거나 비활성을 자동 해제하지 않는다.

PROJ-280·281·286의 설계를 PROJ-311에서 완료 manifest·별도 image 승격·공통 CAS·기능 검증 경계로 구체화한다. 별도 infra 설정 publisher와 실제 권한·branch 보호·Application 인수는 독립 수락 대상이다.
기존 Strimzi·관측 배포 완료를 이 자동화의 완료 증거로 사용하지 않는다. PROJ-276의 사본 범위를 계승하며,
현재 구현 상태와 실제 운영 증거는 각 소유 레포에서 확인한다.

구현·운영은 [infra ADR 0056](https://github.com/soma-376/infra/blob/feature/PROJ-311-gitops-delivery-support/docs/adr/0056-dev-argocd-gitops-operations.md),
backend 공급·승격·rollback은 [backend ADR 0079](https://github.com/soma-376/pulsemetry-backend/blob/feature/PROJ-311-backend-image-promotion/docs/adr/0079-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)을 따른다.
상대 레포 문서·구현이 모두 반영될 때까지 이 계약은 변경 중이다.

## 9. 수락 사례

| 사례 | 요구 결과 |
|---|---|
| Gateway 누락·한 이미지 실패·다른 SHA/run/attempt 혼합 | 완료 B·승격 없음, 현재 P 유지 |
| 원본 workflow 미종료·실패이나 manifest complete | 별도 승격 거부; 자기 실행 성공 대기 금지 |
| 동일 SHA 재실행·동일 digest 재게시 | 새 B 출처 검증; 불필요한 Pod 재시작 없음 |
| image 게시에서 비이미지 바이트·C·repository 변경 | 거부; infra 설정 경로로 인계 |
| infra 재생성이 선택 B·원본 bundle·네 digest 변경 | 거부 |
| 성공 rollback 뒤 fence가 남은 idle에서 config 게시 | 현재 B·거부 이력·fence 보존; image 전용 descendant/run 조건으로 config를 막지 않음 |
| backend·infra 동시 CAS 잠금 시도 | 한 owner/generation만 획득; 나머지는 최신 상태 재검증 |
| stale main HEAD·P 변경·옛 owner의 늦은 쓰기 | 거부; force push·마지막 쓰기 우선 금지 |
| V 이후 P 실패·응답 유실·state commit만 전진 | 실제 P 조회; 이전 선택 보존 또는 확정 결과 기록, state를 P로 오인하지 않음 |
| 잠금 장기 보유·runner 소실·schema 손상 | 자동 탈취 없음; blocked와 운영자 확인 |
| 현재 revision migration/기능 실패, operation 종료, 호환 정상 B 존재 | 같은 owner가 단 한 번 새 V/P image rollback |
| migration 후 구 이미지 호환성 부재·정상 B 부재 | 자동 rollback 없음, blocked |
| config 실패·결과 unknown·operation 진행 중 | 자동 image rollback 없음, blocked 또는 진행 확인 후 판정 |
| rollback 실패·owner 유실 | 추가 시도·옛 owner 쓰기 없음, blocked 및 수동 확인 |
| 일반 image rollback | Argo auto sync/self-heal 유지, 현재 비이미지 보존, DB/PVC/offset 역연산 없음 |
| 성공 rollback 뒤 실패 B 재게시·같은 벡터 재포장 | 현재·과거 Git 실패 이력을 대조해 거부; 이력 조회 불가도 승격 거부 |
| 성공 rollback 뒤 실패 전 생성 run의 늦은 완료·재시도 | strict descendant 여부와 별개로 fence에서 거부 |
| 성공 rollback 뒤 fence 이후 생성한 원본 run·strict descendant·정상 새 벡터 | 재검증 후 자동 재개; 잠금 해제 때 이미 완료된 적격 후보도 조회 |
| offset reset·key/인증서 유지보수 | 공통 잠금·승격 보류와 parent/child reconcile 중지; 기능 확인 후 재개 |
| 기존 Helm 인수·인계 후 사본 writer 실행 | resource/PVC/offset/Secret 보존; 자동화 metadata 존재 시 사본 공급 거부 |
| 승인 뒤 활성 Variable 변경·잘못된 로그 입력 | 승인 입력 유지; 게시 활성화·이미지 대체 거부 |
| 사본 실패 뒤 유효한 원본 plan·deploy receipt | 고정 이미지·현재 baseline 검증 후 사본만 재시도; CDK attempt 차단 유지 |
| metadata-only 실패·metadata 변경·private HEAD 경쟁 | 전체 Secret fallback·force push 없이 게시 중단; 기존 사본 보존 |
| 최초 사본·동일 내용 재실행·게시 응답 유실 | CD 최초 14개 파일 게시·no-op·원격 성공 확인으로 중복 방지 |
