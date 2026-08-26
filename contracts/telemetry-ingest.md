# 계약 — telemetry ingest (OTLP 전송·인증·신원 전파)

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (데몬 forwarder) → **`ai-telemetry-pipeline`** (auth-proxy · collector · processor), 배포 설정은 **`infra`** |
| 관련 ADR | infra ADR-0008(인증 이원화), ADR-0017(collector config 주입), ADR-0023(dev auth-proxy) / telemetryctl ADR-0001(인라인 프록시 토폴로지) |
| 상태 | 확정 — **미해결 배선 2건**(§5) |

데스크탑/CLI에서 발생한 OTLP 신호가 회사 파이프라인에 들어가 **누구 것인지 알 수 있는 상태로** 적재되기까지의 계약이다.

## 1. 경로

```
Claude Code / Codex ── OTLP/HTTP + Bearer <로컬 ingest 토큰> + X-Pulsemetry-Local: 1
  ──▶ 데몬 로컬 수신기 (127.0.0.1:4318, HTTP 전용)
        ├─ SQLite 정규화·집계 (개인용, ~/.pulsemetry/pulsemetry.db)
        └─ forwarder: manifest.signals 게이팅 + privacy 스크럽 → 원본 인코딩 유지 재전송
  ──▶ POST {manifest.otlp.endpoint}/v1/{logs|metrics|traces} + Authorization: Bearer <ptt_>
  ──▶ auth-proxy ── HMAC-SHA256(token) → enrollment.telemetry_tokens 조회
        → 통과 시 x-pulsemetry-* 4헤더 부여, Authorization 제거
  ──▶ otel-collector ── redaction(logs/traces) + batch (+ 제품별 파일 아카이브)
  ──▶ telemetry-processor (OTLP/JSON push)
        ── 신원 스탬핑 → claude_code. / codex. 어댑터 정규화
        ── org provider: enrollment.team_memberships as-of 조인 → team_ids_as_of
  ──▶ ClickHouse enriched_events (JSONEachRow)
```

dev ECS 배포 경로는 ALB `:80 /v1/*` → auth-proxy → `collector.obs.local:4318` → 같은 태스크의 processor(`localhost:8080`).
**구조는 동일하되 collector 설정 파일이 다르다** — 이것이 §5 B4의 원인이다.

## 2. 로컬 수신기 (AI tool → 데몬)

- **HTTP 전용.** `127.0.0.1` + `[::1]` 이중 리슨. 기본 4318, 점유 시 ephemeral 폴백.
- 인증은 **3중 AND**: 상수 시간 Bearer 비교 + `X-Pulsemetry-Local: 1` 헤더 + OPTIONS 거부. 실패 사유는 로그에만 남긴다.
- 큐 포화 시 **429가 아니라 200 + PartialSuccess(rejected=0)** 로 응답한다. 벤더 exporter의 재시도 폭주를 막기 위한 **의도된 드롭 정책**이다.
- 여기 실리는 토큰은 **로컬 ingest 토큰**이며 회사 `ptt_`가 아니다([`enrollment-api.md`](enrollment-api.md) §3).

## 3. 상위 전송 인증 (forwarder → auth-proxy) — ★ 핵심 계약

**forwarder가 싣는 것은 `Authorization: Bearer <ptt_>` 하나뿐이다.**
디바이스·사용자 식별 헤더도, 신원 리소스 속성도 보내지 않는다.
**신원 귀속은 전적으로 토큰에서 파생된다.** 이것이 이 계약의 중심 결정이다 — 클라이언트가 자칭한 신원을 신뢰하지 않는다.

| 항목 | 값 |
|---|---|
| 경로 | `/v1/logs` · `/v1/metrics` · `/v1/traces` |
| 헤더 | `Authorization: Bearer ptt_…` |
| auth-proxy의 파싱 | `/^Bearer\s+([^\s]+)$/i` |
| 토큰 조회 | **HMAC-SHA256(`TOKEN_HASH_SECRET`, 토큰 전문)** hex → `enrollment.telemetry_tokens.token_hash` |
| 거부 조건 | 토큰 미존재(`token_unknown`) / 멤버 상태 `invited`·`suspended`(`member_invited` 등) |
| 통과 시 | 아래 4헤더 부여 후 `Authorization` **제거** |

**auth-proxy가 부여하는 신원 헤더 4종**

| 헤더 | 내용 |
|---|---|
| `x-pulsemetry-token-id` | 검증된 토큰의 id |
| `x-pulsemetry-tenant-id` | 테넌트(조직) |
| `x-pulsemetry-installation-id` | 설치 인스턴스 |
| `x-pulsemetry-member-id` | **인증으로 얻은 유일한 위조 불가 사용자 식별자** |

> **해시 방식과 멤버 상태 전환은 [`enrollment-api.md`](enrollment-api.md) §3·§4가 소유한다.**
> 이 두 지점이 갈라지면 발급된 모든 토큰이 401이 된다.

**회복 루프** — forwarder가 401/403을 받으면 토큰을 무효화하고
`POST /v1/installations/telemetry-token`(`pit_` 인증, 10s 데드라인)으로 **1회 재발급 후 재시도**한다.
재발급도 같은 해시 방식으로 발급되므로, 해시가 어긋난 상태에서는 이 루프가 복구를 만들지 못한다.

## 4. 신원 전파 (auth-proxy → collector → processor) — ★ 가장 자주 깨지는 지점

collector가 헤더를 통과시키려면 **세 요소가 모두** 있어야 한다.

| # | 요소 | 역할 |
|---|---|---|
| 1 | receiver의 `include_metadata: true` | 들어온 HTTP 헤더를 메타데이터로 보존 |
| 2 | `headers_setter/pulsemetry_tenant` 확장 | 아웃바운드 요청에 그 메타데이터를 헤더로 다시 실음 |
| 3 | `batch`의 `metadata_keys` 4종 | 배치가 서로 다른 신원의 요청을 섞지 않게 분리 |

셋 중 하나만 빠져도 **헤더가 collector에서 소실되고 스탬핑이 no-op이 된다.**
그러면 인증은 통과했는데 ClickHouse의 `tenant_id`·`installation_id`가 빈 문자열이 되고 `team_ids_as_of`는 빈 배열이 된다.

**processor의 스탬핑 규칙**

| 리소스 속성 | 출처 헤더 |
|---|---|
| `tenant.id` | `x-pulsemetry-tenant-id` |
| `developer.installation_id` | `x-pulsemetry-installation-id` |

스탬핑은 **클라이언트 자칭 값을 덮어쓴다.** 이것이 I-11(자격증명 대신 파생 식별자로 귀속)의 구현 지점이다.

## 5. 미해결 — ★ E2E를 막고 있는 항목

| # | 결함 | 근거 | 수정 방향 |
|---|---|---|---|
| **B3** | **enrollment DB 배선 단절.** 로컬에서 backend는 자기 Postgres(`pulsemetry`@5432)에, auth-proxy는 파이프라인 Postgres(`enrichment`@55432)에 붙는다. **같은 DB를 보는 구성이 어디에도 없다.** dev ECS에는 enrollment 서버가 아예 미배포다 | backend `docker-compose.yml` vs pipeline `docker-compose.dev.yml`; infra ADR-0023 Follow-up | 공유 RDS `controlplane`을 양쪽이 보게 한다. **DDL 진실원은 backend Flyway**(ADR-0004·0009). 파이프라인 `sql/rds/schema.sql`은 dev 편의용 소비자로 격하. dev ECS에 enrollment 서버 배포(ECR·태스크 정의부터 부재) |
| **B4** | **배포 collector 설정에 §4의 3요소가 없다.** `infra/config/otel-collector.yaml`(ECS가 `--config=env:OTEL_CONFIG`로 실제 기동하는 파일)에 `include_metadata`·`headers_setter`·`batch.metadata_keys`가 전부 빠져 있다. dev in-repo 설정에는 있다 | `infra/config/otel-collector.yaml` vs `ai-telemetry-pipeline/otel-collector-config.yaml:8,11-24,30-34` | infra 설정에 3요소를 이식한다. **두 파일의 드리프트가 원인이므로 한쪽만 고치면 재발한다** |
| M2 | **processor가 `x-pulsemetry-member-id`를 받고도 버린다.** 스탬핑 대상이 tenant/installation뿐이다. `Identity.member_id`는 클라이언트 자칭 `developer.email`/`developer.id` 폴백인데, **이 속성을 발신하는 코드가 세 레포 어디에도 없다** | `otlp_receiver.py:48-51` | member 귀속을 인증 헤더 기반으로 바꾼다. 현재 실트래픽에서 `member_id`는 항상 `None` |
| M3 | `developer.*`·`tenant.id` 리소스 속성 발신자 부재. telemetryctl이 `OTEL_RESOURCE_ATTRIBUTES`를 배선하지 않고, manifest `resource_attributes`는 Codex `environment` 한 키에만 쓰인다. `repository_allowlist`는 완전히 사장 | `config/claude.go`, `codex.go:51-54` | M2와 함께 결정한다 — 신원을 토큰 파생으로 통일하면 발신자는 필요 없다 |
| M4 | **Postgres 장애가 400(영구 오류)으로 분류돼 collector가 배치를 폐기한다.** enrichment 실패가 곧 데이터 유실이 된다 | `providers/org.py:56` → `otlp_receiver.py:139` | 일시 오류를 5xx로 분류해 재시도시킨다 |
| M5 | `pair_call_ids`가 push 단위로만 동작. collector batch가 `tool_decision`/`tool_result`를 갈라놓으면 승인율 KPI가 왜곡된다 | `normalize.py:128`, `call_id.py:46-57` | |
| M6 | **metrics 파이프라인에 `redaction/secrets` 미적용**(dev·배포 공통) — 메트릭 속성의 시크릿이 그대로 적재된다 | 두 collector 설정의 metrics 파이프라인 | |
| M11 | processor가 무인증·무TLS·`0.0.0.0:8080`·바디 무제한인데 compose가 호스트에 노출한다. 시드에 원본 토큰이 커밋돼 있고 `TOKEN_HASH_SECRET` 기본값이 고정이다 | `otlp_receiver.py:118-158`, `seed.sql:83-90` | |
| M12 | collector 이미지가 `:latest`. 파일 아카이브가 무한 append(Fargate 20GiB 소진 시 태스크 사망 — 인프라 주석이 자인) | `.github/workflows/deploy_dev.yml`, infra 설정 주석 | |

**B3·B4가 남아 있는 한 E2E는 성립하지 않는다.** B3는 신호가 들어가느냐를, B4는 들어간 신호가 누구 것인지 아느냐를 깨뜨린다.

## 6. 정합이 확인된 부분

문제는 구간 내부가 아니라 경계에 몰려 있다. 아래는 세 레포가 필드 단위로 일치하는 것들이다.

- OTLP 경로(`/v1/logs|metrics|traces`)와 Bearer 헤더 형식.
- 토큰 형식(`pit_`/`ptt_` + base64url(32B))과 auth-proxy 정규식의 호환.
- manifest 스키마 — telemetryctl `contracts/*.schema.json`, backend `ManifestPayload` 검증, 파이프라인 시드 형식.
- 정규화 어댑터 — Claude Code(`claude_code.` prefix)와 Codex(`codex.` prefix) 모두 구현돼 있고 이벤트·속성 키 매핑이 실제 벤더 신호 형태와 부합한다.
- forwarder는 **원본 바이트 인코딩을 유지**해 재전송하므로 `service.name` 등 벤더 값이 보존된다.
  collector의 `filter/codex`·`filter/claude_code`는 **파일 아카이브 경로에만** 적용되고 실시간 경로는 전량 전달한다.
- ClickHouse 적재 — `enriched_events`(ReplacingMergeTree, ORDER BY event_id), `INSERT … FORMAT JSONEachRow`.
  `record_id`가 멱등 키 역할을 해 I-10 방향과 일치한다(단 `ts`는 초 단위로 절사된다).

### 비대칭으로 남은 것

- Codex 로그는 `attrs["event.name"]`으로, Claude Code는 body 문자열로 매칭한다.
- Codex는 벤더가 비용을 주지 않아 단가표로 추정하며, `pricing.py`의 단가는 **placeholder**다.
  `cost_source` 외에는 실측과 구분할 수단이 없다.
