# 계약 — 공유 도메인 모델 (tenant / team / member)

| 항목 | 내용 |
|---|---|
| 당사자 | **`pulsemetry-backend`**(쓰기·DDL 진실원) ↔ **`ai-telemetry-pipeline`**(읽기 소비자). 설계도는 **`rdb-schema`** |
| 물리 위치 | 공유 RDS `controlplane`의 **`enrollment` 스키마** |
| 관련 ADR | backend ADR-0004(진실원 = Flyway), ADR-0009(native enum) / infra ADR-0012(컨트롤 플레인 DB) |
| 상태 | 확정 |

파이프라인이 신호를 조직·팀·사람에 귀속시키려면 backend가 쓴 테이블을 읽어야 한다.
**그 테이블의 정의가 이 계약이다.**

## 1. 소유권

| 역할 | 레포 | 내용 |
|---|---|---|
| **DDL 진실원** | `pulsemetry-backend` | `libs/enrollment-persistence`의 **Flyway 마이그레이션**. 운영 DB를 바꾸는 유일한 경로 |
| 설계도 | `rdb-schema` | `dbdiagram.dbml`. 팀이 함께 보는 다이어그램이며 **마이그레이션 도구가 아니다** |
| dev 부트스트랩 | `ai-telemetry-pipeline` | `sql/rds/schema.sql`·`seed.sql`. **편의용이며 진실원이 아니다** |
| 소비자 | `ai-telemetry-pipeline` | auth-proxy(`DATABASE_URL`), telemetry-processor(`ENRICHMENT_PG_DSN`) — **읽기만** 한다 |

> **스키마를 바꿔야 하면 backend Flyway를 고친다.** dbml과 파이프라인 DDL은 뒤따라 맞춘다.
> 세 곳이 갈라진 것이 E2E 차단 결함 B3의 절반이었다([`telemetry-ingest.md`](telemetry-ingest.md) §5).

## 2. 물리 타입 — enum

상태값은 **PostgreSQL native enum**으로 표현한다(ADR-0009가 ADR-0004의 varchar + CHECK를 대체).

이유는 공유 DB 때문이다. 파이프라인 소비자(auth-proxy·enricher)는 상태값을 **텍스트로만 비교**하므로
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
| `members` | 구성원 | 웹 사용자(관리자) 인증은 backend Spring Security가 AT·RT를 직접 발급한다(backend ADR-0007 — Cognito 미사용. **구현은 아직 없다** — 현행 관리자 API 인증은 정적 `X-Admin-Token`이다). `cognito_user_sub`는 폐기 예정 컬럼이다(제거 마이그레이션은 그 ADR Follow-up). 일반 사용자는 installation을 통해서만 서비스와 연결된다. `(tenant_id, email)` 유니크 |
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
| D-4 | **`member_status`의 값 집합은 auth-proxy의 거부 로직과 묶여 있다.** 값을 추가하면 auth-proxy를 함께 고친다 | 새 상태의 멤버가 조용히 401이 된다 |
| D-5 | 토큰·초대 코드의 **원문은 어떤 컬럼에도 넣지 않는다.** `*_hash` 컬럼에는 해시만 | 오브젝트·DB 유출이 곧 자격증명 유출 (I-11) |

## 5. 미해결

| # | 항목 |
|---|---|
| B3 | dev ECS에 enrollment 서버가 미배포이고, 로컬에서 backend와 파이프라인이 서로 다른 Postgres를 본다. 공유 RDS `controlplane`을 양쪽이 보는 구성으로 수렴시켜야 한다 — [`telemetry-ingest.md`](telemetry-ingest.md) §5 |
| — | **부트스트랩 주체는 backend Flyway로 확정됐다**(backend ADR-0009). enrollment 서버가 dev에 배포되기 전까지는 backend 명세 §9.4의 로컬 `bootRun` 레시피가 **공식 잠정 절차**다 — 파이프라인 DDL을 psql로 직접 넣는 우회는 폐지됐다. 남은 것은 enrollment 서버의 dev 배포와 ECS에서 마이그레이션을 실행할 자리(infra 새 ADR 예정)다 |
| — | Signal Database(ClickHouse) 쪽 스키마는 이 계약의 범위 밖이다. `enriched_events`의 컬럼 계약은 [`telemetry-ingest.md`](telemetry-ingest.md)가 다룬다 |
