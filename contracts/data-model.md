# 계약 — 공유 도메인 모델 (tenant / team / member)

| 항목 | 내용 |
|---|---|
| 당사자 | **`pulsemetry-backend`**(enrollment 쓰기·DDL 진실원, dev ingest·dashboard 읽기 소비자), legacy **`ai-telemetry-pipeline`**(읽기 소비자). 설계도는 **`rdb-schema`** |
| 물리 위치 | 공유 RDS `controlplane`의 **`enrollment` 스키마** |
| 관련 ADR | backend ADR-0004(진실원 = Flyway), ADR-0009(native enum) / infra ADR-0012(컨트롤 플레인 DB) / [허브 ADR 0023](../adr/0023-dev-backend-eks-only.md)(dev EKS 전용 배포) |
| 상태 | 변경 중 — PROJ-250의 dev 배포·DDL 적용 경계 정합화. 관련 구현 merge 전이며 실제 DB 적용·E2E 완료와 별도 |

파이프라인이 신호를 조직·팀·사람에 귀속시키려면 backend가 쓴 테이블을 읽어야 한다.
**그 테이블의 정의가 이 계약이다.**

## 1. 소유권

| 역할 | 레포 | 내용 |
|---|---|---|
| **DDL 진실원** | `pulsemetry-backend` | `libs/enrollment-persistence`의 **Flyway 마이그레이션**. 운영 DB를 바꾸는 유일한 경로 |
| 설계도 | `rdb-schema` | `dbdiagram.dbml`. 팀이 함께 보는 다이어그램이며 **마이그레이션 도구가 아니다** |
| dev 배포 시 적용 | `pulsemetry-backend` | EKS enrollment-api가 먼저 Flyway를 적용한다. telemetry-ingest는 `spring.flyway.enabled=false`이며 enrollment DDL을 적용하지 않는다 |
| dev 읽기 소비자 | `pulsemetry-backend` | telemetry-ingest·dashboard-api는 공유 RDS `controlplane`의 `enrollment`를 읽는다. dashboard의 별도 캐시 DDL은 [dashboard 계약](dashboard-api.md)을 따른다 |
| legacy 로컬 부트스트랩 | `ai-telemetry-pipeline` | `sql/rds/schema.sql`·`seed.sql`. **편의용이며 진실원이 아니고 dev EKS의 적용 절차도 아니다** |
| legacy 읽기 소비자 | `ai-telemetry-pipeline` | auth-proxy(`DATABASE_URL`), telemetry-processor(`ENRICHMENT_PG_DSN`)의 과거 조사 경계. 이 전체 경로가 prod에 배포됐다는 뜻은 아니다 |

> **스키마를 바꿔야 하면 backend Flyway를 고친다.** dbml과 파이프라인 DDL은 뒤따라 맞춘다.
> 세 곳이 갈라진 것이 과거 E2E 차단 결함 B3의 한 원인이었다. 현재 dev 구현·live 검증 구분은 [`telemetry-ingest.md`](telemetry-ingest.md) §5를 따른다.

## 2. 물리 타입 — enum

상태값은 **PostgreSQL native enum**으로 표현한다(ADR-0009가 ADR-0004의 varchar + CHECK를 대체).

당시 타입 결정의 배경은 공유 DB였다. legacy 파이프라인 소비자(auth-proxy·enricher)는 상태값을 **텍스트로만 비교**하므로
어느 물리 타입이든 동작하지만, backend는 Hibernate `ddl-auto: validate`와 `@Enumerated(STRING)` 때문에
물리 타입에 민감하다. 두 DDL이 병존하면 backend가 공유 DB의 기존 스키마 위에서 기동하지 못한다.

**enum 10종**

| enum | 값 |
|---|---|
| `tenant_status` | `active` · `suspended` · `terminated` |
| `team_status` | `active` · `archived` |
| `member_role` | `owner` · `admin` · `member` |
| **`member_status`** | **`invited` · `active` · `suspended`** |
| `installation_status` | `active` · `revoked` |
| `platform_type` | `windows` · `macos` · `linux` |
| `ai_vendor` | `anthropic` · `openai` · `google` |
| `contract_type` / `contract_status` / `token_type` | `rdb-schema/dbdiagram.dbml` 참조 |

값을 추가·제거하는 것은 **계약 변경**이다. 양쪽 레포 담당자의 합의가 필요하다.

## 3. 핵심 엔티티

아래 엔티티 설명은 기존 모델 조사 내용을 보존한다. 특히 `members`의 인증 구현 상태 서술은 당시 조사 기준이며
현행 인증 상태 판정에 사용하지 않는다. PROJ-250은 세부 인증·스키마 정책을 변경하지 않는다.

```
tenants ──┬── teams ──── team_memberships ──── members
          ├── members ──┬── invitations
          │             └── installations ──┬── installation_credentials
          │                                 ├── telemetry_tokens
          │                                 └── installation_manifest_assignments
          ├── manifests
          └── contracts ──┬── contract_term_commitments
                          ├── contract_token_discounts
                          └── contract_memberships
```

| 테이블 | 뜻 | 계약상 중요한 점 |
|---|---|---|
| `tenants` | 조직 | 모든 데이터의 테넌트 경계 |
| `teams` | 팀 | 약속하는 **최소 집계 단위**([`../product/prd.md`](../product/prd.md) §4) |
| `members` | 구성원 | 웹 사용자(관리자) 인증은 backend Spring Security가 AT·RT를 직접 발급한다(backend ADR-0007 — Cognito 미사용. **구현은 아직 없다** — 현행 관리자 API 인증은 정적 `X-Admin-Token`이다). 비밀번호는 `password_hash`(NULL이면 가입 미완료)에 담고, `cognito_user_sub`는 제거됐다(backend `V4`). 일반 사용자는 installation을 통해서만 서비스와 연결된다. **구성원 식별은 `(tenant_id, email)` 유니크 하나뿐이다** |
| **`team_memberships`** | 소속 **관계와 기간**(`joined_at` · `left_at`) | **as-of 조인의 근거.** 소속은 시점 함수다 — "지금 어느 팀인가"가 아니라 "그 이벤트 시각에 어느 팀이었나"로 귀속한다 |
| `installations` | 설치 인스턴스 | 데스크탑의 인증 주체. 사람 계정과 다르다 |
| `telemetry_tokens` | `ptt_` 토큰의 **해시**만 | 해시 방식은 [`enrollment-api.md`](enrollment-api.md) §4가 소유 |
| `manifests` | 테넌트별 OTel 설정, 버전 행 누적 | `is_active = true`는 tenant당 최대 하나 |
| `contracts` 계열 | 조직 계약 단가 | **조회 시 2차 가공**의 입력. 파이프라인은 읽지 않는다 |

## 4. 계약상 불변식

| # | 규칙 | 어겼을 때 |
|---|---|---|
| D-1 | **DDL을 바꾸는 유일한 경로는 backend Flyway다.** 파이프라인 `sql/rds/*`와 `dbdiagram.dbml`은 뒤따른다 | 공유 DB에서 backend가 기동하지 못하거나 소비자 쿼리가 깨진다 |
| D-2 | **파이프라인은 `enrollment` 스키마에 쓰지 않는다.** 읽기 전용 소비자다 | 쓰기 주체가 둘이 되어 정합성이 깨진다 ([`../architecture/overview.md`](../architecture/overview.md) I-6) |
| D-3 | **팀 귀속은 `team_memberships`의 as-of 조인으로 한다.** 현재 소속으로 과거 이벤트를 귀속하지 않는다 | 사람이 팀을 옮기면 과거 비용이 새 팀으로 옮겨간다 |
| D-4 | **`member_status`의 값 집합은 토큰 검증의 거부 로직과 묶여 있다.** 값을 추가하면 dev backend의 검증과 legacy auth-proxy 소비 여부를 함께 확인한다 | 새 상태의 멤버가 조용히 401이 된다 |
| D-5 | 토큰·초대 코드의 **원문은 어떤 컬럼에도 넣지 않는다.** `*_hash` 컬럼에는 해시만 | 오브젝트·DB 유출이 곧 자격증명 유출 (I-11) |

## 5. 미해결

| # | 항목 |
|---|---|
| B3 | 과거의 별도 로컬 Postgres·enrollment 미배포 가정은 현행 dev 코드의 설명이 아니다. infra generator는 EKS enrollment·ingest·dashboard에 공유 RDS `controlplane` 입력을 공급한다. 실제 DB 연결·마이그레이션·E2E 확인은 남아 있다 — [`telemetry-ingest.md`](telemetry-ingest.md) §5 |
| — | **부트스트랩 주체는 backend Flyway다**(backend ADR-0009). dev는 enrollment-api를 먼저 배포해 Flyway 적용을 확인한 뒤 telemetry-ingest·dashboard-api를 배포한다. ingest Flyway는 비활성화하고 dashboard는 자기 캐시 DDL을 적용한다. 로컬 `bootRun`은 dev EKS의 잠정 배포 절차가 아니다. prod 마이그레이션 실행 자리는 별도 결정이며, 파이프라인 DDL을 psql로 직접 넣는 우회는 사용하지 않는다 |
| — | Signal Database(ClickHouse) 쪽 스키마는 이 계약의 범위 밖이다. [`telemetry-ingest.md`](telemetry-ingest.md)의 `enriched_events` 설명은 legacy 조사 기록이며, 현행 backend 스키마를 그 설명으로 대체하지 않는다 |
