# Pulsemetry 문서 허브

Pulsemetry 멀티레포의 **제품·아키텍처·레포 간 계약**에 대한 단일 출처다. 코드는 여기 없다.
레포 하나만 아는 내용은 그 레포의 `docs/`에 있고, 이 허브는 레포를 가로지르는 것만 담는다.

## 무엇이 어디 사는가 (배치 판별 규칙)

| 성격 | 위치 | 예 |
|---|---|---|
| 한 레포만 아는 구현 지식 | 그 레포의 `docs/`·`AGENTS.md` | CDK 스택 구조, Go 패키지 경계, Gradle 모듈 규칙 |
| **두 레포가 합의해야 하는 인터페이스** | 이 허브의 [`contracts/`](contracts/README.md) | enroll API 필드, 토큰 해시 방식, OTLP 전송 헤더 |
| 시스템 전체 그림·용어 | 이 허브의 [`architecture/`](architecture/overview.md)·[`glossary.md`](glossary.md) | 컴포넌트 지도, 데이터 흐름, 불변식 |
| 결정의 이유 | ADR — 스코프 규칙은 [`adr/README.md`](adr/README.md) | 왜 ClickHouse인가, 왜 2단 토큰인가 |
| 제품 정의·범위 | 이 허브의 [`product/`](product/prd.md) | 페르소나, MVP 범위, Non-goals |

레포 간 제품·화면 범위는 허브 `product/`에, 구현·API 소비 상세는 `pulsemetry-frontend/docs/`에 둔다.

## 읽는 순서

1. **처음 오면** — [`product/prd.md`](product/prd.md) → [`architecture/overview.md`](architecture/overview.md) → [`architecture/repos.md`](architecture/repos.md)
2. **레포에 코드를 쓰기 전** — [`architecture/repos.md`](architecture/repos.md)에서 그 레포의 경계 → 그 레포가 당사자인 [`contracts/`](contracts/README.md) 문서
3. **설계를 바꾸기 전** — [`adr/README.md`](adr/README.md)의 스코프 규칙과 전 레포 카탈로그
4. **환경 세팅** — [`onboarding.md`](onboarding.md)

## 문서 지도

| 경로 | 담는 것 | 상태 |
|---|---|---|
| [`product/prd.md`](product/prd.md) | 문제 정의·페르소나·MVP 범위·성공 지표·Non-goals | 증류 완료 (원본 `archive/PLAN.md`) |
| [`product/roadmap.md`](product/roadmap.md) | 마일스톤 골격 | 골격 |
| [`architecture/overview.md`](architecture/overview.md) | 컴포넌트 지도·데이터 흐름·아키텍처 불변식 | 증류 완료 (원본 `archive/SA.md`) |
| [`architecture/repos.md`](architecture/repos.md) | 레포별 역할·경계·현재 상태 | 조사 기반 신규 |
| [`contracts/`](contracts/README.md) | 레포 간 계약 4건 | enrollment 확정; ingest·data-model·dashboard의 dev 실행·배포 계약 변경 중 (ADR 0023) |
| [`adr/README.md`](adr/README.md) | 크로스레포 ADR 인덱스 + 전 레포 카탈로그 | 크로스레포 결정·타 브랜치의 번호 예약 안내 |
| [`glossary.md`](glossary.md) | 도메인 용어 | 골격 |
| [`onboarding.md`](onboarding.md) | 클론 배치·에이전트 셋업·첫 PR | 완성 |
| [`archive/`](archive/README.md) | 증류 이전 원본 PLAN/SA/AA/IA | **기준 문서 아님** |

## ADR을 각 레포에 분산해 두는 이유

크로스레포 ADR만 이 허브가 소유하고, 단일 레포 구현 ADR은 그 레포에 남긴다. 근거는 셋이다.

1. **ADR 개정은 코드와 같은 PR·diff에서 리뷰되어야 한다.** 중앙화하면 결정 하나를 바꾸는 일이 두 레포의 PR로 쪼개지고, 리뷰어가 코드와 결정을 나란히 볼 수 없게 된다.
2. **"코드와 ADR이 다르면 ADR이 우선"은 에이전트가 ADR을 실제로 발견한다는 전제 위에 선다.** 레포를 열자마자 보이는 `docs/adr/`가 중앙 fetch보다 확실하다.
3. **기존 40여 건을 이관하는 이득은 검색 편의뿐이고, 대가는 링크 파손이다.** 검색은 이 허브의 카탈로그와 `adr` 스킬이 해결한다.

크로스레포 결정만 반대로 중앙이 소유한다. 단일 레포에 두면 다른 레포에서 비가시적이고 소유가 모호해지기 때문이다.

## PROJ-250 현재 상태와 main 통합 인계

[dev EKS 전용 ADR 0023](adr/0023-dev-backend-eks-only.md)은 infra PROJ-248의 로컬 구현과
backend PROJ-249의 로컬 이미지 공급 workflow를 기준으로 한다. PROJ-248은 인프라 코드,
PROJ-249는 backend ECS 조회·재배포 제거, PROJ-250은 이 허브의 계약 정합화를 담당한다.
관련 구현과 계약의 머지 전까지 세 계약은 변경 중이며, backend·infra 담당자의 교차 검토 대상이다.

이 작업은 PROJ-228의 `2b3ca08`을 잇는 후속 변경이다. 최신 원격 main 통합·ADR 번호 충돌 정리와
기존 미커밋 수정은 포함하지 않는다. 로컬 구현·검증은 실제 이미지 게시·AWS 철거·Snapshot·EKS 기능 검증이나
DB 연결 한도 문제 해결을 증명하지 않는다. 실제 철거는 [infra 런북](../infra/docs/runbooks/dev-ecs-retirement.md),
수동 설치·운영은 [EKS 런북](../infra/docs/runbooks/dev-eks-manual-deployment.md)을 따른다.
현재 infra 문서의 허브 정합화 PROJ-249 인계 표기는 이 작업에서 수정하지 않았으며, 담당 범위는 위 구분을 따른다.

후속 main 통합에서는 파일 전체를 덮어쓰지 않고 다음 변경을 보존한다. 로컬 `origin/main`은 최신 원격 상태의 근거로 쓰지 않는다.

- 원격 `0022-frontend-ecs-image-delivery.md`와 이 브랜치 `0022-user-jwt-key-provisioning.md`는 다른 결정이다.
  기존 파일명·링크를 보존하고 번호와 제목을 함께 인용한다. frontend ECS 배포는 dev 백엔드 제거 대상이 아니다.
- 원격 dashboard 계약의 사용 환산액 절: 가격판·토큰 의미·null 처리·관측 부분합(ADR 0008·0011).
- 원격 telemetry-ingest 계약의 상태 코드·재시도 §8과 M6·M11·M14 갱신.
- 원격 data-model 계약의 LocalSeeder/Flyway 구분과 telemetry-ingest 읽기 소비자·Flyway 미실행 설명.
- 원격 ADR 인덱스의 0006·0007(`Proposed`)·0008·0011·frontend 0022와 각각의 상태.

이 목록은 이번 조사에서 확인한 통합 주의점이다. main 통합 시 원격을 다시 확인하며, PROJ-250 완료를
main 통합 완료로 기록하지 않는다.

## 규칙

- 문서 본문은 한국어, 파일명·코드·식별자는 영어.
- 브랜치는 `main` 단일. 협업 컨벤션 v2의 develop/squash 플로우는 이 레포에 적용하지 않는다.
- 문서와 코드가 어긋나면 둘 중 하나가 틀린 것이다. 발견 즉시 고친다. "일단 코드대로 가고 나중에 문서화"는 금지.
- `archive/`는 신뢰하지 않는다. 증류본이 기준이다.
