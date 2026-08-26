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
| [`pulsemetry-backend`](#pulsemetry-backend) | Kotlin / Spring Boot / Gradle | enrollment API, **enrollment 스키마의 진실원(Flyway)** | `develop` | 9건 |
| [`telemetryctl`](#telemetryctl) | Go | CLI + 데스크탑 데몬, manifest 계약 스키마 | `develop` | 8건 |
| [`ai-telemetry-pipeline`](#ai-telemetry-pipeline) | TypeScript + Python | auth-proxy, telemetry-processor | `develop` | 없음 |
| [`infra`](#infra) | AWS CDK v2 (TS) | **모든 AWS 리소스**, collector 배포 설정 | `develop` | 23건 |
| [`rdb-schema`](#rdb-schema) | dbml | 설계도(다이어그램). 마이그레이션 아님 | `main` | — |
| [`otel-collector`](#otel-collector) | yaml | **superseded — 신규 작업 금지** | `main` | — |
| [`team-376-llm-wiki`](#team-376-llm-wiki) | Markdown (Obsidian) | 회의·멘토링·의사결정 원본 누적 | `main` | wiki/decisions |
| [`.github`](#github) | Markdown + yaml | `CONVENTION.md`, PR·워크플로 템플릿 | `main` | — |

`docs`·`agent-skills`·`rdb-schema`·`team-376-llm-wiki`·`.github`는 `main` 단일 브랜치다.
코드 레포 4종(`pulsemetry-backend`·`telemetryctl`·`ai-telemetry-pipeline`·`infra`)만 협업 컨벤션 v2의
develop/squash 플로우를 따른다.

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

Kotlin + Spring Boot, Gradle 멀티모듈. [`overview.md`](overview.md)의 **Auth Service** 자리를 맡는다.

```
apps/enrollment-api/         Spring Boot 애플리케이션 (유일한 app)
libs/enrollment-persistence/ JPA 엔티티 · 리포지토리 · Flyway 마이그레이션
```

- **소유**: `POST /v1/enroll`, `POST /v1/installations/telemetry-token`, `POST /v1/invitations`,
  부트스트랩 스크립트·바이너리 서빙(`GET /windows|/unix|/bin/{f}`), **manifest 저장**,
  그리고 **enrollment 스키마의 진실원(Flyway)**.
- **아직 없지만 이 레포의 몫**: 사람 계정·로그인(이 레포가 Auth Service다), manifest 작성 API(현재 수동 INSERT).
  구현이 없는 것이지 소유가 없는 것이 아니다.
- **소유가 확정되지 않음**: **대시보드 API**의 소재(이 레포의 모듈인지 별도 레포인지 — [`../contracts/dashboard-api.md`](../contracts/dashboard-api.md) §2),
  그리고 **텔레메트리 파이프라인 + ClickHouse 스키마**(backend ADR-0006 `Proposed`가 이 레포로의 병합을 제안 중. 아래 `ai-telemetry-pipeline` 항목 참조).
- **소유하지 않음**: AWS 리소스·배포 설정(`infra`), 로컬 수신기·데몬·manifest 계약 스키마 파일(`telemetryctl`), 스키마 다이어그램(`rdb-schema`).
- 모듈 경계·네임스페이스 규칙은 ADR-0008, 인증 계층은 ADR-0007.
- 스키마 enum 물리 타입은 **native enum**(ADR-0009가 ADR-0004의 varchar+CHECK를 대체). 진실원은 여전히 Flyway다.
- 계약: [`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) · 서버 측 상세 명세는 레포의 `docs/enrollment-server-spec.md`.

> **루트의 `PLAN.md` · `PR.md` · `RALPH-PLAN.md` · `DOMAIN-BOUNDARY-NOTES.md` · `PLAN-ADR-0008.md` ·
> `enrollment-api-branch-review.md` · `PR2-*.html`은 스크래치다. 스펙으로 신뢰하지 않는다.**
> 소스 주석 다수가 존재한 적 없는 `PLAN.md §6.2 / A5 / R4 / L11`을 인용한다 — 이 인용도 스펙이 아니다.
> 권위 있는 문서는 `docs/enrollment-server-spec.md`와 `docs/adr/`뿐이다.

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

- **소유**: 로컬 수신기, 로컬 집계·보존, 벤더 도구 설정 배선, 데몬 라이프사이클,
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
- ⚠️ **이 레포의 존속이 확정 상태가 아니다.** backend ADR-0006(`Proposed`)이 파이프라인을 Kotlin/Spring으로
  재작성해 `pulsemetry-backend`로 병합하고 ClickHouse 스키마 소유권도 가져가는 것을 제안한다.
  **채택 전까지 소유는 이 레포에 있다.** 채택되면 이 절과 backend 절, `contracts/telemetry-ingest.md`의
  소유권 서술을 함께 고친다. 이 결정은 두 레포에 걸리므로 스코프 규칙상 허브 ADR이 맞다 —
  [`../adr/README.md`](../adr/README.md)의 크로스레포 후보 참조.
- 알려진 상태: 레포 테스트 0개(CI는 auth-proxy typecheck/build만). README·`docs/`는 PROJ-52 이전 `src/` 구조 기준으로 스테일이다.

## infra

AWS CDK v2 (TypeScript) **단일 인프라 레포**. ADR-0009에 따라 인프라는 여기서만 관리한다.

```
lib/common/  환경 무관 계약 상수 + 순수 헬퍼
lib/prod/    운영 4스택 + synthProd()
lib/dev/     dev 4스택 + synthDev()
lib/cicd/    DeployStack + synthCicd()   (배포 역할 4개 — ADR-0024)
config/otel-collector.yaml   ★ ECS가 실제로 기동하는 collector 설정
```

- **소유**: 모든 AWS 리소스, **배포용 collector 설정**, 배포 역할.
- **소유하지 않음**: 앱 코드. 앱 레포 CI는 이미지 빌드 → ECR push → `ecs update-service --force-new-deployment`까지만 한다.
  **태스크 정의를 바꾸려면 반드시 이 레포를 경유한다.**
- 폴더 의존 방향은 `prod → common`, `dev → common`, `cicd → common` **단방향**. `prod ↔ dev` 상호 참조 금지.
- region은 `ap-northeast-2`. account는 `CDK_DEFAULT_ACCOUNT`에서만 주입하고 코드·문서에 기록하지 않는다.
- **`AGENTS.md`(719줄)가 이 레포의 실질 핸드오프 문서다.** 3장(불변 규칙)과 5장(남은 작업)을 먼저 읽는다.
- ADR 번호 `0020`은 로그 그룹 정책용으로 **예약**돼 있다. 새 ADR은 `0025`부터.

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
