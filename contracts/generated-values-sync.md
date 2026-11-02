# 계약 — dev generated values와 workflow 기반 GitOps 배포

| 항목 | 내용 |
|---|---|
| 당사자 | `pulsemetry-backend` 이미지·rollback publisher, `infra` 설정 publisher → private `infra-generated-values`; 계약은 `docs` 소유 |
| 사본 저장소 | `soma-376/infra-generated-values`, private, 배포 선택 기본 브랜치 `main` |
| 관련 결정 | [ADR 0030](../adr/0030-generated-values-private-sync-copy.md)의 인계 전 사본 원칙을 계승·부분 대체하는 [ADR 0031](../adr/0031-dev-gitops-delivery.md) |
| 상태 | 변경 중 — Image Updater 도입 철회와 workflow 게시 설계 Accepted; 구현·실제 GitOps 인수는 별도 검증 |

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
| `images` | `enrollment-api`, `telemetry-ingest`, `telemetry-gateway`, `dashboard-api`의 정확한 네 key; 각각 `repository`, `digest`, `platform` |
| `enrollmentAssets` | enrollment에 포함한 고정 CLI·GUI 릴리스/자산 lock과 checksum 검증 결과 |
| 검증 증거 참조 | 원본 실행 결과·build output digest·ARM64 manifest 검사 결과 |

각 repository는 해당 앱의 기존 `soma-376/` ECR repository로 제한한다. platform은 `linux/arm64`이고,
digest는 실제 게시한 `sha256` digest다. 다시 조회한 mutable tag로 build output digest를 대체하지 않는다.
Gateway 누락·부분 성공·다른 SHA 또는 run/attempt 조합은 완료 묶음이 아니다. 같은 SHA 재실행에서 digest가
달라지면 새 B로 다시 검증한다. 동일 digest라도 다른 실행의 B이면 출처를 재검증하되 불필요한 Pod 재시작은
요구하지 않는다. manifest의 status만 믿지 않고 원본 게시 workflow 전체의 최종 성공을 확인한다.

B는 manifest 바이트 바깥에서 계산한다. 사본은 원본 바이트를 유지하며 다시 직렬화한 내용을 같은 B라고
표시하지 않는다. 승격·재실행·rollback workflow의 run/attempt를 원본 manifest에 덮어쓰지 않는다. 각
publisher 실행의 출처는 delivery state와 별도 검증 기록으로 연결한다. 별도 승격 workflow가 원본 실행의
종료를 확인하므로 같은 실행이 자기 성공을 기다리는 순환을 만들지 않는다.

## 3. private 파일과 수정 경계

| 경로 | 의미와 writer |
|---|---|
| `dev/generated/*.yaml` | infra 생성 사본; backend는 아래 image 변경만 허용, infra는 설정 재생성 |
| `dev/infra-commit.txt` | C 전체 SHA 한 줄; infra 설정 publisher만 변경 |
| `dev/bundle.json` | 원본 완료 manifest의 바이트 사본; backend image/rollback publisher가 선택 |
| `dev/desired.yaml` | `schemaVersion: 1`, `infraCommit: C`, `valuesCommit: V`, `bundleManifestSha256: B`; 잠금 소유 publisher만 변경 |
| `dev/applications/*.yaml` | C의 infra 템플릿에서 생성한 child Application; 잠금 소유 publisher가 desired와 함께 C/V 참조 변경 |
| `dev/delivery-state.json` | §4의 공통 잠금·진행·실패·재개 상태; 양 publisher가 동일 CAS 계약으로 변경 |
| 루트 `README.md`, `AGENTS.md` | 허용 파일·소유권과 이 계약 안내 |

`image.repository`는 허용 repository, `image.digest`는 `sha256:<64자리 소문자 16진수>`다. chart는 두 값을
`repository@digest`로 렌더한다. `image.ref`나 tag-only 입력을 함께 두지 않는다. initContainer와 앱의 같은
이미지 사용 및 ARM64 검증을 보존한다.

- **backend image/rollback:** 현재 선택된 V에서 시작한다. 네 앱의 digest와 원본 bundle·필수 provenance를
  갱신하고 C는 보존한다. generated의 비이미지 바이트와 나머지 파일은 그대로 유지한다. repository는
  앱별 허용값과 일치해야 하며 임의 repository 교체를 이미지 갱신으로 취급하지 않는다. Application은
  새 V 참조만 바꾸고 C·비이미지 spec을 보존한다. 호환 C가 없거나 새 runtime이 필요하면 먼저 infra 설정
  경로로 조정하며 image publisher가 설정을 몰래 고치지 않는다.
- **infra config:** 고정 C의 생성기·추적 입력·검증한 외부 metadata로 전체 generated를 재생성한다. 현재
  선택된 B와 `dev/bundle.json` 원본 바이트 및 네 repository/digest를 보존한다. 최신 ECR tag나 임의 다른
  B를 선택하지 않는다. C와 Application 템플릿 변경은 이 경로에서 검증한다.
- 두 경로 모두 파일 허용목록·render·현재 schema/runtime/key/TLS·앱 계약 호환성을 검사한다. 준비 중 현재
  P/C/V/B가 바뀌면 최신 기준에서 다시 검증하며 서로 다른 실행의 YAML을 섞지 않는다.

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

아래는 `dev/delivery-state.json`의 **문서상 schemaVersion 1 계약**이다. 이번 문서 개정은 실제 JSON 파일,
workflow, 잠금 또는 복구 자동화를 만들지 않는다.

| 필드 | 의미 |
|---|---|
| `schemaVersion` | `1`; 미지원·손상 상태는 자동 진행 거부 |
| `state` | `idle`, `deploying`, `rolling_back`, `blocked` 중 하나 |
| `generation` | 잠금 획득 또는 명시적인 운영자 소유권 인계마다 증가하는 단조 증가 정수 |
| `owner` | `repository`, `workflowRef`, `workflowRunId`, `workflowRunAttempt`; `idle`에서는 null |
| `expectedDesiredP` | 작업이 비교해야 할 현재 desired P; 실제 선택이 바뀌면 소유자가 CAS로 갱신 |
| `operation.kind` | `image`, `config`, `rollback`; idle에서는 operation이 null |
| `lastKnownGood` | 기능 검증이 끝난 `infraCommit`, `valuesCommit`, `bundleManifestSha256`, `desiredCommit` |
| `failedRelease` | 최신 실패의 `bundleManifestSha256`, `backendCommit`, app-keyed `images` repository/digest 벡터, `failureRecordedAt`; 이전 기록은 Git 실패 이력에서 보존 |
| `resumeFence` | `active`, `failedBackendCommit`, `recordedAt`; 실패 뒤 자동 이미지 승격의 재개 경계 |

images 벡터는 정확히 네 앱 key를 고정하고 key 순서에 의존하지 않는 canonical 표현으로 비교한다.
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

## 7. 실패·자동 rollback·재개

backend image 배포에서 **현재 선택한 목표 revision의 실패가 확정**되고 진행 중 Argo operation이 없을
때만, 같은 owner/generation이 자동 rollback 후보를 검토한다. timeout·연결 끊김·이전 revision의 실패·
아직 실행 중인 operation은 확정 실패가 아니다. 먼저 잠금을 유지한 채 실패 B·전체 이미지 벡터·source와
failure fence를 공통 state에 CAS로 영속화한다. 아래 현재 호환성 검사가 통과한 뒤에만 `rolling_back`으로
전환하고 자동 rollback을 한 번 시도한다. 근거가 없거나 비호환이면 rollback을 시작하지 않고 blocked다.
rollback workflow가 producer B를 새로 만들지 않는다.

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

failure fence는 성공한 CAS 상태 commit에 저장된 시각이며 그 commit ID는 Git 이력에서 얻는다. 자기 SHA를
같은 파일에 쓰지 않는다. 잠금 해제 시 이미 완료된 적격 후보도 다시 조회해 재개하며 새 webhook만 기다리지
않는다. 최신성·실패 이력·fence 검증은 실제 잠금 획득·P 게시 전에 반복한다. 새 정상 배포 뒤에도 실패 기록을
지우지 않는다. 수동 rollback·차단 해제는 같은 잠금·호환성·검증 경로를 사용하고 자동 재개 경계를 임의로
우회하지 않는다.

## 8. 유지보수·최초 인수·현재 상태

offset reset, key/인증서 교체와 수동 데이터 작업은 일반 image rollback과 다르다. 공통 잠금과 자동 승격
보류를 먼저 확보하고 해당 parent/child의 auto sync·self-heal을 필요한 범위에서 중지한다. parent가 child의
중지를 복구하지 않도록 선언도 맞춘다. 대상·사유·현재 P/C/V/B·복귀값과 실제 종료를 확인한 뒤 운영하고,
Git 정합·기능 확인 후 재개한다. key verifier→signer 순서와 인증서 metadata 기반 명시적 재배포는 유지한다.
prune off는 데이터 보존 보장의 대체가 아니며 Namespace·CRD·Kafka CR·PVC 삭제는 별도 운영 절차다.

완료 manifest 도입 전의 기존 이미지는 같은 SHA/run을 꾸며 B로 만들지 않는다. 고정 C와 handoff snapshot H로
수동 인수하고 검증한 새 B·V/P·공통 state가 준비된 뒤 자동화를 활성화한다. Argo 인계 전의 수동 사본 절차는
Strimzi 13개 YAML과 대응 C에만 적용한다. `dev/bundle.json`, `dev/desired.yaml`, `dev/applications/`,
`dev/delivery-state.json` 중 하나라도 있으면 수동 사본 복사를 중단한다.

PROJ-280·281·286은 설계 문서다. 완료 manifest, 별도 image 승격·설정 publisher, 공통 Git CAS 상태,
실패 검증·자동 rollback·재개, 권한·branch 보호, Application/CMP/health 및 실제 인수는 구현·검증 대상이다.
기존 Strimzi·관측 배포 완료를 이 자동화의 완료 증거로 사용하지 않는다. PROJ-276의 사본 범위를 계승하며,
현재 구현 상태와 실제 운영 증거는 각 소유 레포에서 확인한다.

구현·운영은 [infra ADR 0056](https://github.com/soma-376/infra/blob/docs/PROJ-280-dev-argocd-adr/docs/adr/0056-dev-argocd-gitops-operations.md),
backend 공급·승격·rollback은 [backend ADR 0076](https://github.com/soma-376/pulsemetry-backend/blob/docs/PROJ-286-backend-image-delivery-adr/docs/adr/0076-dev-앱-이미지는-검증한-묶음으로-GitOps에-인계한다.md)을 따른다.
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
| 기존 Helm 인수·인계 후 수동 사본 시도 | resource/PVC/offset/Secret 보존; 자동화 metadata 존재 시 수동 복사 거부 |
