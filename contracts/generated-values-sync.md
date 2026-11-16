# 계약 — dev generated values 게시와 Argo CD 자동 동기화

| 항목 | 내용 |
|---|---|
| 당사자 | `infra` 설정 publisher, `pulsemetry-backend` 이미지 publisher → private `infra-generated-values`; 계약은 `docs` 소유 |
| 배포 저장소 | `soma-376/infra-generated-values`, private, 선택 branch `main` |
| 결정 | [ADR 0036](../adr/0036-simple-dev-git-publication-and-argocd-sync.md) |
| 계약 버전 | desired schemaVersion 2; backend 완료 manifest schemaVersion 1 유지 |
| 상태 | 변경 중 — PROJ-334 게시 구현·PROJ-335 실제 root 연결 검증은 별도 |

## 1. 범위와 소유권

backend는 동일 source SHA/run/attempt에서 성공한 네 ARM64 이미지의 원본 완료 manifest와 digest를 공급한다. 별도 게시 workflow는 최종 성공한 생산 workflow와 artifact를 검증한 뒤 이미지 변경을 Git에 게시한다.

infra는 AWS/CDK·Helm chart·공식 profile/runtime·생성기와 설정 게시를 소유한다. CDK의 plan·승인·같은 source/assembly 배포 규칙은 유지한다. 설정 publisher는 현재 선택 이미지를 유지하고, backend publisher는 현재 infra와 비이미지 설정을 유지한다.

Argo CD만 Git 선언을 Kubernetes에 적용한다. 두 publisher는 Kubernetes·Argo에 쓰지 않고 rollout 완료를 기다리지 않는다. Argo CD 자체는 기존 수동 Helm 관리다. prod·frontend·perf와 수동 Kafka 관리 Job 실행은 범위 밖이다.

## 2. 선택과 private 파일

| 식별자 | 의미 |
|---|---|
| C | chart·공식 runtime·생성기의 검증한 infra 전체 commit SHA |
| B | backend 완료 manifest 원본 UTF-8 바이트의 SHA-256 |
| V | generated·bundle·producer 증거를 담은 immutable snapshot commit SHA |
| P | V와 C를 가리키는 desired·Application 선언을 담은 commit SHA |

`dev/desired.yaml`은 다음 필드만 갖는다. SHA는 40자리 소문자 16진수, bundle hash는 64자리 소문자 16진수다.

```yaml
schemaVersion: 2
infraCommit: <C>
valuesCommit: <V>
bundleManifestSha256: <B>
```

| 경로 | 내용과 변경 주체 |
|---|---|
| `dev/generated/*.yaml` | 정확한 13개 생성 사본. infra는 재생성하고 backend는 네 앱의 image digest만 바꾼다. |
| `dev/infra-commit.txt` | C와 줄바꿈 하나. infra 설정 게시만 변경한다. |
| `dev/bundle.json` | 완료 manifest schemaVersion 1의 원본 바이트. backend 이미지 게시만 변경한다. |
| `dev/producer.json` | 검증한 원본 생산 provenance. backend 이미지 게시만 변경한다. |
| `dev/desired.yaml` | schemaVersion 2의 현재 C/V/B 선택. 두 publisher가 Application 참조와 함께 변경한다. |
| `dev/applications/<id>.yaml` | 기존 catalog의 child 18개. 검증한 C/V 전체 SHA를 참조한다. |
| `dev/root.yaml` | private main의 `dev/applications`만 읽는 root. 최초 연결·운영자 관리 대상이다. |
| 루트 `README.md`, `AGENTS.md` | 저장소 운영 안내. publisher가 덮어쓰지 않는다. |

generated 목록은 `base.yaml`, `lbc.yaml`, `runtime.yaml`, `enrollment-api.yaml`, `telemetry-ingest.yaml`, `telemetry-gateway.yaml`, `telemetry-kafka-admin.yaml`, `dashboard-api.yaml`, `clickhouse.yaml`, `fluent-bit.yaml`, `ascp.yaml`, `observability.yaml`, `kafka-strimzi.yaml`이다. offset-admin 사본은 Job 실행 요청이 아니다.

현재 선택 데이터는 main의 최신 generated 파일을 임의로 조합하지 않고 `desired.valuesCommit`이 가리키는 V에서 읽는다. V의 bundle hash·infra-commit·이미지와 main의 desired·Application 참조가 일치해야 한다.

이 버전은 `delivery-state.json`, `activation.json`, 배포 nonce, `evidence/`, lastKnownGood·실패 fence를 만들지 않는다. 이전 버전의 상태 파일이 발견되면 자동 삭제·변환하지 않고 명시적인 전환을 요구한다.

## 3. 이미지와 producer 증거

완료 manifest는 기존 schemaVersion 1과 정확한 네 앱 목록을 유지한다. 각 이미지는 허용한 `soma-376/<app>` repository, `sha256:<64자리 소문자 16진수>` digest와 ARM64 manifest digest, `linux/arm64`, packaged deployment contract hash를 갖는다. `B`를 manifest 내부에 추가하거나 원본을 재직렬화하지 않는다.

`producer.json`은 검증 결과의 다음 필드만 갖는다.

- `repository`, `workflowRef`, `workflowRunId`, `workflowRunAttempt`, `backendCommit`
- 원본 run의 `createdAt`, `conclusion`, `event`, `headBranch`
- `artifactId`, 원본 ZIP의 `artifactSha256`

repository는 `soma-376/pulsemetry-backend`, workflow는 허용한 `deploy_dev.yml@refs/heads/develop`, conclusion은 `success`, event는 `push` 또는 `workflow_dispatch`, headBranch는 `develop`이다. run·attempt·artifact ID는 양의 정수 문자열이다. `createdAt`은 원본 run 생성 시각이며 재시도 attempt 시작 시각으로 대신하지 않는다.

게시기는 GitHub API로 workflow ID/path·성공 run/attempt·source·artifact ID/ZIP hash를 검증하고 immutable 이미지에서 packaged contract를 실행 없이 추출해 비교한다. producer checkout이나 artifact 안의 코드를 게시 권한으로 실행하지 않는다. Git에는 정제된 provenance만 남기며 원문 API 응답·로그를 복사하지 않는다.

## 4. 두 publisher의 변경 경계

### infra 설정 게시

- 실행 source는 검토한 `develop`의 전체 SHA다. 게시 직전에도 source 최신성과 원본 plan/deploy/receipt·현재 AWS baseline을 확인한다.
- 같은 C의 공식 profile/runtime과 현재 AWS output·Strimzi metadata를 사용해 13개 파일을 생성·Helm 검증한다.
- 현재 선택 V의 네 image repository/digest와 `bundle.json`·`producer.json` 원본 바이트를 보존한다. 최신 ECR 태그나 별도 backend 실행을 설정 변경에 끼워 넣지 않는다.
- 검증한 Application 구조와 C를 갱신하되 기존 identity·releaseName·namespace·syncPolicy를 보존한다.

### backend 이미지 게시

- 성공한 원본 생산 실행을 검증하고 게시 직전에 최신 성공 후보를 다시 평가한다. 현재 선택보다 오래된 원본 run이나 develop 계보 밖 source는 게시하지 않는다.
- 현재 선택 V의 네 앱 digest, 원본 bundle·producer만 바꾼다. image repository, C, 나머지 generated 비이미지 바이트는 그대로 유지한다.
- 현재 Application의 values 참조만 새 V로 갱신한다. chart·namespace·releaseName·권한·syncPolicy는 변경하지 않는다.
- 같은 이미지 조합이라도 새 원본 증거가 필요하면 provenance를 갱신할 수 있다. Pod template에 게시 nonce를 넣지 않는다.

## 5. Git 게시·충돌·결과

1. private main HEAD와 현재 desired/V를 읽고 파일·선택·변경 경계를 검증한다.
2. 새 generated·bundle·producer가 담긴 V를 현재 main 위에 로컬 commit으로 만든다.
3. V를 참조하는 desired·18개 Application의 P를 로컬 commit으로 만든다. P를 한 번의 정상 fast-forward push로 게시한다.
4. push는 예상한 기존 main HEAD를 검사한다. 충돌하면 최신 main의 선택을 다시 읽고 반대 영역을 보존한 후보를 재생성한다. 최초 시도를 포함해 최대 3회다.
5. push 응답이 유실되면 원격 이력에 정확한 P가 포함됐는지 확인한다. 확인되면 게시 성공, 확인 불가면 실패다. force push·자동 rebase는 하지 않는다.

내용과 선택이 같으면 commit을 만들지 않는 no-op이다. 성공 결과는 `published` 또는 `noop`이며, 더 최신의 성공 후보가 확인되면 게시를 중단하고 workflow를 실패시킨다. 공개 로그에는 원문 values·API 응답·자격증명을 남기지 않는다.

Git 게시까지만 직렬화하므로 이전 rollout이 끝나기 전에 새 선택을 게시할 수 있다. Argo CD는 중간 버전을 건너뛰고 최신 선언으로 수렴할 수 있다. `published`는 Argo Synced/Healthy나 실제 기능 성공을 뜻하지 않는다. 게시 실패는 workflow 실패이며 자동 rollback이나 전역 blocked 상태를 만들지 않는다.

## 6. 최초 연결과 Argo 운영

최초 연결은 기존 18개 Application과 실제 C·values·네 이미지·release inventory를 확인한 뒤 수행한다. 여러 snapshot에서 사용하던 generated를 하나로 묶을 때 기존 바이트와 렌더 결과를 보존한다. 새 chart나 이미지 전환은 이 단계에 섞지 않는다.

검토한 초기 snapshot과 Application 선언을 먼저 게시하고, 운영자가 기존 EKS context로 root를 등록한다. root는 `main/dev/applications`만 읽고 자신은 그 경로 밖에 둔다. root·child 모두 auto sync와 self-heal을 켜며 prune·allowEmpty·삭제 finalizer는 끈다. retry limit은 0을 유지한다. 고정 child revision을 평가하는 기존 health·wave·CMP를 재사용한다.

publisher는 이후 syncPolicy를 보존한다. 유지보수는 Git에서 해당 child의 자동화를 끄고 root 반영·진행 operation 종료를 확인한 뒤 수행한다. 복구는 검토한 이전 Git 선언이나 수정 선언을 게시하는 수동 작업이며 DB downgrade·PVC 복원·Kafka offset reset을 자동화하지 않는다.

## 7. 보안·권한

기존 infra 배포·values-reader와 backend의 ECR 읽기 역할을 재사용한다. 이 연결만을 위한 신규 AWS 역할이나 CI의 Kubernetes 쓰기 권한은 만들지 않는다. private Git reader와 publisher 자격은 분리한다.

Secret 원문·token·개인키·비밀번호·kubeconfig·AWS 조회 원문을 Git·로그·공개 artifact에 넣지 않는다. Kafka TLS 입력은 이름과 UID/resourceVersion에서 유래한 metadata revision만 사용한다. 실제 endpoint·ARN이 있는 generated는 private 저장소 안에만 보관한다. repository 쓰기 권한은 필드별 격리를 제공하지 않으므로 파일 허용목록·소유권 검증을 필수로 둔다.

## 8. 현재 상태와 완료 근거

PROJ-334는 이 계약과 두 publisher를 구현한다. PROJ-335는 root·auto sync·self-heal의 실제 dev 연결을 검증한다. 이 문서 작성은 Git 게시·클러스터 적용·기능 검증 완료를 뜻하지 않는다.

완료 근거는 동시 게시·응답 유실·no-op·오래된 실행·반대 영역 보존 테스트, 고정 chart/values 렌더, root+18개 목표 revision의 Synced/Healthy, 초기 child·PVC·이미지·mount 보존, Git 변경의 자동 반영과 비파괴 self-heal 결과다. 로그인·조회·수집·저장 전체 기능과 인증서 회전 검증은 별도 상태로 기록한다.
