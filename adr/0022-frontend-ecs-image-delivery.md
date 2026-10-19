# 0022. 프론트 이미지는 브랜치 머지 후 ECS에 자동 배포한다

## Status

Accepted — [PROJ-242](https://team376.atlassian.net/browse/PROJ-242). infra ADR 0028의 매 릴리스 digest 입력 결정을 부분 대체한다. 인프라의 task definition 소유권과 단일 BFF 프로세스 제약은 유지한다.

## Context

`pulsemetry-frontend`는 검증한 ARM64 이미지를 ECR에 게시하고 `infra`는 고정 digest를 task definition에 넣는다. 현재 방식은 앱 변경마다 인프라 PR을 요구한다. 사용자는 develop 머지 후 dev EC2, main 머지 후 prod Fargate 자동 반영을 요청했다. 앱 CI에 Terraform 및 task definition 등록 권한을 주지 않는 기존 경계를 유지해야 한다.

## Decision

- `ci.yml`은 검증만 담당한다. `cd.yml`은 develop/main push에서 같은 커밋의 `ci.yml`을 workflow_call로 호출한 뒤 검사가 모두 성공하면 각각 dev/prod에 배포한다. PR은 검증만 한다. 환경별 `ECS_DEPLOY_ENABLED=true` 설정으로 최초 활성화하며, 비활성 환경은 기존 이미지 게시만 수행한다.
- ECR은 커밋 SHA·실행 ID·시도 번호 태그를 immutable로 보관한다. 정확히 `deploy` 태그만 변경을 허용한다. 이 태그는 배포 후보를 가리키며 실행 버전의 증거는 ECS deployment ID와 task image digest다.
- Terraform은 runtime의 `automatic_image_deployment=true`일 때 `:deploy`를 참조한다. false일 때는 기존 `image_digest` 고정을 유지한다. 전환 전에 현재 정상 digest에 `deploy`를 붙인다. 자동 배포 중 runtime의 digest는 최초 전환/비상 고정용이며 매 앱 릴리스마다 수정하지 않는다.
- task definition과 서비스 설정은 infra가 계속 소유한다. 앱 CI는 자기 환경 ECR 게시, ECS 서비스 강제 재배포와 검증에 필요한 조회만 수행한다. 앱 역할에 RegisterTaskDefinition, PassRole, Terraform state 접근, secret 값 조회 권한을 주지 않는다.
- ECS `versionConsistency`를 활성화하고 dev EC2는 이미지 pull 실패 시 로컬 캐시로 대체하지 않도록 설정한다. ECS는 새 배포에서 태그를 digest로 해석한다.
- 브랜치별 배포를 직렬화하고 진행 중 배포를 새 push로 취소하지 않는다. 게시 직전과 태그 갱신 직전에 브랜치 HEAD를 확인한다. 배포 시작 후 생긴 커밋은 다음 실행에서 반영한다.
- 배포 전 서비스가 정상이고 task definition이 예상 ECR의 `:deploy`를 참조하는지 확인한다. 실행 중인 태스크의 digest를 복구 기준으로 저장한다. 새 deployment가 완료되고 실제 태스크 digest가 게시된 digest와 일치해야 성공이다. 실패·롤백·시간 초과는 CI 실패다.
- 실패 시 자신이 갱신한 태그인지 확인한 뒤 이전 정상 digest로 태그를 복구한다. ECS circuit breaker의 롤백은 별도로 확인한다. 작업 강제 취소·runner 소실 시 운영자가 실행 상태와 태그를 대조하고 복구한다.
- 인프라 apply와 앱 CD의 동시 실행은 허용하지 않는다. 인프라 변경 전 해당 환경 CD를 비활성화하고 실행이 끝난 뒤 apply한다. 두 저장소의 GitHub concurrency는 서로 잠그지 못한다.

## Alternatives Considered

- 앱 CI에서 새 digest로 task definition을 등록한다: 버전 추적은 쉽지만 인프라 소유권을 분할하고 RegisterTaskDefinition·PassRole이 필요해 채택하지 않는다.
- digest 변경 PR을 매번 infra에 만든다: 검토 가능하지만 요청한 머지 후 즉시 자동 배포가 아니며 저장소 간 쓰기 인증도 추가된다.
- 모든 이미지 태그를 mutable로 둔다: 과거 산출물 식별을 보장하지 못해 배포용 한 태그로 제한한다.

## Consequences/Tradeoffs

### Positive

- AWS 리소스와 task definition은 Terraform으로 유지하며 앱 릴리스만 자동 반영한다.
- 커밋별 이미지와 실제 실행 digest로 배포 결과를 추적하고 교차 환경 접근을 제한한다.

### Negative

- `deploy` 자체는 변할 수 있으므로 태그 문자열만으로 실행 버전을 식별할 수 없다. Actions 요약에 커밋·digest·deployment ID를 남긴다.
- 최초 bootstrap·정상 태그·runtime 전환·GitHub 활성화 순서를 지켜야 한다. prod runtime이 없으면 운영 자동 배포도 활성화할 수 없다.
- BFF 단일 프로세스 때문에 기존 태스크 중지 후 교체하며 짧은 중단을 감수한다. EC2 pull 정책 전환은 user data 변경으로 인스턴스를 교체한다.
- 사람의 강제 취소나 두 저장소의 동시 변경은 자동 복구만으로 보장하지 않는다. 유지보수 중 CD 비활성화와 복구 절차가 필요하다.

## Acceptance Criteria

- 오래된 HEAD, 누락된 서비스, 진행 중 배포, 잘못된 이미지 참조, 실패·롤백·digest 불일치를 검출한다.
- IAM과 Terraform mock 검증 및 프론트 필수 검사를 통과한다. 실제 환경 활성화와 배포 검증은 코드 검사 결과와 구분한다.

## References

- [프론트 BFF 운영 안내](https://github.com/soma-376/pulsemetry-frontend/blob/develop/docs/bff-auth.md)
- [infra ADR 0028](https://github.com/soma-376/infra/blob/develop/docs/adr/0028-frontend-alb-and-container-runtime.md)
- [ECS 이미지 digest 해석](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html#deployment-container-image-stability)
- [ECR 태그 변경 예외](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-tag-mutability.html)
