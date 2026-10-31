# 레포별 역할·경계·상태

soma-376 org의 레포 지도다. **어느 레포에 코드를 쓸지, 그 레포가 무엇을 소유하는지**를 여기서 정한다.
목표 구조는 [`overview.md`](overview.md), 레포 사이의 인터페이스는 [`../contracts/`](../contracts/README.md)에 있다.

## 클론 배치 (형제 체크아웃)

에이전트 스킬과 문서 참조가 이 배치를 1차 경로로 가정한다. 반드시 한 부모 디렉토리 아래 형제로 클론한다.

```
soma-376/                        ← git 레포가 아닌 컨테이너 폴더
├── docs/                        ← 이 레포. 제품·아키텍처·계약의 단일 출처
├── agent-skills/                ← 공용 에이전트 스킬 (Claude 플러그인 겸 Codex 스킬 소스)
├── pulsemetry-backend/
├── telemetryctl/
├── ai-telemetry-pipeline/
├── infra/
├── infra-generated-values/  ← private generated·GitOps 선택 저장소
├── rdb-schema/
├── otel-collector/              ← superseded
├── team-376-llm-wiki/
└── .github/
```

## 한눈에

| 레포 | 스택 | 소유하는 것 | 기본 브랜치 | ADR |
|---|---|---|---|---|
| [`docs`](#docs) | Markdown | 제품·아키텍처·**레포 간 계약** | `main` | 크로스레포 ADR |
| [`agent-skills`](#agent-skills) | Markdown + bash | 공용 에이전트 스킬 4종 | `main` | — |
| [`pulsemetry-backend`](#pulsemetry-backend) | Kotlin / Spring Boot / Gradle | enrollment·ingest·Gateway·dashboard 앱, 원천 Flyway·앱 이미지 | `develop` | 레포 인덱스 |
| [`telemetryctl`](#telemetryctl) | Go | CLI + 데스크탑 데몬, manifest 계약 스키마 | `develop` | 8건 |
| [`ai-telemetry-pipeline`](#ai-telemetry-pipeline) | TypeScript + Python | auth-proxy, telemetry-processor | `develop` | 6건 |
| [`infra`](#infra) | AWS CDK v2 (TS) / Helm / Terraform | AWS·Kubernetes·생성기·배포와 복구 | `develop` | 레포 인덱스 |
| [`infra-generated-values`](#infra-generated-values) | YAML / JSON / Markdown | private generated 사본·완료 manifest·desired 선택 | `main` | 허브 ADR 0031 |
| [`rdb-schema`](#rdb-schema) | dbml | 설계도(다이어그램). 마이그레이션 아님 | `main` | — |
| [`otel-collector`](#otel-collector) | yaml | **superseded — 신규 작업 금지** | `main` | — |
| [`team-376-llm-wiki`](#team-376-llm-wiki) | Markdown (Obsidian) | 회의·멘토링·의사결정 원본 누적 | `main` | wiki/decisions |
| [`.github`](#github) | Markdown + yaml | `CONVENTION.md`, PR·워크플로 템플릿 | `main` | — |

`docs`·`agent-skills`·`rdb-schema`·`team-376-llm-wiki`·`.github`는 `main` 단일 브랜치다.
코드 레포 4종(`pulsemetry-backend`·`telemetryctl`·`ai-telemetry-pipeline`·`infra`)만 협업 컨벤션 v2의
develop/squash 플로우를 따른다. `docs`는 develop 없이 티켓 브랜치에서 `main`으로 직접 PR을 열고,
`.github/workflows/`의 PR 자동화가 main 대상 PR에 적용된다.

---

## docs

이 레포. 제품 정의·시스템 아키텍처·**레포 간 계약**의 단일 출처.

- **소유**: `product/` · `architecture/` · `contracts/` · 크로스레포 `adr/` · `glossary.md`
- **소유하지 않음**: 단일 레포 구현 지식. 그것은 각 레포의 `docs/`·`AGENTS.md`에 있다.
- 배치 판별 규칙은 [`../README.md`](../README.md).

## agent-skills

Claude Code 플러그인 마켓플레이스이자 Codex 스킬 소스. 스킬 4종 — `spec` · `adr` · `adr-new` · `conventions`.

- 각 코드 레포에 커밋된 `.agents/skills` 심링크가 이 레포의 `plugins/pulsemetry/skills`를 가리킨다(형제 배치 전제).
- 형제 배치가 아닌 환경은 `scripts/install-codex-skills.sh`가 `~/.agents/skills`에 설치한다.
- 설치·업데이트 경로는 [`../onboarding.md`](../onboarding.md).

## pulsemetry-backend

Kotlin + Spring Boot, Gradle 멀티모듈. dev의 enrollment-api·telemetry-ingest·telemetry-gateway·dashboard-api
네 앱과 앱별 ARM64 이미지를 소유한다. 데이터·인증·공개 API 계약은 각 기존 계약을 유지하며
GitOps 도입으로 변경하지 않는다.

```
apps/enrollment-api/         인증·설치·조직 관리, enrollment/telemetry_ops Flyway 선행
apps/telemetry-gateway/      JWT 검증·Kafka 발행
apps/telemetry-ingest/       Kafka 소비·HTTP 복구 경로·마스킹·변환·보강·적재
apps/dashboard-api/          대시보드 조회
libs/enrollment-persistence/ 원천 JPA·Flyway
```

- **소유**: 앱 코드·검증·스키마, 네 앱의 ECR 게시와 같은 SHA·run/attempt의 완료 manifest.
  태그 순차 변경을 배포 완료로 보지 않으며 실제 digest·ARM64 검증과 Gateway 포함을 요구한다.
- **소유하지 않음**: AWS/CDK·Kubernetes/Helm·private desired의 변경과 배포. 이 경계는
  [GitOps 공급 계약](../contracts/generated-values-sync.md)이 소유한다. backend CI는 이미지를 공급하고
  infra 승격기가 C/V/B를 검증해 Argo에 인계한다.
- dev 수집 경로와 성공 의미는 [Kafka 계약](../contracts/telemetry-kafka.md), 기존 HTTP 복구 경로는
  [telemetry ingest 계약](../contracts/telemetry-ingest.md)을 따른다. 프로세스 내부 처리 모듈은 backend
  `docs/module-map.md`가 소유한다.
- 원천 스키마의 진실원은 backend Flyway다. 새 버전 배포는 enrollment의 migration 성공 뒤
  ingest → Gateway → dashboard 순서다. 앱 교체 도중 구버전·신버전 공존과 rollback 호환성을 검증한다.
- `telemetryctl`의 로컬 수신기·데몬·manifest JSON Schema와 `infra`의 배포 설정을 이 레포로 옮기지 않는다.

### 현재 상태

네 앱과 Strimzi 실행 경로는 기존 코드·[Kafka 전환 검증](../contracts/telemetry-kafka.md#현재-상태와-검증)을 따른다.
[ADR 0031](../adr/0031-dev-gitops-delivery.md)의 완료 manifest와 자동 승격은 승인 설계이며 후속 구현·인수 대상이다.
미병합 [PR #11](https://github.com/soma-376/docs/pull/11)의 dashboard·JWT 상세 계약 전체를 이 작업으로 채택하지 않는다.

> **루트의 `PLAN.md` · `PR.md` · `RALPH-PLAN.md` · `DOMAIN-BOUNDARY-NOTES.md` · `PLAN-ADR-0008.md` ·
> `enrollment-api-branch-review.md` · `PR2-*.html`은 스크래치다. 스펙으로 신뢰하지 않는다.**
> 권위 있는 소스는 backend의 `docs/`와 ADR 및 이 허브의 채택된 계약이다.

## telemetryctl

Go. CLI(`pulsemetry enroll`)와 데스크탑 데몬. [`overview.md`](overview.md)의 **Desktop Application** + **Local Store**.

```
cmd/telemetryctl/   진입점
internal/
  enrollment/  contract/    서버와의 enroll 계약
  receiver/    otlpdecode/  로컬 OTLP 수신기(127.0.0.1:4318) · 디코드 · 스크럽
  store/ session/ rollup/   SQLite 로컬 집계
  forward/                  회사 엔드포인트로 상위 전송
  config/                   ~/.claude/settings.json · ~/.codex/config.toml 배선
  credential/ autostart/    OS 키링 · launchd/systemd 등록
contracts/*.schema.json     ★ manifest·envelope JSON Schema — 계약의 기계 판독 원본
```

- **소유**: 로컬 수신기, 로컬 집계·보존, 벤더 도구 설정 배선, 데몬 라이프사이클, GUI 설치·정책 상태 표시,
  그리고 **manifest 계약 스키마 파일**(`contracts/enrollment-manifest.schema.json`).
- **소유하지 않음**: 서버 측 발급 로직, 조직 정책의 결정.
- 데몬은 프라이버시 **1차** 집행 지점이다. 로컬 배선은 의도적으로 과수집하고, 상위 전송 직전 forwarder가
  회사 manifest의 `signals` 게이팅과 `privacy` denylist 스크럽을 적용한다.
- 로컬 보존 400일(ADR-0008), 로컬 파이프라인 opt-out(ADR-0006), 비정상 종료 시에만 자동 재시작(ADR-0007).
- **Windows 자동 시작 미구현**(PROJ-56). manifest `protocol: "grpc"`는 클라이언트가 상위 전송을 지원하지 않아 데몬이 기동을 거부한다.
- 계약: [`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) · [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md)

## ai-telemetry-pipeline

앱 2개가 한 레포에 있다. [`overview.md`](overview.md)의 **Collector 뒤편 — Masker·Adapter·Enricher** 자리.

```
apps/auth-proxy/           TypeScript. OTLP 요청의 토큰 검증 → x-pulsemetry-* 4헤더 부여
apps/telemetry-processor/  Python. 신원 스탬핑 → 벤더별 어댑터 정규화 → 조직 결합 → ClickHouse 적재
  normalizer/  claude_code. · codex. 이벤트 매핑
  enrichment/  team_memberships as-of 조인
otel-collector-config.yaml ★ dev(in-repo) collector 설정 — 배포 설정은 infra 소유
sql/rds/                   dev 부트스트랩용 DDL·시드 (진실원 아님 — backend Flyway가 진실원)
```

- **소유**: 토큰 검증과 신원 헤더 부여, OTLP 정규화 규칙, ClickHouse `enriched_events` 적재.
- **소유하지 않음**: 토큰 **발급**(backend), enrollment 스키마 **DDL의 진실원**(backend Flyway),
  **배포 collector 설정**(infra).
- `sql/rds/schema.sql`은 dev 편의용 부트스트랩이다. 스키마를 바꿔야 하면 backend Flyway를 고친다.
- 계약: [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md)
- **이 레포는 backend로 병합된다**([허브 ADR 0004](../adr/0004-telemetry-pipeline-repo-merge.md)).
  파이프라인 로직 전체가 Kotlin으로 재작성돼 `pulsemetry-backend`로 옮겨가고, auth-proxy는 폐기되며
  ClickHouse 스키마 소유권도 함께 이동한다. **이관이 끝나면 이 레포는 아카이브된다.**
  위 소유 서술은 기존 구현의 범위이며 현재 dev 실행 경로는 backend Gateway·Strimzi·ingest다.
  prod의 기존 collector·processor 실행 계약은 dev GitOps 전환으로 바꾸지 않는다.
- 알려진 상태: 레포 테스트 0개(CI는 auth-proxy typecheck/build만).
  이 공백을 메우는 특성화 테스트 확보가 이식 착수의 선행 조건이다(허브 ADR 0004 Follow-up).
  README·`docs/`의 코드 경로·구조 서술은 PROJ-79에서 `apps/` 배치에 맞췄다.
  다만 `docs/normalizer.md`·`docs/diagnostics.md`의 **필드 단위 정의는 아직 코드와 대조하지 않았다.**

## infra

AWS CDK v2 (TypeScript)·Helm과 별도 frontend Terraform을 담는 인프라 레포다.
AWS/CDK와 Kubernetes/Helm, Auto Mode의 노드 생성·교체 책임을 구분한다.

```
lib/common/  환경 무관 계약 상수·헬퍼
lib/prod/    prod 스택
lib/dev/     dev Network·Data·Edge·EKS
lib/cicd/    OIDC 배포 역할
helm/        dev workload·platform·Kafka·관측 선언
config/otel-collector.yaml   prod 배포용 collector 설정
```

- **소유**: AWS 리소스·배포 역할, Kubernetes chart·runtime 입력·generated 생성기,
  후보 검증·승격, Application/CMP·health·인수 및 복구 절차.
- **소유하지 않음**: 앱 코드와 앱 이미지 생성. backend는 네 앱 완료 manifest를 공급하고
  infra는 [GitOps 계약](../contracts/generated-values-sync.md)에 따라 검증한 C/V/B를 선택한다.
- Argo CD·Image Updater 자체의 설치·갱신은 수동 Helm 관리다. 기존 release를 인계한 dev 앱·Kafka·
  ClickHouse·platform·관측은 auto sync/self-heal·prune off로 운영한다. 같은 자원에 Helm과 Argo의
  상시 writer를 둘 수 없다. Strimzi가 만든 Pod·Service·PVC·인증서 Secret의 소유자는 Strimzi다.
- prod ECS와 frontend Terraform/ECS는 이 dev 결정의 대상이 아니다. 기존 task definition·
  Terraform 소유권과 별도 앱 CD 계약을 유지한다. backend dev CI에 ECS/CDK/Helm 실행을 추가하지 않는다.
- 폴더 의존은 `prod → common`, `dev → common`, `cicd → common` 단방향이다.
  실제 account 식별자·ARN·endpoint는 공개 원본에 넣지 않는다.
- `AGENTS.md`의 불변 규칙과 남은 작업, `docs/adr/README.md`의 현재 ADR 번호·상태를 먼저 확인한다.

### 현재 상태

Strimzi의 배포·관측 검증과 GitOps 자동화는 별도다. PROJ-280·281·286은 설계 문서이며
Argo/Updater 설치·candidate 승격·Application/CMP·실제 인수 완료를 뜻하지 않는다.
인수 전에는 기존 infra 수동 Helm 런북을 유지한다.

## infra-generated-values

`soma-376/infra-generated-values`는 private 저장소이며 기본 브랜치는 `main`이다.
공급 경계의 단일 출처는 [generated values 계약](../contracts/generated-values-sync.md)이다.

- **소유**: `dev/generated/` 사본·`dev/infra-commit.txt`의 C·`dev/bundle.json` 원본 manifest 사본,
  `dev/desired.yaml`의 선택·`dev/applications/`의 infra 템플릿에서 생성된 child Application.
- **소유하지 않음**: 입력·생성기·chart·배포 정책. infra 원본을 수정해 재생성하며 역동기화하지 않는다.
- `candidate/dev`는 비배포 후보이며 Updater는 네 앱 image.ref만 수정한다. infra 승격기가
  main에 snapshot V를 먼저 append하고 별도 P에서 desired와 영향을 받는 child의 C/V pin을 함께 바꾼다.
- Argo CD는 main의 Application을 읽고 child의 C/V를 고정해 적용한다. 저장소 자격증명·Secret 원문·
  kubeconfig·조회 원본·개인 경로는 저장하지 않는다. 실제 ARN·내부 endpoint 참조는 private 사본에만 둔다.

### 현재 상태

[ADR 0031](../adr/0031-dev-gitops-delivery.md)은 승인 설계이며 실제 후보 공급·승격·GitOps 인수는 후속 검증 대상이다.
[PR #16](https://github.com/soma-376/docs/pull/16)의 PROJ-276 선행 브랜치를 기반으로 기존 사본
소유권을 계승한다. [generated 사본 ADR 0030](../adr/0030-generated-values-private-sync-copy.md)의
직접 수정 금지·공급 파일·수동 배포 범위는 ADR 0031의 image 예외·V/P 승격으로 부분 대체·확장한다.
인수 전의 수동 사본 절차와 인수 후 publisher 경로를 구분한다.

## rdb-schema

`dbdiagram.dbml` 한 파일. 팀이 함께 보는 **설계도**다.

- **소유하지 않음**: 마이그레이션. dbdiagram은 다이어그램 도구이지 마이그레이션 도구가 아니다.
  운영 DB를 어떤 순서로 바꿀지의 진실원은 `pulsemetry-backend`의 Flyway다(ADR-0004).
- **원격이 없다 — 현재 local-only 레포다.** 팀이 공유하려면 원격화가 선행돼야 한다.
- 공유 도메인 모델의 서술은 [`../contracts/data-model.md`](../contracts/data-model.md).

## otel-collector

> **superseded by `ai-telemetry-pipeline`. 신규 작업 금지.**

초기 collector 실험 레포다. 현재 파이프라인은 `ai-telemetry-pipeline`이고 배포 설정은 `infra`가 소유한다.
여기 있는 `otel-collector-config.yaml`·`teams.json`은 어느 환경에도 반영되지 않는다.
GitHub archive 처리는 팀 확인 후 진행한다.

## team-376-llm-wiki

Obsidian으로 읽는 팀 위키. 회의·멘토링·의사결정 내역을 누적한다.

- 3-Layer 구조: `sources/`(불변 원본) → `wiki/`(가공 페이지) → `CLAUDE.md`(규칙).
- **운영 매뉴얼은 이 레포의 `CLAUDE.md`다.** 그대로 따른다.
- `wiki/decisions/`의 결정 페이지는 ADR과 성격이 다르다 — 회의에서 나온 결정의 기록이며,
  코드 구조를 구속하는 ADR은 각 레포 `docs/adr/`와 이 허브 `adr/`에 있다.

## .github

org 프로필 레포.

- **소유**: [`CONVENTION.md`](https://github.com/soma-376/.github/blob/main/CONVENTION.md)(협업 컨벤션 v2),
  `.github-template/`(PR 템플릿·워크플로·CODEOWNERS — **새 레포의 `.github/` 디렉토리용**),
  `.repo-template/`(**새 레포 루트용** 에이전트 배선 파일).
- 새 레포를 만들 때 두 템플릿을 모두 복사한다. 절차는 `CONVENTION.md` §0.

---

## 아직 없는 레포

| 예정 | 담을 것 | 착수 전 기획 위치 |
|---|---|---|
| frontend (대시보드) | Dashboard API 클라이언트, 시나리오 런처 UI | 이 허브 `product/` (레포 생성 시 이관) |

Dashboard API 서버가 `pulsemetry-backend`에 들어갈지 별도 레포가 될지는 미정이다.
[`../contracts/dashboard-api.md`](../contracts/dashboard-api.md) 참조.
