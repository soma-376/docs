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

레포가 아직 없는 표면(frontend)의 착수 전 기획은 허브 `product/`에 임시 보관하고, 레포가 생기면 이관한다.

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
| [`contracts/`](contracts/README.md) | 레포 간 계약 4건 | enrollment·ingest·data-model 확정, dashboard 골격 |
| [`adr/README.md`](adr/README.md) | 크로스레포 ADR 인덱스 + 전 레포 카탈로그 | 카탈로그 완성, 크로스레포 ADR 3건 |
| [`glossary.md`](glossary.md) | 도메인 용어 | 골격 |
| [`onboarding.md`](onboarding.md) | 클론 배치·에이전트 셋업·첫 PR | 완성 |
| [`archive/`](archive/README.md) | 증류 이전 원본 PLAN/SA/AA/IA | **기준 문서 아님** |

## ADR을 각 레포에 분산해 두는 이유

크로스레포 ADR만 이 허브가 소유하고, 단일 레포 구현 ADR은 그 레포에 남긴다. 근거는 셋이다.

1. **ADR 개정은 코드와 같은 PR·diff에서 리뷰되어야 한다.** 중앙화하면 결정 하나를 바꾸는 일이 두 레포의 PR로 쪼개지고, 리뷰어가 코드와 결정을 나란히 볼 수 없게 된다.
2. **"코드와 ADR이 다르면 ADR이 우선"은 에이전트가 ADR을 실제로 발견한다는 전제 위에 선다.** 레포를 열자마자 보이는 `docs/adr/`가 중앙 fetch보다 확실하다.
3. **기존 40여 건을 이관하는 이득은 검색 편의뿐이고, 대가는 링크 파손이다.** 검색은 이 허브의 카탈로그와 `adr` 스킬이 해결한다.

크로스레포 결정만 반대로 중앙이 소유한다. 단일 레포에 두면 다른 레포에서 비가시적이고 소유가 모호해지기 때문이다.

## 규칙

- 문서 본문은 한국어, 파일명·코드·식별자는 영어.
- 브랜치는 `main` 단일. 협업 컨벤션 v2의 develop/squash 플로우는 이 레포에 적용하지 않는다.
- 문서와 코드가 어긋나면 둘 중 하나가 틀린 것이다. 발견 즉시 고친다. "일단 코드대로 가고 나중에 문서화"는 금지.
- `archive/`는 신뢰하지 않는다. 증류본이 기준이다.
