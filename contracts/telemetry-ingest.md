# 계약 — telemetry ingest (OTLP 전송·인증·신원 전파)

| 항목 | 내용 |
|---|---|
| 당사자 | dev: **`telemetryctl`** (데몬 forwarder) → **`pulsemetry-backend`** (`telemetry-ingest`), 배포 설정은 **`infra`**. 기존 `ai-telemetry-pipeline` 조사는 아래에서 별도 구분 |
| 관련 ADR | **허브 [ADR-0001](../adr/0001-otlp-authentication-model.md)(OTLP 인증 모델 — infra ADR-0008을 대체)** / [ADR 0023](../adr/0023-dev-backend-eks-only.md)(dev EKS 전용 배포) / infra ADR-0017(collector config 주입), ADR-0023(과거 dev auth-proxy) / telemetryctl ADR-0001(인라인 프록시 토폴로지) |
| 상태 | 변경 중 — PROJ-250의 dev EKS 전용 계약 정합화. 관련 구현 merge 전이며 실제 배포·B3 live 검증 완료와 별도 |

데스크탑/CLI에서 발생한 OTLP 신호가 회사 파이프라인에 들어가 **누구 것인지 알 수 있는 상태로** 적재되기까지의 계약이다.

## 1. 경로

### dev 구현·배포 계약

[ADR 0023](../adr/0023-dev-backend-eks-only.md)에 따라 dev 백엔드는 EKS를 단독 실행 환경으로 사용한다.
PROJ-248의 infra 로컬 구현과 PROJ-249의 backend 로컬 workflow 변경을 반영한 계약이며,
현재 AWS 배포 상태나 E2E 통과를 뜻하지 않는다.

```
AI tool → 데몬 로컬 수신기 → forwarder (manifest.otlp.endpoint, Bearer ptt_)
  → dev EKS ALB → backend telemetry-ingest (:4316)
      ├─ Spring Security 토큰 검증·신원 귀속 ← 공유 RDS controlplane
      └─ 수집·변환·보강·적재 → S3 / EKS ClickHouse
```

OTLP 경로는 `/v1/logs`·`/v1/metrics`·`/v1/traces`를 유지한다. dev는 auth-proxy·OTel Collector·
별도 processor 컨테이너를 거치지 않고 backend의 한 프로세스에서 처리한다.
enrollment-api가 공유 RDS의 Flyway를 먼저 적용한 뒤 telemetry-ingest와 dashboard-api를 배포한다.
telemetry-ingest의 `spring.flyway.enabled`는 `false`이며, 배포 순서와 검증은
[infra EKS 런북](../../infra/docs/runbooks/dev-eks-manual-deployment.md)을 따른다.

아래 legacy 경로와 §3·§4의 auth-proxy 상세, §5·§6의 기존 조사 결과, §7의 collector·normalizer 설명은 과거 파이프라인 조사 범위다.
§7의 forwarder `Scrub`은 dev EKS 전송에서도 유지되는 클라이언트 프라이버시 책임이다.
이 문서 정리는 세부 인증·OTLP·정규화·스키마·프라이버시 정책을 개정하거나 그 항목을 재검증하지 않는다.
prod에 Collector·processor가 남는다는 사실만으로 아래 auth-proxy 전체 경로가 prod에 배포됐다고 해석하지 않는다.

### legacy 파이프라인 조사 경로

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

당시 dev ECS 조사 경로는 ALB `:80 /v1/*` → auth-proxy → `collector.obs.local:4318` → 같은 태스크의 processor(`localhost:8080`)였다.
당시 dev/배포 collector 설정의 **신원 전파 3요소는 PROJ-77로 동일해졌다**(§5 B4 해소 기록).
나머지 차이(gRPC 리시버·`create_directory`·exporter endpoint)는 당시 환경 차이이며 현행 dev 경로가 아니다.

## 2. 로컬 수신기 (AI tool → 데몬)

- **HTTP 전용.** `127.0.0.1` + `[::1]` 이중 리슨. 기본 4318, 점유 시 ephemeral 폴백.
- 인증은 **3중 AND**: 상수 시간 Bearer 비교 + `X-Pulsemetry-Local: 1` 헤더 + OPTIONS 거부. 실패 사유는 로그에만 남긴다.
- 큐 포화 시 **429가 아니라 200 + PartialSuccess(rejected=0)** 로 응답한다. 벤더 exporter의 재시도 폭주를 막기 위한 **의도된 드롭 정책**이다.
- 여기 실리는 토큰은 **로컬 ingest 토큰**이며 회사 `ptt_`가 아니다([`enrollment-api.md`](enrollment-api.md) §3).

## 3. 상위 전송 인증 (legacy auth-proxy 조사)

**OTLP `Authorization`에 실리는 값은 `ptt_`다** — 토큰 종류는 확정이다([ADR-0001](../adr/0001-otlp-authentication-model.md)).
검증 **주체**는 별개 축이다. 아래 표는 당시 auth-proxy(`ai-telemetry-pipeline`) 구현을 조사한 기록이며,
dev 구현의 검증 주체는 §1의 backend Spring Security 계층이다.
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

## 4. 신원 전파 (legacy auth-proxy → collector → processor 조사)

collector가 헤더를 통과시키려면 **세 요소가 모두** 있어야 한다.

| # | 요소 | 역할 |
|---|---|---|
| 1 | receiver의 `include_metadata: true` | 들어온 HTTP 헤더를 메타데이터로 보존 |
| 2 | `headers_setter/pulsemetry_tenant` 확장 | 아웃바운드 요청에 그 메타데이터를 헤더로 다시 실음 |
| 3 | `batch`의 `metadata_keys` 4종 | 배치가 서로 다른 신원의 요청을 섞지 않게 분리 |

셋 중 하나만 빠져도 **헤더가 collector에서 소실되고 스탬핑이 no-op이 된다.**
그러면 인증은 통과했는데 ClickHouse의 `tenant_id`·`installation_id`가 빈 문자열이 되고 `team_ids_as_of`는 빈 배열이 된다.

> **legacy collector 경로의 규칙 — 두 설정 파일의 신원 전파 3요소는 함께 바꾼다.**
> `infra/config/otel-collector.yaml`(prod ECS 기동)과 `ai-telemetry-pipeline/otel-collector-config.yaml`(파이프라인 로컬 실험)은
> 의도된 두 벌 구성이며, `include_metadata` · `headers_setter/pulsemetry_tenant` · `batch.metadata_keys` 셋은
> 반드시 같은 값이어야 한다. 한쪽만 고쳐 실제로 갈라진 이력이 B4다(PROJ-77로 복구).
> 자동 검증 장치는 두지 않는다 — 다시 갈라진 사실이 발견되면 그때 감지 장치를 재검토한다.

**processor의 스탬핑 규칙**

| 리소스 속성 | 출처 헤더 |
|---|---|
| `tenant.id` | `x-pulsemetry-tenant-id` |
| `developer.installation_id` | `x-pulsemetry-installation-id` |

스탬핑은 **클라이언트 자칭 값을 덮어쓴다.** 이것이 I-11(자격증명 대신 파생 식별자로 귀속)의 구현 지점이다.

## 5. B3의 구현·검증 경계와 legacy 조사 항목

B3는 현재 로컬 구현과 live 검증을 나누어 기록한다. 그 밖의 B4·M 항목은 과거 파이프라인 조사 결과를
보존한 것이며, 표의 현재성 표현·결함·해소 기록을 현행 dev backend나 prod의 검증 결과로 사용하지 않는다.

| # | 결함 | 근거 | 수정 방향 |
|---|---|---|---|
| **B3** | **과거 조사:** backend와 별도 파이프라인의 로컬 Postgres가 분리돼 있었고 당시 dev ECS에는 enrollment 서버가 없었다. **현재 로컬 구현:** EKS enrollment·ingest와 dashboard 원천 연결은 공유 RDS `controlplane`을 사용한다. **live 검증:** 실제 연결·마이그레이션·enrollment→OTLP E2E는 별도 확인이 필요하다 | infra `scripts/eks-values.mjs`의 앱 DB URL 생성, EKS 런북; backend enrollment/ingest `application.yaml` | enrollment-api의 Flyway 선행 적용 후 ingest·dashboard를 배포한다. ingest Flyway는 비활성화한다. 실제 RDS·스키마와 수집 E2E 증거를 확인하기 전 B3를 배포 완료로 닫지 않는다 |
| ~~B4~~ | **해소됨(PROJ-77).** `infra/config/otel-collector.yaml`에 신원 전파 3요소가 모두 반영됐다(`04cd98e`). 두 파일의 드리프트가 재발 원인이므로 collector 설정 변경 시 두 파일을 함께 본다(§4의 규칙) | `infra/config/otel-collector.yaml` | — |
| M2 | **processor가 `x-pulsemetry-member-id`를 받고도 버린다.** 스탬핑 대상이 tenant/installation뿐이다. `Identity.member_id`는 클라이언트 자칭 `developer.email`/`developer.id` 폴백인데, **이 속성을 발신하는 코드가 세 레포 어디에도 없다** | `otlp_receiver.py:48-51` | member 귀속을 인증 헤더 기반으로 바꾼다. 현재 실트래픽에서 `member_id`는 항상 `None` |
| M3 | `developer.*`·`tenant.id` 리소스 속성 발신자 부재. telemetryctl이 `OTEL_RESOURCE_ATTRIBUTES`를 배선하지 않고, manifest `resource_attributes`는 Codex `environment` 한 키에만 쓰인다. `repository_allowlist`는 완전히 사장 — 미집행 필드의 지위는 [`enrollment-api.md`](enrollment-api.md) §5의 「집행되지 않는 manifest 필드」가 소유한다 | `config/claude.go`, `codex.go:51-54` | M2와 함께 결정한다 — 신원을 토큰 파생으로 통일하면 발신자는 필요 없다 |
| ~~M4~~ | **해소됨(PROJ-79).** `org.py`의 psycopg 접근이 `OperationalError`를 `BackendUnavailable`로 변환해 RDS 장애가 503(재시도 가능)으로 분류된다(pipeline `5c9c59e`) — 레포 문서 5곳의 "RDS/ClickHouse 장애는 503" 서술이 참이 됐다 | `providers/org.py`의 `OrgProvider._load()` | — |
| M5 | `pair_call_ids`가 push 단위로만 동작. collector batch가 `tool_decision`/`tool_result`를 갈라놓으면 승인율 KPI가 왜곡된다 | `normalize.py:128`, `call_id.py:46-57` | |
| M6 | **metrics 파이프라인에 `redaction/secrets` 미적용**(dev·배포 공통) — 메트릭 속성의 시크릿이 그대로 적재된다 | 두 collector 설정의 metrics 파이프라인 | |
| M11 | processor가 무인증·무TLS·`0.0.0.0:8080`·바디 무제한인데 compose가 호스트에 노출한다. 시드에 원본 토큰이 커밋돼 있고 `TOKEN_HASH_SECRET` 기본값이 고정이다 | `otlp_receiver.py:118-158`, `seed.sql:83-90` | |
| M12 | collector 이미지가 `:latest`. 파일 아카이브가 무한 append(Fargate 20GiB 소진 시 태스크 사망 — 인프라 주석이 자인) | `.github/workflows/deploy_dev.yml`, infra 설정 주석 | **태그 고정 승인됨(PROJ-79, 실행 대기)** — 현재 구동 버전으로 고정하기로 했다. 실행 계획은 infra ADR-0017 Follow-up. 구동 버전 확인(AWS)이 선행이며, 고정되면 이 행을 해소 표기한다 |
| M13 | **강등(회사 직결) 경로의 manifest `privacy` 집행 공백.** grpc 테넌트·키링 실패로 강등되면(§6) 벤더가 회사 Collector로 직결돼 1차 집행 지점인 포워더 `Scrub`(§7)이 경로 밖이 된다. 집행은 벤더 설정 계층으로 되돌아가는데, manifest 연결이 Claude는 6필드 중 5, **Codex는 `log_user_prompt` 1필드뿐**이고 `collect_user_email`은 양 벤더 모두 미집행이다. collector `redaction/secrets`는 `allow_all_keys: true`라 이 공백을 메우지 않는다 | telemetryctl `installer/apply.go` 강등 분기, `config/claude.go`·`codex.go`, 두 collector 설정 | **벤더별 설정의 privacy 매핑을 6필드 전부로 확장한다**(Codex에 대응 설정 표면이 실재하는지 확인 선행). 매핑이 불가능한 필드는 이 계약에 그 사실을 명시한다. telemetryctl ADR 0006 Follow-up이 실행 항목을 소유한다 |

B3의 과거 배선 단절을 현재 코드의 부재로 반복하지 않는다. 공유 RDS 입력과 배포 순서는 구현돼 있으나,
실제 연결·DDL·기능 증거 없이 E2E 완료를 주장하지 않는다. B4(PROJ-77)·M4(PROJ-79)의 해소 기록은 legacy 조사 범위다.

## 6. legacy 조사에서 정합이 확인된 부분

아래는 당시 세 레포를 필드 단위로 대조한 기록이다. 현행 backend의 정규화·적재 스키마나 실트래픽 상태를 재검증한 결과가 아니다.

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

## 7. 프라이버시 집행 지점 — 현행 클라이언트와 legacy 서버 조사

forwarder `Scrub`은 상위 서버의 실행 환경과 무관하게 회사 manifest의 `privacy`를 집행한다.
로컬 파이프라인이 배선된 dev EKS 전송에도 적용되며, 스크럽 실패 시 해당 배치를 전송하지 않는다.
아래 표의 collector·normalizer와 두 계층의 한계는 과거 파이프라인 조사 범위다. dev backend의 현행 집행 정책을 이 표로 대체하지 않는다.

현행 클라이언트와 과거 서버의 프라이버시 집행 책임은 다음과 같이 구분한다.

| 계층 | 위치 | 맡는 것 | 덮지 않는 것 |
|---|---|---|---|
| **포워더 `Scrub` — 1차 집행 지점** | telemetryctl `internal/otlpdecode` | 회사 manifest `privacy` 기준 **원문·tool details 제거**. `privacyRules`가 `Privacy` 6필드와 1:1 대응 | **회사 직결 강등 경로**(§6 grpc·키링 실패) — 포워더가 경로 밖이라 벤더 설정 계층만 남는다(§5 M13) |
| Collector `redaction/secrets` | 두 collector 설정 | **시크릿 값 마스킹**(API 키·JWT·PEM 등 정규식) | 원문 속성 제거(`allow_all_keys: true`) · **metrics 파이프라인**(§5 M6) |
| 정규화 어댑터 allowlist | `ai-telemetry-pipeline` normalizer | `Prompt` payload를 길이·명령 이름으로 한정(pipeline ADR 0001) | collector의 **원본 아카이브(`file/*`) 경로** — 정규화 이전 페이로드가 그대로 남는다 |

과거 서버 조사에서 상위 두 계층은 manifest 기준의 원문·tool details 제거를 대신하지 못했다.
현행 dev EKS 경로에서도 클라이언트는 상위 서버에 의존하지 않고 전송 전에 forwarder `Scrub`을 적용한다.
**회사 직결로 강등된 설치에는 포워더 자체가 없다**(§6). 그때 집행은 벤더 설정 계층으로 되돌아가며,
그 계층의 manifest 연결은 Claude 5/6 · Codex 1/6 · `collect_user_email` 0/2로 불완전하다 — §5 M13.
회사 수집 범위를 바꾸는 변경은 telemetryctl의 골든 픽스처 테스트가 그 경로의 방어선임을 전제로 리뷰한다.
(telemetryctl ADR 0003·0006의 "회사 manifest 준수는 전적으로 `internal/forward`가 집행한다"는 단언도
로컬 파이프라인이 배선된 경로에 한한다 — 강등 경로의 예외는 §5 M13과 ADR 0006 Follow-up이 소유한다.)
