# 0021. Dashboard runtime과 배포 경계를 backend와 infra가 나눠 소유한다

## Status

Accepted (backend ADR 0022·0023의 허브 채택 조건을 충족한다. overview I-5·I-6의 조직 설정 쓰기 주체와
원천/캐시 구분을 대체한다. 사용자 JWT 키 원본·버전·파일 공급은 [ADR 0022](0022-user-jwt-key-provisioning.md)가
보충하며 나머지 결정은 유효하다. 실제 AWS 배포·DB 권한·live E2E 완료를 뜻하지 않는다.)

## Context

`pulsemetry-backend`에는 enrollment-api, telemetry-ingest와 별도 앱 `:apps:dashboard-api`가 구현돼 있다.
`pulsemetry-frontend`는 분석·설정·좌석·알림을 조회하고 같은 URI의 쓰기는 enrollment-api에 보낸다.
허브의 기존 dashboard 계약은 소재 미정인 골격이고 overview I-5·I-6은 조직 설정 쓰기를 Dashboard API에 배정한다.
그대로 배포하면 조회 앱에 원천 쓰기 계정을 주거나 GET을 enrollment에서 떼어낼 수 있다.

Dashboard는 원천 RDS/ClickHouse를 읽고 별도 연결로 snapshot·알림 캐시를 쓴다.
backend ADR 0022·0023은 이 캐시 예외와 배포 전 허브 채택을 요구한다.
같은 앱이 ECS/EKS에서 동작하려면 `infra`와 설정·계정·artifact·라우팅 책임을 함께 확정해야 한다.
이 ADR의 당사자는 `pulsemetry-backend`, `pulsemetry-frontend`, `infra`다.

## Decision

1. Dashboard API는 `pulsemetry-backend/apps/dashboard-api`에 둔다. 기본 포트는 **8081**이며
   동일 `linux/arm64` image를 ECS와 EKS가 사용한다. ECR `soma-376/dashboard-api`의
   immutable `sha-<commit>` 및 승격 `dev` tag의 digest 대응과 실제 runtime digest를 기록한다.
2. 원천을 읽는 dashboard와 조직 관리 명령을 쓰는 enrollment를 분리한다.
   전체 path·method·OPTIONS 소유는 [dashboard 계약 §2](../contracts/dashboard-api.md#2-공개-경로와-method-소유)가 기준이다.
   `auth/me`, onboarding·invitations GET과 기존 관리 쓰기는 enrollment에 남긴다.
3. Dashboard는 RDS/ClickHouse **source/cache 네 연결·역할별 계정**을 받는다.
   원천 쓰기를 금지하고 자신의 `dashboard_cache` DDL·읽기·쓰기만 허용한다.
   캐시 계정에도 snapshot 복사와 공개 CAS에 필요한 제한된 원천 SELECT를 준다.
4. infra는 DB/schema·계정·권한·runtime 값을 준비하고 backend는 캐시 DDL을 소유한다.
   enrollment-api가 `enrollment`·`telemetry_ops` Flyway를 먼저 실행하고 dashboard가
   RDS cache Flyway 뒤 ClickHouse cache 멱등 DDL을 기동 시 적용한다. 실패하면 기동도 실패한다.
5. 현재 SQL 구조 때문에 RDS source/cache는 같은 logical database, ClickHouse source/cache는 같은 server다.
   RDS cache에는 `telemetry_ops.tenant_retention_boundary` SELECT 및 현재 cache V1의 최초 적용이 요구하는 database CREATE,
   ClickHouse cache에는 복사 원천 SELECT가 필요하다. enrollment 알림 확인은 `dashboard_cache.alerts` SELECT를 받는다.
6. Dashboard는 enrollment가 발급한 RS256 AT를 정확한 kid 공개키 맵·issuer·audience로 검증하고,
   현재 user session·member·tenant 상태를 source RDS에서 읽는다. JWT private key, token hash HMAC key,
   enrollment 관리자 토큰과 raw S3 권한은 dashboard에 전달하지 않는다.
   `PULSEMETRY_USER_AUTH_PUBLIC_KEY_FILE`은 kid=local의 값이며 임의 kid는 명시적 맵으로 배포한다.
   키 원본은 [ADR 0022](0022-user-jwt-key-provisioning.md)의 환경별 개인키·공개키 고정 Secret 두 개다.
   dashboard는 공개키 Secret만 받아 파일로 검증하고, dev ECS/EKS는 같은 공개키 원본·버전을 사용한다.
7. backend는 앱 이미지 공급과 기존 task definition에 대한 ECS CD를 맡는다.
   infra CDK는 ECS service/task·AWS/EKS 자원·ALB/TG/WAF·runtime 주입을, infra Helm은 Kubernetes workload를 맡는다.
   EKS는 같은 artifact digest로 수동 배포한다. backend CD가 CDK/Helm이나 task definition 등록을 수행하지 않는다.
8. 두 ALB는 명시 path+method로 D/E를 나누고 shared URI의 preflight는 OPTIONS+
   `Access-Control-Request-Method`로 구분한다. public default는 404, health는 내부 target group 확인용이다.
   WAF 앱 내용 검사에는 dashboard 경로를 포함하고 기존 등록 rate 범위와 OTLP 예외는 유지한다.

## Constraints

- 설정 이름·기본값·기동 검증과 운영자 판단 조건은 [dashboard 계약 §4](../contracts/dashboard-api.md#4-runtime-설정-인터페이스)를 따른다.
  배포 수치를 local 프로필의 편의값으로 대신하지 않는다.
- source readOnly 옵션은 권한을 대신하지 않는다. 원천 SELECT 계정을 cache/master 계정으로 대체하지 않는다.
- 별도 캐시는 수집 원천과 다르다. Signal 원천의 Enricher 단일 쓰기, raw 조회/재처리 제한과 비용 이원화 결정은 유효하다.
- NodePort 숫자는 PROJ-223 구현에서 확정한다. SG/TG/output/Helm Service 값이 일치하기 전에는 배포하지 않는다.
- JWT 키 생성은 명시적 초기화에서만 수행한다. 키 파일 설정·배포 버전·보존·교체/롤백은 ADR 0022를 따르며,
  공개키를 앱 이미지에 포함하거나 일반 재배포에서 새 키를 생성하지 않는다.

## Alternatives Considered

### A. 별도 dashboard 레포·별도 ECS/EKS 이미지

- 장점: 릴리스 권한과 코드 경계를 더 강하게 분리할 수 있다.
- 단점: 공통 persistence/security 및 같은 코드의 공급 경로가 중복된다.
- 탈락 이유: 현재 독립 앱/배포 경계로 충분하며 동일 artifact 검증을 유지한다.

### B. 원천 DB 계정 하나로 조회·캐시·관리 명령 수행

- 장점: 연결과 권한 공급이 단순하다.
- 단점: dashboard 조회에서 원천을 변경할 수 있고 캐시 DDL과 enrollment 마이그레이션 책임이 섞인다.
- 탈락 이유: 역할별 최소 권한과 원천 읽기/관리 쓰기 분리를 지킬 수 없다.

### C. JWT 서명만 확인하는 dashboard

- 장점: 인증 시 DB 의존을 줄인다.
- 단점: 현재 세션 폐기·계정 비활성화·role 변경을 즉시 반영하는 구현을 잃는다.
- 탈락 이유: 기존 `UserAccessVerifier`와 서비스 인증 계약을 유지한다.

## Consequences/Tradeoffs

### Positive

- frontend·backend·infra가 같은 path/method 표와 설정 인터페이스를 사용한다.
- dashboard는 원천 변경 권한 없이 snapshot·알림 캐시를 독립적으로 관리한다.
- ECS/EKS를 같은 image digest로 비교하고 배포 소유자를 분명히 할 수 있다.

### Negative

- 네 연결 계정과 캐시 DDL/원천 SELECT 예외, enrollment의 cache 알림 SELECT를 함께 관리해야 한다.
- RDS 같은 logical DB·ClickHouse 같은 server 배치 제약과 cache V1 최초 적용의 database CREATE 권한을 감수한다.
- dashboard 기동은 캐시 DDL에, 인증은 현재 RDS 세션 조회에 의존한다.
- 공유 URI의 method/preflight 분기는 listener 규칙 수와 회귀 검증 부담을 늘린다.

## Follow-up

- PROJ-220: infra ADR/런북에 ECS·EKS 배포 순서, 주입·TG/WAF와 설치/기능 증거를 반영한다.
- PROJ-221: ECS dashboard service/task를 추가한다. PROJ-222: runtime·계정·권한 전달을 구현한다.
- PROJ-223: Helm workload를 추가하고 dashboard NodePort를 확정한다. PROJ-224: ALB/TG/WAF 라우팅을 반영한다.
- PROJ-225: 네 계정 권한과 인증·snapshot·CORS·복구 live 증거를 별도 배포 검증으로 확보한다.

## Acceptance Criteria

- backend ADR 0022·0023과 운영 명세가 이 결정 및 현재 config/controller 소유를 가리킨다.
- 계약의 설정표는 dashboard `application.yaml`의 식별자와 일치하며 모든 공개 endpoint의 method·OPTIONS 소유가 명시된다.
- overview I-5·I-6, repository map, 계약/ADR 인덱스가 같은 경계를 설명한다.
- 배포 기록은 설치 readiness, 인증/조회/snapshot 기능, DB 권한과 digest 검증을 구분한다.
- 키 원본의 채택과 실제 파일 공급·권한·인증 교체 증거를 구분하고 ADR 0022의 실패/롤백 조건을 확인한다.

## References

- [dashboard API 계약](../contracts/dashboard-api.md)
- [ADR 0022 - 사용자 JWT 키 원본·파일 공급](0022-user-jwt-key-provisioning.md)
- [시스템 아키텍처](../architecture/overview.md), [레포 경계](../architecture/repos.md)
- backend `DashboardApiProperties`, `SecurityConfig.dashboardAuthenticator()`, `SourceStoreConfig`, `CacheSchemaInitializer`
- backend `RdsCacheSchema.migrate()`, `SnapshotManifestStore.publish()`, `SnapshotCopySql.intake()`, `UserAccessVerifier.verify()`
- backend ADR 0022(별도 dashboard 앱·읽기 전용 원천), ADR 0023(snapshot·캐시 DDL), ADR 0026(관리 명령 경계)
- infra ADR 0007(ECR 선공급), 0024(CD 역할), 0029(dashboard 배포 결정)
