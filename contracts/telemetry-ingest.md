# 계약 — telemetry ingest (OTLP 전송·인증·신원 전파)

| 항목 | 내용 |
|---|---|
| 당사자 | **`telemetryctl`** (데몬 forwarder) → **`ai-telemetry-pipeline`** (auth-proxy · collector · processor), 배포 설정은 **`infra`** |
| 관련 ADR | **허브 [ADR-0001](../adr/0001-otlp-authentication-model.md)(OTLP 인증 모델 — infra ADR-0008을 대체)** / infra ADR-0017(collector config 주입), ADR-0023(dev auth-proxy) / telemetryctl ADR-0001(인라인 프록시 토폴로지) |
| 상태 | 확정 — **미해결 배선 1건**(§5 B3) |

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
dev/배포 collector 설정은 각각 유지하되 **신원 전파 3요소는 PROJ-77로 동일해졌다**(§5 B4 해소).
나머지 차이(gRPC 리시버·`create_directory`·exporter endpoint)는 환경 차이다.

## 2. 로컬 수신기 (AI tool → 데몬)

- **HTTP 전용.** `127.0.0.1` + `[::1]` 이중 리슨. 기본 4318, 점유 시 ephemeral 폴백.
- 인증은 **3중 AND**: 상수 시간 Bearer 비교 + `X-Pulsemetry-Local: 1` 헤더 + OPTIONS 거부. 실패 사유는 로그에만 남긴다.
- 큐 포화 시 **429가 아니라 200 + PartialSuccess(rejected=0)** 로 응답한다. 벤더 exporter의 재시도 폭주를 막기 위한 **의도된 드롭 정책**이다.
- 여기 실리는 토큰은 **로컬 ingest 토큰**이며 회사 `ptt_`가 아니다([`enrollment-api.md`](enrollment-api.md) §3).

## 3. 상위 전송 인증 (forwarder → auth-proxy) — ★ 핵심 계약

**OTLP `Authorization`에 실리는 값은 `ptt_`다** — 토큰 종류는 확정이다([ADR-0001](../adr/0001-otlp-authentication-model.md)).
검증 **주체**는 별개 축이다: 현행은 auth-proxy(`ai-telemetry-pipeline`)지만 **폐기 예정**이며,
backend ADR-0007의 collector 이관과 함께 backend Spring Security 계층으로 이관된다.
manifest revision 대조는 이 경로에 없다 — 불투명 토큰에는 클레임이 없고, 대조는 관리자 API 경로의 몫이다.

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

> **규칙 — 두 collector 설정 파일의 신원 전파 3요소는 함께 바꾼다.**
> `infra/config/otel-collector.yaml`(ECS 기동)과 `ai-telemetry-pipeline/otel-collector-config.yaml`(dev in-repo)은
> 의도된 두 벌 구성이며, `include_metadata` · `headers_setter/pulsemetry_tenant` · `batch.metadata_keys` 셋은
> 반드시 같은 값이어야 한다. 한쪽만 고쳐 실제로 갈라진 이력이 B4다(PROJ-77로 복구).
> 자동 검증 장치는 두지 않는다 — 다시 갈라진 사실이 발견되면 그때 감지 장치를 재검토한다.

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
| ~~B4~~ | **해소됨(PROJ-77).** `infra/config/otel-collector.yaml`에 신원 전파 3요소가 모두 반영됐다(`04cd98e`). 두 파일의 드리프트가 재발 원인이므로 collector 설정 변경 시 두 파일을 함께 본다(§4의 규칙) | `infra/config/otel-collector.yaml` | — |
| M2 | **processor가 `x-pulsemetry-member-id`를 받고도 버린다.** 스탬핑 대상이 tenant/installation뿐이다. `Identity.member_id`는 클라이언트 자칭 `developer.email`/`developer.id` 폴백인데, **이 속성을 발신하는 코드가 세 레포 어디에도 없다** | `otlp_receiver.py:48-51` | member 귀속을 인증 헤더 기반으로 바꾼다. 현재 실트래픽에서 `member_id`는 항상 `None` |
| M3 | `developer.*`·`tenant.id` 리소스 속성 발신자 부재. telemetryctl이 `OTEL_RESOURCE_ATTRIBUTES`를 배선하지 않고, manifest `resource_attributes`는 Codex `environment` 한 키에만 쓰인다. `repository_allowlist`는 완전히 사장 — 미집행 필드의 지위는 [`enrollment-api.md`](enrollment-api.md) §5의 「집행되지 않는 manifest 필드」가 소유한다 | `config/claude.go`, `codex.go:51-54` | M2와 함께 결정한다 — 신원을 토큰 파생으로 통일하면 발신자는 필요 없다 |
| ~~M4~~ | **해소됨(PROJ-79).** `org.py`의 psycopg 접근이 `OperationalError`를 `BackendUnavailable`로 변환해 RDS 장애가 503(재시도 가능)으로 분류된다(pipeline `5c9c59e`) — 레포 문서 5곳의 "RDS/ClickHouse 장애는 503" 서술이 참이 됐다 | `providers/org.py`의 `OrgProvider._load()` | — |
| M5 | `pair_call_ids`가 push 단위로만 동작. collector batch가 `tool_decision`/`tool_result`를 갈라놓으면 승인율 KPI가 왜곡된다 | `normalize.py:128`, `call_id.py:46-57` | |
| M6 | **metrics 파이프라인에 `redaction/secrets` 미적용**(dev·배포 공통) — 메트릭 속성의 시크릿이 그대로 적재된다 | 두 collector 설정의 metrics 파이프라인 | |
| M11 | processor가 무인증·무TLS·`0.0.0.0:8080`·바디 무제한인데 compose가 호스트에 노출한다. 시드에 원본 토큰이 커밋돼 있고 `TOKEN_HASH_SECRET` 기본값이 고정이다 | `ai-telemetry-pipeline`의 `otlp_receiver.py`, `sql/rds/seed.sql`의 telemetry_tokens 블록 | **구 파이프라인에만 남은 항목이다.** 대체 경로인 `:apps:telemetry-ingest`는 인증이 가장 앞이고(§8) 원본 바디에 상한이 있으며, backend `LocalSeeder`는 원문 토큰을 저장하지 않는다. 구 파이프라인이 내려가면 함께 사라진다(PROJ-106) |
| M12 | collector 이미지가 `:latest`. 파일 아카이브가 무한 append(Fargate 20GiB 소진 시 태스크 사망 — 인프라 주석이 자인) | `.github/workflows/deploy_dev.yml`, infra 설정 주석 | **태그 고정 승인됨(PROJ-79, 실행 대기)** — 현재 구동 버전으로 고정하기로 했다. 실행 계획은 infra ADR-0017 Follow-up. 구동 버전 확인(AWS)이 선행이며, 고정되면 이 행을 해소 표기한다 |
| M13 | **강등(회사 직결) 경로의 manifest `privacy` 집행 공백.** grpc 테넌트·키링 실패로 강등되면(§6) 벤더가 회사 Collector로 직결돼 1차 집행 지점인 포워더 `Scrub`(§7)이 경로 밖이 된다. 집행은 벤더 설정 계층으로 되돌아가는데, manifest 연결이 Claude는 6필드 중 5, **Codex는 `log_user_prompt` 1필드뿐**이고 `collect_user_email`은 양 벤더 모두 미집행이다. collector `redaction/secrets`는 `allow_all_keys: true`라 이 공백을 메우지 않는다 | telemetryctl `installer/apply.go` 강등 분기, `config/claude.go`·`codex.go`, 두 collector 설정 | **벤더별 설정의 privacy 매핑을 6필드 전부로 확장한다**(Codex에 대응 설정 표면이 실재하는지 확인 선행). 매핑이 불가능한 필드는 이 계약에 그 사실을 명시한다. telemetryctl ADR 0006 Follow-up이 실행 항목을 소유한다 |

**B3가 남아 있는 한 E2E는 성립하지 않는다.** B3는 신호가 들어가느냐를 깨뜨린다.
신원 귀속(B4)은 PROJ-77로, RDS 장애 분류(M4)는 PROJ-79으로 해소됐다.

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
- **`otlp.protocol = grpc`는 계약상 유효하지만 클라이언트가 배선하지 못한다.** telemetryctl의
  forwarder는 grpc 상위 전달을 지원하지 않으므로(`forward.ErrGRPCUnsupported`) grpc 테넌트는 로컬
  파이프라인 배선에서 제외되고 회사 직결로 강등된다 — 로컬 대시보드에 데이터가 쌓이지 않는다.
  grpc 테넌트를 구성하기 전에 이 제약을 확인한다(telemetryctl ADR 0001·0006, [`enrollment-api.md`](enrollment-api.md) §6 M9).
  현재 grpc로 구성된 테넌트는 없다. **강등 상태에서는 §7의 1차 집행 지점(포워더 `Scrub`)이
  경로 밖이 된다** — 그 privacy 집행 공백은 §5 M13이 소유한다.

## 7. 프라이버시 집행 지점

원문(프롬프트·assistant 응답)과 tool details가 상위로 새지 않게 막는 계층은 셋이고, **맡는 일이 서로 다르다.**

| 계층 | 위치 | 맡는 것 | 덮지 않는 것 |
|---|---|---|---|
| **포워더 `Scrub` — 1차 집행 지점** | telemetryctl `internal/otlpdecode` | 회사 manifest `privacy` 기준 **원문·tool details 제거**. `privacyRules`가 `Privacy` 6필드와 1:1 대응 | **회사 직결 강등 경로**(§6 grpc·키링 실패) — 포워더가 경로 밖이라 벤더 설정 계층만 남는다(§5 M13) |
| Collector `redaction/secrets` | 두 collector 설정 | **시크릿 값 마스킹**(API 키·JWT·PEM 등 정규식) | 원문 속성 제거(`allow_all_keys: true`) · **metrics 파이프라인**(§5 M6) |
| 정규화 어댑터 allowlist | `ai-telemetry-pipeline` normalizer | `Prompt` payload를 길이·명령 이름으로 한정(pipeline ADR 0001) | collector의 **원본 아카이브(`file/*`) 경로** — 정규화 이전 페이로드가 그대로 남는다 |

manifest 기준의 원문·tool details 제거는 **로컬 파이프라인이 배선된 상태에서는 포워더 한 곳에서만**
일어난다 — 상위 두 계층은 다른 일을 하므로 포워더 `Scrub`의 회귀는 상위에서 걸러지지 않는다.
**회사 직결로 강등된 설치에는 포워더 자체가 없다**(§6). 그때 집행은 벤더 설정 계층으로 되돌아가며,
그 계층의 manifest 연결은 Claude 5/6 · Codex 1/6 · `collect_user_email` 0/2로 불완전하다 — §5 M13.
회사 수집 범위를 바꾸는 변경은 telemetryctl의 골든 픽스처 테스트가 그 경로의 방어선임을 전제로 리뷰한다.
(telemetryctl ADR 0003·0006의 "회사 manifest 준수는 전적으로 `internal/forward`가 집행한다"는 단언도
로컬 파이프라인이 배선된 경로에 한한다 — 강등 경로의 예외는 §5 M13과 ADR 0006 Follow-up이 소유한다.)

## 8. 상태 코드와 재시도 — `:apps:telemetry-ingest`

근거는 [ADR 0006](../adr/0006-otlp-ingest-retry-and-status-contract.md)이다.

> **이 절은 새 경로의 계약이다.** 배포된 경로는 아직 auth-proxy → collector → processor 이고
> §1·§3·§4가 그 현행을 서술한다. 두 서술이 어긋나 보이는 것은 전환이 진행 중이기 때문이며,
> 앞의 절들을 고치는 시점은 [ADR 0005](../adr/0005-single-app-telemetry-topology.md) Follow-up이
> **infra가 collector 컨테이너를 내리는 때**로 못박았다.

| 상태 | 언제 | 데몬의 처분 |
|---|---|---|
| `200` | 성공. 레코드 0건도 성공이다 | — |
| `401` | 토큰이 없거나 검증에 실패했다. 사유 열한 가지가 **하나의 본문**으로 접힌다(§3) | 토큰 무효화 후 재발급·재시도 **1회** |
| `400` | 디코드·압축 해제 실패(깨진 gzip·protobuf 포함), **그리고 영구 실패** — 정규화가 실패했거나, 보강 단계가 RDS 스키마 드리프트 같은 영구 오류를 만났거나, ClickHouse가 요청을 거부(4xx)했다 | **즉시 폐기** |
| `405`·`415`·`404` | 메서드·`Content-Type`·경로가 계약 밖이다. 본문은 `text/plain` | 즉시 폐기 |
| `413` | 압축 전 원본 바디가 상한을 넘었다 | 즉시 폐기 |
| `503` + `Retry-After` | 일시 장애 — RDS·ClickHouse에 닿지 못했거나, ClickHouse 스키마가 아직 적용되지 않았거나, 아카이브에 실패했다. 분류되지 않은 예외의 기본값도 503이다 | 재시도 |

- **영구 실패가 4xx인 이유는 데몬이 4xx만 폐기하기 때문이다.** `classify()`는 5xx를 전부
  재시도하므로 500으로는 폐기가 만들어지지 않는다. 이 표는 데몬의 판정에 맞춰 서버를 정한
  것이며, **telemetryctl은 이 결정으로 바뀌지 않는다.**
- **서버에 큐도 내부 재시도도 없다.** 버퍼는 데몬의 큐(64건·32 MiB, 논블로킹 드롭)이고,
  변환 이후가 실패해도 원본은 아카이브에 남아 재처리의 원천이 된다.
- **`Retry-After`는 하한이다.** 데몬은 자기 백오프와 `max`를 취하고 15초에서 자른다.
  크게 잡으면 3회 예산을 기다림으로 태운다.
- **`429`는 쓰지 않는다.** 데몬과 프록시 양쪽이 이미 지원하는 채널이지만 서버가 낸 적이 없다.
  도입 조건은 ADR 0006 Follow-up이 소유한다.
- **정규화·보강의 영구 오류도 400이다.** 같은 입력은 재시도해도 같고, 원본은 아카이브에 남아
  재처리의 원천이 된다. 보강 모듈 자체는 연결 계열만 일시 장애로 감싸고 나머지를 전파하며, 조립 앱이
  그 전파된 예외 중 `NonTransientDataAccessException`(자원 계열 제외)을 400으로 매핑한다 —
  매핑 표는 backend `IngestPipeline` KDoc 하나다.
- **인증이 가장 앞이다.** 401은 405·415보다 먼저 난다 — 통과한 요청만 수집 단계에 닿아야
  거부될 데이터가 외부 저장소에 적재되지 않는다([ADR 0005](../adr/0005-single-app-telemetry-topology.md)).
