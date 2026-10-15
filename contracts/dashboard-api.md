# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | `pulsemetry-frontend` ↔ `pulsemetry-backend` ↔ `infra` |
| 상태 | 확정 — runtime·저장소·배포 책임은 [ADR 0021](../adr/0021-dashboard-runtime-and-deployment-contract.md), 사용자 JWT 키 공급은 [ADR 0022](../adr/0022-user-jwt-key-provisioning.md) |
| API 구현 | `pulsemetry-backend/apps/dashboard-api` (`:apps:dashboard-api`, 기본 포트 **8081**) |

이 문서는 조회/명령 소유, runtime 설정과 배포 전제를 정한다. HTTP 필드·오류·페이지네이션 상세는
backend의 `docs/api/README.md`, `docs/api/reference/`와 frontend의 `docs/api/`가 소유한다.
문서 채택과 실제 DB 권한·이미지 공급·AWS 배포·live E2E 완료는 별도 증거다.

## 1. 애플리케이션 경계

- dashboard-api는 RDS와 ClickHouse의 원천을 읽고 **자신의 `dashboard_cache`만 쓴다**.
  조직·구성원·정책·계약·초대·좌석·알림 확인 명령은 enrollment-api가 맡는다.
- Signal 원천 쓰기와 raw S3 접근은 수집/복구 경계에 남긴다. dashboard에 원천 변경 권한,
  enrollment 관리자 토큰, token hash HMAC key, JWT private key를 전달하지 않는다.
- 조직 계약 적용 금액은 조회 시 계산한다. 공시 기준 금액과 계약 적용 금액을 구분한다.
  조회 한도를 넘거나 snapshot 구축이 실패하면 부분 결과를 정상 총액으로 공개하지 않는다.
- 시나리오 가용성은 조회 시 판정하고, 시나리오는 지표를 새로 만들지 않는다.
  화면 링크의 `?scenario={id}` 규칙은 [제품 정의](../product/prd.md)를 따른다.

## 2. 공개 경로와 method 소유

모든 업무 API 접두는 `/api/v1`이다. 표의 `O`는 **`/api/v1/organizations/{organizationId}`**다.
`D`는 dashboard-api, `E`는 enrollment-api다. 동일 URI라도 method가 다르면 다른 소유자다.
특히 **`GET /api/v1/auth/me`, `GET O/onboarding`, `GET O/invitations`는 E**에 남는다.

### 2-1. dashboard 조회

각 행의 일반 요청은 `GET`이며, 유효한 CORS preflight의 `OPTIONS`와
`Access-Control-Request-Method: GET`도 D로 보낸다.

| Path | Method | 소유 | 구현 근거 |
|---|---|---|---|
| `O/analytics/overview` | GET | D | `OverviewController` |
| `O/analytics/teams` | GET | D | `TeamsController` |
| `O/analytics/teams/{teamId}` | GET | D | `TeamsController` |
| `O/analytics/teams/{teamId}/users` | GET | D | `TeamsController` |
| `O/teams` | GET | D | `TeamsController` |
| `O/members/dashboard` | GET | D | `MembersController` |
| `O/members` | GET | D | `MembersController` |
| `O/members/unassigned` | GET | D | `MembersController` |
| `O/seat-reclaim-candidates` | GET | D | `MembersController` |
| `O/members/{memberId}/seats` | GET | D | `MembersController` |
| `O/vendors/{vendorId}/seats` | GET | D | `MembersController` |
| `O/contract-input-status` | GET | D | `SettingsController` |
| `O/settings` | GET | D | `SettingsController` |
| `O/vendors` | GET | D | `SettingsController` |
| `O/vendors/{vendorId}` | GET | D | `SettingsController` |
| `O/installations` | GET | D | `SettingsController` |
| `O/alerts` | GET | D | `AlertController` |
| `O/alerts/{alertId}` | GET | D | `AlertController` |
| `O/operations/{operationId}` | GET | D | `OperationController` |
| `O/ingest-status` | GET | D | `IngestStatusController` |
| `/api/v1/vendor-catalog` | GET | D | `VendorCatalogController` |
| `/api/v1/vendor-catalog/{vendorId}/plans` | GET | D | `VendorCatalogController` |

### 2-2. enrollment 조직 조회·명령

유효한 CORS preflight의 `OPTIONS`는 각 행의 method를
`Access-Control-Request-Method`에 지정한 경우 E로 보낸다.
관리 기능이 꺼진 경우의 API 미제공은 앱에서 판정한다.

| Path | Method | 소유 | 구현 근거 |
|---|---|---|---|
| `O/onboarding` | GET | E | `ManagementController` |
| `O/invitations` | GET | E | `ManagementController` |
| `O/teams` | POST | E | `ManagementController` |
| `O/teams/{teamId}` | PATCH, DELETE | E | `ManagementController` |
| `O/member-team-assignments` | POST | E | `ManagementController` |
| `O/members/{memberId}` | PATCH | E | `ManagementController` |
| `O/invitations/batch` | POST | E | `ManagementController` |
| `O/invitations/{invitationId}/revoke` | POST | E | `ManagementController` |
| `O/invitations/{invitationId}/reissue` | POST | E | `ManagementController` |
| `O/members/{memberId}/installation-invitations` | POST | E | `ManagementController` |
| `O/vendors` | POST | E | `ManagementController` |
| `O/vendors/{vendorId}` | PATCH, DELETE | E | `ManagementController` |
| `O/vendors/{vendorId}/contract` | PUT, DELETE | E | `ManagementController` |
| `O/collection-policy` | PUT | E | `ManagementController` |
| `O/onboarding/complete` | POST | E | `ManagementController` |
| `O/installation-update-notifications` | POST | E | `ManagementController` |
| `O/vendors/{vendorId}/connection` | PUT, DELETE | E | `VendorConnectionController` |
| `O/vendors/{vendorId}/connection/verify` | POST | E | `VendorConnectionController` |
| `O/vendors/{vendorId}/connection/sync` | POST | E | `ManagementController` |
| `O/vendors/{vendorId}/seats` | POST | E | `ManagementController` |
| `O/vendors/{vendorId}/seats/import` | POST | E | `ManagementController` |
| `O/vendors/{vendorId}/seats/{seatId}` | PATCH | E | `ManagementController` |
| `O/vendors/{vendorId}/seats/{seatId}/release` | POST | E | `ManagementController` |
| `O/seat-reclaims/preview` | POST | E | `ManagementController` |
| `O/seat-reclaims` | POST | E | `ManagementController` |
| `O/seat-reclaims/{operationId}/restore` | POST | E | `ManagementController` |
| `O/operations/{operationId}/targets/{targetId}/confirm` | POST | E | `ManagementController` |
| `O/operations/{operationId}/targets/{targetId}/cancel` | POST | E | `ManagementController` |
| `O/settings/alert-rules/{ruleId}` | PATCH | E | `AlertRuleController` |
| `O/alerts/{alertId}/acknowledge` | POST | E | `AlertRuleController` |

### 2-3. 기존 enrollment·인증·설치 경로

| Path | Method | 소유 | OPTIONS preflight |
|---|---|---|---|
| `/api/v1/auth/organizations` | POST | E | E, 요청 method=POST |
| `/api/v1/auth/token` | POST | E | E, 요청 method=POST |
| `/api/v1/auth/cli/token` | POST | E | E, 요청 method=POST |
| `/api/v1/auth/refresh` | POST | E | E, 요청 method=POST |
| `/api/v1/auth/logout` | POST | E | E, 요청 method=POST |
| `/api/v1/auth/me` | GET | E | E, 요청 method=GET |
| `/api/v1/auth/oidc/authorize` | GET | E | 브라우저 이동용, CORS API로 추가하지 않음 |
| `/api/v1/auth/oidc/callback/{registrationId}` | GET | E | 브라우저 callback, CORS API로 추가하지 않음 |
| `/api/v1/manifest` | GET | E | E, 요청 method=GET |
| `/api/v1/enroll` | POST | E | 현재 앱 CORS 등록 없음 |
| `/api/v1/installations/telemetry-token` | POST | E | 현재 앱 CORS 등록 없음 |
| `/api/v1/installations/{installationId}/heartbeat` | POST | E | 현재 앱 CORS 등록 없음 |
| `/api/v1/invitations` | POST | E | 현재 앱 CORS 등록 없음 |
| `/api/v1/invitations/{id}/revoke` | POST | E | 현재 앱 CORS 등록 없음 |
| `/api/v1/inquiries` | POST | E | E, 요청 method=POST, 문의 기능/출처 설정 사용 |
| `/api/v1/check-updates` | GET | E | 현재 앱 CORS 등록 없음 |
| `/windows` | GET | E | 현재 앱 CORS 등록 없음 |
| `/unix` | GET | E | 현재 앱 CORS 등록 없음 |
| `/bin/{filename}` | GET | E | 현재 앱 CORS 등록 없음 |

인증 CORS는 `EnrollmentSecurityConfig`에서 사용자 인증이 켜진 경우에 등록한다.
OIDC 두 경로는 `OidcLoginConfig`의 전용 security chain이 처리한다.
폐기된 password login/authorize 경로는 새 routing 표에 넣지 않는다.

### 2-4. edge·OPTIONS·health 경계

- ECS와 EKS ALB 모두 같은 표를 적용한다. `GET /api/v1/organizations/*`처럼 조직 조회를
  통째로 D에 넘기지 않고, 명시 path와 method를 조합한다. 미등록 path/method의 default는 **404**다.
- 공유 URI의 preflight는 `OPTIONS`만으로 소유자를 고를 수 없다. path + OPTIONS +
  `Access-Control-Request-Method`로 나눈다. 예: `O/teams` GET preflight는 D, POST preflight는 E다.
  헤더가 없거나 지원하지 않는 method인 OPTIONS는 default 404다.
- dashboard CORS는 GET/OPTIONS, 허용 헤더 `Authorization`, `Content-Type`, `X-Request-Id`,
  노출 헤더 `X-Request-Id`, `Retry-After`, `ETag`, `allowCredentials=false`다. origin은 명시 목록이다.
  enrollment 쓰기 CORS는 추가로 `Idempotency-Key`, `If-Match`와 `Location`을 사용한다.
- `GET /api/v1/healthz`는 각 앱의 **내부** health 경로다. target group이 자신의 서비스로 확인하며
  public listener rule을 만들지 않는다. dashboard health는 생존 확인이고 저장소/인증 기능 검증을 대신하지 않는다.
- `/v1/traces`, `/v1/metrics`, `/v1/logs`는 기존 telemetry-ingest 소유다. dashboard 추가로 옮기지 않는다.
  Spring의 암묵적 HEAD 응답을 새 public method 계약으로 확대하지 않는다.

## 3. 네 저장소 연결과 DDL 책임

source/cache 설정은 별도 연결·별도 계정으로 전달한다. 앱의 readOnly 옵션만으로 권한을 대신하지 않는다.

| 연결 | DB 범위와 필요한 권한 | 책임 |
|---|---|---|
| RDS source | `enrollment`, `telemetry_ops` schema USAGE + 필요한 원천 테이블 SELECT. 새 테이블도 마이그레이션 소유자의 default privileges로 전달 | infra가 계정·권한 준비; dashboard `SourceStoreConfig`의 primary DataSource/JPA/JdbcClient가 읽음 |
| RDS cache | `dashboard_cache` 소유/DDL·읽기·쓰기; V1 cache 마이그레이션 최초 적용에 필요한 database CREATE; `telemetry_ops` USAGE + `tenant_retention_boundary` SELECT | infra가 schema·계정 준비; dashboard `RdsCacheSchema.migrate()`가 전용 Flyway 실행 |
| ClickHouse source | 분석 테이블·ledger SELECT만. 계정 profile의 readonly는 요청별 `readonly=2`와 상한 설정을 허용해야 함 (`readonly=1` 고정 금지) | infra가 계정·권한 준비; dashboard 원천 reader가 query 시간·행·byte 한도 적용 |
| ClickHouse cache | cache DB의 DDL·읽기·쓰기 + snapshot 복사에 필요한 원천 `telemetry_events`, `telemetry_metric_points` SELECT | infra가 DB·계정 준비; dashboard가 멱등 table/view DDL과 `INSERT ... SELECT` 실행 |

RDS source/cache는 **같은 PostgreSQL logical database**에 연결한다. cache 공개 CAS가 같은 SQL 안에서
`telemetry_ops.tenant_retention_boundary`를 읽기 때문이다. ClickHouse source/cache도 **같은 ClickHouse server**에
연결한다. 캐시 연결의 `INSERT ... SELECT`가 source database를 식별자로 한정해 원천을 읽는다.
URL·계정이 두 묶음이라는 사실만으로 다른 서버/DB 배치를 허용하지 않는다.

`enrollment`·`telemetry_ops`의 원천 Flyway는 enrollment-api가 먼저 실행한다.
Dashboard는 `spring.flyway.enabled=false`, `spring.jpa.hibernate.ddl-auto=validate`이며 원천을 변경하지 않는다.
기동 시 `CacheSchemaInitializer`가 RDS cache Flyway(`classpath:db/dashboard-cache`,
`dashboard_cache.flyway_schema_history`) 뒤 ClickHouse cache 멱등 DDL(`clickhouse/dashboard-cache`)을 적용한다.
실패하면 기동을 실패시킨다. DB 자체와 초기 권한은 infra가 준비하고 캐시 테이블/뷰·마이그레이션은 backend가 소유한다.
현재 RDS V1의 `CREATE SCHEMA IF NOT EXISTS dashboard_cache`는 schema가 미리 있어도 최초 적용 시 database CREATE를 요구한다.
V1의 성공 이력이 있으면 매 기동마다 이 문장이 다시 실행되는 것은 아니다. 이 추가 권한의 유지/회수는
다음 마이그레이션의 필요 권한과 함께 infra가 관리한다.

알림 확인 명령은 enrollment-api가 dashboard cache 알림 존재를 검증하므로 enrollment 계정에
`dashboard_cache` USAGE + `alerts` SELECT가 필요하다. dashboard 캐시 기동 후 이 grant를 확인한다.
그 권한을 이유로 dashboard에 enrollment의 쓰기 계정·master 계정을 재사용하지 않는다.

## 4. runtime 설정 인터페이스

아래 표는 dashboard `application.yaml`의 실제 env 매핑이다. `p`는 `pulsemetry.dashboard`다.
**모든 `PULSEMETRY_DASHBOARD_*` 운영 수치/연결값은 PORT를 제외하면 기본값이 없다.**
infra는 배포 환경의 주소·계정·측정한 운영 한도를 전달하고 backend는 매핑·형식 검증·소비 코드를 소유한다.
비밀번호는 코드가 빈 문자열을 허용하므로 DB 정책에 맞춰 공급하며, 빈 비밀번호를 기동 검증이 막는다고 가정하지 않는다.

| 환경변수 | property | 기본값 / 검증·전달 책임 |
|---|---|---|
| `PULSEMETRY_DASHBOARD_PORT` | `server.port` | 8081; ECS container port와 Helm targetPort 일치 |
| `PULSEMETRY_USER_AUTH_ENABLED` | `pulsemetry.user-auth.enabled` | false; 실제 인증 조회를 제공하려면 true |
| `PULSEMETRY_USER_AUTH_ISSUER` | `pulsemetry.user-auth.issuer` | 빈 값; enabled=true일 때 발급 서버와 일치하는 nonblank 값 |
| `PULSEMETRY_USER_AUTH_AUDIENCE` | `pulsemetry.user-auth.audience` | 빈 값; enabled=true일 때 발급 서버와 일치하는 nonblank 값 |
| `PULSEMETRY_USER_AUTH_PUBLIC_KEY_FILE` | `pulsemetry.user-auth.public-key-files.local` | 빈 값; kid=local 전용 경로. 배포의 임의 kid는 명시적 `public-key-files` 맵 사용 |
| `PULSEMETRY_USER_AUTH_ALLOWED_ORIGINS` | `pulsemetry.user-auth.allowed-origins` | 빈 목록; frontend의 실제 origin 목록 |
| `PULSEMETRY_MANAGEMENT_ENABLED` | `pulsemetry.management.enabled` | false; enrollment 기능과 맞추는 UI capability |
| `PULSEMETRY_MAIL_ENABLED` | `pulsemetry.mail.enabled` | false; enrollment와 일치, 안내 가능 여부 표시만 함 |
| `PULSEMETRY_DASHBOARD_RETRY_AFTER` | `p.retry-after` | 1초 이상 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_URL` | `p.clickhouse.source.url` | nonblank, 원천 server |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_DATABASE` | `p.clickhouse.source.database` | `[A-Za-z_][A-Za-z0-9_]*` |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_USERNAME` | `p.clickhouse.source.username` | nonblank, SELECT 계정 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_PASSWORD` | `p.clickhouse.source.password` | DB 정책에 맞춰 공급 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_QUERY_TIMEOUT` | `p.clickhouse.source.query-timeout` | 1초 이상 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_MAX_RESULT_ROWS` | `p.clickhouse.source.max-result-rows` | 1 이상 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_SOURCE_MAX_RESULT_BYTES` | `p.clickhouse.source.max-result-bytes` | 1 이상 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_CACHE_URL` | `p.clickhouse.cache.url` | nonblank, source와 같은 server |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_CACHE_DATABASE` | `p.clickhouse.cache.database` | 식별자 형식, infra가 준비한 cache DB |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_CACHE_USERNAME` | `p.clickhouse.cache.username` | nonblank, cache + 복사용 원천 SELECT 계정 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_CACHE_PASSWORD` | `p.clickhouse.cache.password` | DB 정책에 맞춰 공급 |
| `PULSEMETRY_DASHBOARD_CLICKHOUSE_CACHE_QUERY_TIMEOUT` | `p.clickhouse.cache.query-timeout` | 1초 이상 |
| `PULSEMETRY_DASHBOARD_RDS_SOURCE_URL` | `p.rds.source.url` | nonblank JDBC URL |
| `PULSEMETRY_DASHBOARD_RDS_SOURCE_USERNAME` | `p.rds.source.username` | nonblank, 원천 SELECT 계정 |
| `PULSEMETRY_DASHBOARD_RDS_SOURCE_PASSWORD` | `p.rds.source.password` | DB 정책에 맞춰 공급 |
| `PULSEMETRY_DASHBOARD_RDS_CACHE_URL` | `p.rds.cache.url` | nonblank JDBC URL, source와 같은 logical DB |
| `PULSEMETRY_DASHBOARD_RDS_CACHE_USERNAME` | `p.rds.cache.username` | nonblank, cache 계정 |
| `PULSEMETRY_DASHBOARD_RDS_CACHE_PASSWORD` | `p.rds.cache.password` | DB 정책에 맞춰 공급 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_BUILD_TIMEOUT` | `p.snapshot.build-timeout` | 1초 이상 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_PURGE_GRACE` | `p.snapshot.purge-grace` | 0 이상 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_MAX_CONCURRENT_BUILDS` | `p.snapshot.max-concurrent-builds` | 1 이상 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_MAX_COPY_ROWS` | `p.snapshot.max-copy-rows` | 1 이상 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_MAX_COPY_BYTES` | `p.snapshot.max-copy-bytes` | 1 이상 |
| `PULSEMETRY_DASHBOARD_SNAPSHOT_CLEANUP_INTERVAL` | `p.snapshot.cleanup-interval` | 1초 이상 |
| `PULSEMETRY_DASHBOARD_MEMBERS_IDLE_DAYS` | `p.members.idle-days` | 7·14·30·60 중 하나 |
| `PULSEMETRY_DASHBOARD_INGEST_WINDOW` | `p.ingest.window` | 1분 이상, 정수 분 단위 |
| `PULSEMETRY_DASHBOARD_INGEST_DELAYED_AFTER` | `p.ingest.delayed-after` | 0보다 큼 |
| `PULSEMETRY_DASHBOARD_INGEST_DOWN_AFTER` | `p.ingest.down-after` | delayed-after보다 큼 |
| `PULSEMETRY_DASHBOARD_COMPLETENESS_SETTLE_AFTER` | `p.completeness.settle-after` | 0보다 큼 |
| `PULSEMETRY_DASHBOARD_SEATS_STALE_AFTER` | `p.seats.stale-after` | 0보다 큼 |
| `PULSEMETRY_DASHBOARD_ALERTS_EVALUATION_INTERVAL` | `p.alerts.evaluation-interval` | 0보다 큼 |
| `PULSEMETRY_DASHBOARD_ALERTS_LEASE` | `p.alerts.lease` | 0보다 큼 |

두 RDS 연결의 `connection-timeout`은 YAML의 고정 **3s**이며 대응 env는 없다.
운영자는 ingest window를 설치 heartbeat 주기보다, seats stale-after를 enrollment 동기화 주기보다,
alerts lease를 한 평가 회차보다 길게 정한다. completeness settle-after는 전송 재시도·적재 완료를 포괄한다.
이 운영 조건은 모두 cross-property 기동 검증으로 강제되는 것은 아니다.

### 인증키와 현재 세션

`PUBLIC_KEY_FILE`은 **`kid=local` 맵 항목**의 파일만 바꾼다. 파일명 변경으로 JWT kid가 바뀌지 않는다.
발급 서버가 `local` 이외 kid를 사용하면 `pulsemetry.user-auth.public-key-files`에 **정확한 kid → 파일 경로**를
YAML 또는 `SPRING_APPLICATION_JSON`으로 명시한다. 회전 기간에는 허용 kid별 공개키를 함께 배포한다.
검증키는 X.509 PEM RSA 공개키이며 2048bit 이상이다. 원격 jku/x5u에서 키를 가져오지 않는다.

키 공급의 권위는 [ADR 0022](../adr/0022-user-jwt-key-provisioning.md)다.

| 항목 | 공통 계약 |
|---|---|
| 원본 | 환경별 개인키·공개키 **고정 ARN 두 개**의 Secrets Manager Secret. `aws/secretsmanager` 암호화. dev ECS/EKS는 같은 원본 |
| 값 형식 | 개인키는 RSA 2048bit 이상 PKCS8 PEM; 공개키는 정확한 `kid → X.509 PEM` JSON 키링 |
| 최초 생성 | 명시적 초기화에서 키쌍·kid 생성. 두 Secret 등록과 시험 서명/검증 뒤 배포 참조 확정. 부분 등록 실패는 같은 키로 복구 |
| 소비 권한 | enrollment는 개인키·공개키, dashboard는 공개키만. ingest에는 JWT Secret 권한 없음 |
| 파일 전달 | infra가 선택 버전의 개인키와 키링의 각 공개키를 별도 PEM 파일로 공급. 앱에는 경로·kid 맵 전달 |
| 배포 기록 | 두 ARN/`VersionId`·enrollment `active-kid`·양쪽 앱의 공개키 파일 맵을 함께 기록. dashboard는 개인키 참조를 소비하지 않음 |
| 보존 | 활성·승인된 롤백 버전을 staging label로 보존. 두 Secret의 `AWSPREVIOUS`를 한 쌍으로 가정하지 않음 |
| 적용 | 버전 지정 파일을 명시적 재배포로 읽음. 자동 회전·자동 reload 없음. 로컬 Compose 키는 배포 원본이 아님 |

실제 full ARN은 계정 ID를 포함하므로 제한된 배포 입력·기록에서 관리하고 저장소 문서·예시·공개 출력에는
placeholder를 쓴다. 실제 키 원문이나 계정 ID를 이 계약 문서에 기록하지 않는다.

공개키 Secret JSON 자체는 PEM 파일이 아니다. 공급 단계가 각 값을 파일로 변환하고
`pulsemetry.user-auth.public-key-files`의 kid별 경로를 구성한다. YAML 또는 `SPRING_APPLICATION_JSON`에는
PEM 원문을 넣지 않는다. 선택한 키링 전체의 누락·빈 파일·읽기 불가·잘못된 PEM·키쌍/kid 불일치는
배포 실패 조건이다. 현행 로더의 빈 경로 필터링을 완전한 키링 준비 검사로, modulus 일치 검사를
시험 서명/검증 완료로 취급하지 않는다.

교체는 **모든 enrollment·dashboard verifier에 신·구 공개키 적용 → 모든 enrollment signer의
개인키/active-kid 전환 → 마지막 구 키 발급 후 330초 이상 대기 → 구 공개키 제거** 순서다.
330초는 `UserJwt.lifetime` 300초와 시간 허용 오차 30초를 합한 값이며 전환 시작 시점으로 재지 않는다.
dev ECS/EKS의 남은 구 replica까지 verifier 적용을 확인하기 전 signer를 전환하지 않는다.
부분 실패는 겹치는 공개키 상태를 유지하고 중단한다. 제거 후 롤백은 겹치는 공개키를 모든 verifier에
먼저 복구한 뒤 구 signer로 전환하며, 신 AT도 마지막 신 키 발급 후 330초 이상 검증되게 신 공개키를 유지한다.

Dashboard `SecurityConfig.dashboardAuthenticator()`는 `UserJwt.verifier()`로 서명/issuer/audience를 확인한 뒤
`UserAccessVerifier.verify()`로 현재 `enrollment.user_sessions`, `members`, `tenants`를 읽어
세션 만료·폐기, 구성원/조직 활성 상태, 조직·role 일치를 확인한다.
이 경로는 source RDS SELECT 계정으로 동작한다. 서명만 검증하는 무상태 모델로 축약하지 않는다.
Dashboard는 AT/RT를 발급·갱신하거나 인증 시도 기록을 쓰지 않는다. 웹 조회 권한은 owner/admin이며
타 조직 자원과 일반 구성원 접근 판정은 backend API 명세를 따른다.

## 5. 이미지·배포 소유

| 주체 | 소유하는 것 |
|---|---|
| backend | 같은 `:apps:dashboard-api`의 ARM64 이미지 build/push; 기존 task definition을 쓰는 ECS CD |
| infra CDK | ECS task/service·runtime 주입·IAM·DB 계정/권한 준비·ALB/TG/WAF·EKS AWS 자원 |
| infra Helm | Kubernetes Deployment/Service/설정·secret mount·TargetGroupBinding과 수동 Helm 배포 |
| frontend | API 소비, 화면 필드/상태, 동일 출처 BFF에서 조회 D·명령 E 선택 |

ECR `soma-376/dashboard-api`의 **동일 `linux/arm64` artifact**를 ECS와 EKS에서 사용한다.
backend CI의 immutable `sha-<commit>` tag와 승격용 `dev` tag가 가리키는 digest를 함께 기록하고,
ECS task가 실제 당긴 digest와 EKS의 tag/digest가 같은 artifact인지 확인한다.
EKS를 위해 다른 dashboard 이미지를 재빌드하지 않는다. backend CD는 CDK deploy, task definition 등록,
EKS Helm 배포를 수행하지 않는다. 앱 공급 → infra 서비스 생성 → ECS CD → 같은 digest의 수동 EKS 배포 순서와
실패·복구 절차는 infra 런북이 소유한다.
키 원본의 최초 등록·보존·IAM·버전 선택·runtime 파일 공급도 infra가 소유한다.
backend는 파일 로딩·서명/검증·기동 검증을 맡으며, 키 교체를 위해 이미지를 재빌드하지 않는다.

## 6. 현재 화면·검증 경계

현재 화면은 개요·팀 분석·구성원·운영/보안·설정이며, 개요는 화면 전용 단일 응답과 지표별 상태를 제공한다.
목록·상세·snapshot 페이지네이션·CSV는 같은 값/상태/사유를 보존한다. 설정 조회와 관리 명령의 소유는 §2대로다.
`contract-input-status`는 미입력과 만료 계약을 별도 목록으로 제공하고, 만료 계약 갱신은 설정의 해당 벤더 폼에서
E의 계약 저장 명령을 호출한다. 상세 응답과 화면 동작은 각 레포 API 문서가 기준이다.

이 문서 작업은 runtime·소유·배포 계약 채택이다. 배포 전에는 네 계정의 실제 권한·동일 DB/server 전제,
캐시 DDL 기동, 정확한 kid/issuer/audience, 인증 조회/snapshot 재사용, CORS의 GET/쓰기 분기와
미등록 method/path 404를 각각 검증한다. Running/Ready 또는 health 200만으로 이 검증을 완료했다고 기록하지 않는다.
NodePort 숫자는 이 허브에서 지정하지 않으며 PROJ-223에서 확정한 값을 CDK SG/TG/output과 Helm Service가 공유한다.

**JWT 공급의 현재 상태**: backend의 파일 기반 RS256 로더와 kid 검증은 존재한다.
ADR 0022의 최초 키 생성·Secret 등록·버전 고정 파일 공급·AWS 배포는 이 문서 채택으로 완료되지 않는다.
배포 수락에는 키 누락·빈 파일·읽기 불가·잘못된 PEM·RSA 크기 미달·키쌍/kid 불일치·부분 등록 복구·
전체 교체와 제거 후 롤백 검증, 활성/롤백 label 보존 확인이 포함된다. health 200·template 검증으로 대신하지 않는다.
