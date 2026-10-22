# 0023. dev 백엔드는 EKS에서 실행하고 이미지 공급과 수동 배포를 분리한다

## Status

Accepted (PROJ-250. dashboard ADR 0021의 dev ECS/EKS 병행 실행·이미지 비교·ECS CD·두 ALB 전제와
JWT ADR 0022의 dev ECS 병행 소비·writer 전달 부분을 대체한다. 나머지 조회/명령·저장소·라우팅·인증·키 수명
결정은 유효하다. 관련 구현과 계약은 머지 전까지 변경 중이며, 실제 AWS 철거·배포·기능 검증 완료를 뜻하지 않는다.)

## Context

[PROJ-247](https://team376.atlassian.net/browse/PROJ-247)은 dev DB 연결 한도 문제로 완전한 전환 검증 전에
ECS를 조기 제거하기로 했다. 이 사유는 연결 누수 진단이나 EKS 전환의 기능·복구 검증 완료를 뜻하지 않는다.
infra PROJ-248은 EKS 전용 인프라를, backend PROJ-249는 ECS 조회·재배포 없는 이미지 공급을 구현한다.
허브 계약이 ECS CD와 두 실행 환경의 비교를 계속 요구하면 제거된 서비스에 이미지 공급을 종속시키게 된다.

이 결정의 당사자는 `pulsemetry-backend`와 `infra`다. `pulsemetry-frontend`·`telemetryctl`의 공개 API·
인증·전송 계약은 유지한다. PROJ-250은 허브의 레포 간 실행·배포 책임을 정합화하며 인프라 철거 명령이나
앱 구현을 소유하지 않는다.

## Decision

1. dev 백엔드는 기존 EKS에서 enrollment-api·telemetry-ingest·dashboard-api와 ClickHouse를 실행한다.
   dev ECS 실행 경로를 제거하되 prod ECS와 별도 frontend Terraform ECS의 계약은 유지한다.
   기존 RDS·S3·Secret 원본·EKS·PVC·ALB/TG/WAF·private API와 SSM relay를 보존한다.
2. backend dev CI는 세 앱의 `linux/arm64` 이미지 검증·build/push만 수행한다. ECS 서비스 사전 조회·
   재배포·안정화 대기와 CDK·Helm·kubectl·task definition 등록을 수행하지 않는다.
   infra의 backend-dev 역할은 세 앱 ECR 공급 권한만 가지며 EKS 배포 권한을 추가하지 않는다.
3. 세 `sha-<commit>` 이미지가 모두 게시된 뒤 각 build output digest로 `dev` 태그를 enrollment → ingest →
   dashboard 순서로 승격한다. 게시 실패 시 승격하지 않고, 승격 중 실패 시 이후 앱을 중단하며 이미 갱신한
   태그는 유지한다. 같은 digest의 재실행은 멱등 처리한다. 태그 갱신은 원자적 배포나 EKS rollout이 아니다.
   `sha-<commit>`도 재실행으로 이동할 수 있으므로 commit·태그 대응과 **immutable digest**를 구별해 기록한다.
4. infra CDK는 AWS 자원·IAM을, infra Helm과 운영 절차는 Kubernetes workload·runtime 공급·수동 배포를,
   EKS Auto Mode는 노드 생성·교체를 소유한다. 운영자는 확인한 앱별 digest와 runtime/workload 입력으로
   배포한다. 실제 Pod imageID를 ARM64 manifest digest에 대조하며 image index 사용 시 하위 manifest 관계도 확인한다.
5. dev 설치는 저장소·권한 준비 후 enrollment의 원천 Flyway 적용을 확인하고 ingest·dashboard 순서로 진행한다.
   ingest는 Flyway를 실행하지 않고 dashboard는 자기 캐시 DDL을 기동 시 적용한다. 키 교체는 일반 설치 순서와
   별개로 전체 verifier → signer → 마지막 구 키 발급 후 330초 → 구 공개키 제거 순서를 따른다.
6. dev EKS는 기존 JWT 두 원본과 선택 버전을 유지한다. enrollment는 개인키·공개키, dashboard는 공개키만,
   ingest는 JWT 권한 없음이라는 경계를 유지한다. Pod Identity·ASCP/CSI로 별도 파일을 공급하고 명시적으로 재배포한다.
   ECS 제거를 이유로 원본을 재생성하지 않으며, 철거 전 실제 남은 verifier/signer도 키 교체 대상에 포함한다.
7. 명시 path·method·OPTIONS·default 404와 기존 WAF 정책을 dev EKS ALB에 유지한다. 공개 API, 포트 설정의
   식별자, 인증 정책과 키 수명은 바꾸지 않는다. 이미지 게시 성공·Pod readiness·기능 검증 성공은 각각 기록한다.
8. 실제 철거는 backend 이미지 공급과 이 계약의 정합화 이후 [infra 철거 런북](../../infra/docs/runbooks/dev-ecs-retirement.md)을
   따른다. ECS ClickHouse 데이터는 수동 EBS Snapshot으로 보존하고 EKS PVC로 이관하지 않는다.
   Snapshot 완료와 복원 검증은 별개이며, 세부 중지·삭제·배포 순서는 허브에 중복하지 않는다.

## Constraints

- Secret 원문·실제 계정 ID는 저장소 문서나 공개 검증 출력에 넣지 않는다. ARN·버전·digest의 실제 배포 기록은
  제한된 운영 입력과 증거에 남긴다. 키 형식·보존·최소 권한은 [JWT ADR 0022](0022-user-jwt-key-provisioning.md)를 따른다.
- prod의 기존 ECS 자원·배포 역할과 frontend Terraform의 배포 계약을 dev 백엔드 제거에 포함하지 않는다.
- 문서 채택이나 코드에서 스택을 제거한 사실은 실제 AWS 스택 삭제를 뜻하지 않는다. HTTP/TLS·SSM 접근·
  DB 권한·인증·수집·조회·복구는 운영 증거로 별도 확인한다.

## Alternatives Considered

- ECS/EKS 병행 검증을 끝낸 뒤 제거: 기존 결정이지만 DB 연결 한도 문제에 따른 조기 제거 선택과 맞지 않는다.
- ECS 서비스만 중지하고 이미지 공급 workflow를 유지: 서비스 존재·상태 조회에 의존하므로 철거 후 이미지 공급이 막힌다.
- backend CI에 EKS 자동 배포를 추가: 이번 제거에 필요하지 않으며 앱 공급 역할과 인프라 운영 권한을 함께 넓힌다.
- `dev` 태그만 배포 기록으로 사용: 가변 태그와 부분 승격 때문에 실제 실행 버전을 재현할 수 없다.

## Consequences/Tradeoffs

### Positive

- 이미지 공급이 ECS 서비스 존재에 의존하지 않으며 앱 공급과 인프라 운영 책임이 분명해진다.
- 기존 EKS·데이터·키 원본을 재사용하고 immutable digest로 공급물과 실행 버전을 대조한다.

### Negative

- 완전한 기능·복구 검증 전에 ECS 복귀 경로를 제거한다. 삭제 후 복구에는 이전 인프라 재생성과 Snapshot 복원이
  필요할 수 있으며 코드 revert나 Helm rollback만으로 ECS 데이터를 복구할 수 없다.
- 세 `dev` 태그의 승격은 부분 성공할 수 있다. 운영자는 같은 릴리스의 앱별 digest를 확인해야 한다.
- 수동 배포·키 교체·기능 검증 부담을 유지한다. DB 연결 문제 해결 여부도 별도 전후 관측이 필요하다.

## Follow-up

- PROJ-248: infra 코드·로컬 검증과 철거 런북을 소유한다. PROJ-249: backend 이미지 공급 workflow를 소유한다.
  PROJ-250: 이 허브의 배포 계약을 소유한다. backend·infra 담당자가 교차 검토하고 관련 구현과 계약의 반영을 확인한다.
- **현재 상태**: infra의 EKS 전용 구성과 backend의 이미지 공급 workflow는 로컬 변경에서 확인했다.
  PROJ-250에서는 실제 이미지 게시·AWS 철거·Snapshot·EKS 기능·DB 연결 개선을 검증하지 않는다.
- 최신 main 통합과 동번호 ADR 충돌 정리는 [허브 통합 인계](../README.md#proj-250-현재-상태와-main-통합-인계)를 따른다.

## Acceptance Criteria

- dashboard·telemetry-ingest·data-model 계약, architecture와 인덱스가 dev EKS 실행 및 이미지 공급/수동 배포 경계를 일치시킨다.
- 기존 dashboard ADR 0021·JWT ADR 0022의 부분 대체 범위가 본문과 인덱스에 표시되고 나머지 계약은 유지된다.
- workflow의 세 이미지 게시·digest 기반 순차 승격·부분 실패 처리와 문서가 일치한다. 이미지 게시를 배포 완료로 쓰지 않는다.
- 문서 링크·앵커·diff를 검사하고 잔존 ECS 표현을 prod·frontend·과거 기록·철거 전 실제 잔여로 구분한다.

## References

- [Dashboard 계약](../contracts/dashboard-api.md), [telemetry ingest 계약](../contracts/telemetry-ingest.md), [공유 데이터 계약](../contracts/data-model.md)
- [infra ADR 0035 - dev 백엔드 EKS 전용](../../infra/docs/adr/0035-dev-backend-eks-only.md)
- [infra EKS 수동 배포 런북](../../infra/docs/runbooks/dev-eks-manual-deployment.md), [PROJ-248 로컬 검증 기록](../../infra/docs/runbooks/proj248-local-validation.md)
- backend `.github/workflows/deploy_dev.yml`의 `Build and push`·`Promote dev image tags`, `promote_image()`
- backend enrollment-api·telemetry-ingest `application.yaml`의 `spring.flyway.enabled`
- infra `synthDev()`, `DEPLOY_TARGETS`, EKS 런북의 최초 이미지·공통 저장소 준비 및 JWT 교체 절차
