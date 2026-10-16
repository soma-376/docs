# 0007. 분석 snapshot과 수집 생애주기 요약을 원본 저장소와 분리한다

## Status

Proposed — 문서 검토용 초안이다. 채택 전에는 현행 I-4·I-5·I-6과 공유 도메인 계약 D-2가 유효하다. 이 ADR은 Dashboard API의 소재, backend ADR 0018, 대표 팀 정책, 가격표 관리 주체를 확정하지 않는다.

## Context

관련 레포는 `pulsemetry-backend`(수집·정규화·적재), `pulsemetry-frontend`(조회 계약), `infra`(DB 권한·자원·정리 작업), `rdb-schema`(RDS 설계도), 이 문서 허브다. Dashboard API의 구현 레포는 별도 결정이다.

프론트 API 제안의 snapshot은 목록 순서뿐 아니라 사용 데이터와 가격·계약·팀 매핑 버전을 고정한다. 팀 목록에서 받은 snapshotId로 팀 상세와 사용자 목록을 처음 요청할 수도 있다. endpoint별 첫 응답만 보관하면 하위 요청에서 같은 데이터 상태를 재현할 수 없다.

교체 저장된 분석 원본에 `received_time <= cutoff`를 나중에 적용해도 과거 버전이 복원되지는 않는다. cutoff 뒤에 정정된 관측이 통째로 빠질 수 있다. 여러 endpoint가 공유할 분석 입력을 한 번 선택해 보관할 필요가 있다.

한편 분석 테이블과 수신 ledger는 보존 정책으로 삭제된다. 남아 있는 행의 `min/max`만으로 조직의 최초 수신이나 수집 이력 존재 여부를 판단하면, 보존 정리 후 기존 조직이 `never_observed`로 되돌아간다. 분석 기록의 보존과 수집 생애주기 요약의 수명은 다르다.

현행 [아키텍처 §3·§4](../architecture/overview.md)는 Signal Database 쓰기를 Enricher로 한정하고 Dashboard API 쓰기를 금지한다. RDS 공유 도메인 계약의 [D-2](../contracts/data-model.md)는 파이프라인의 `enrollment` 쓰기를 금지한다. 임시 분석 복제와 수집 운영 상태를 도입하려면 저장소별 소유권을 명시적으로 구분해야 한다.

## Decision

이 절은 채택 시 적용할 제안이다. 현재의 쓰기 권한을 부여하거나 기존 규칙을 자동 대체하지 않는다.

### 1. 원본과 임시 조회 데이터를 분리한다

ClickHouse의 별도 논리 DB `dashboard_cache`에 일반 MergeTree 테이블 `snapshot_usage`를 둔다. 분석 원본의 선택 결과를 복사하는 일시적 조회 저장소이며 정규화 진실원이 아니다. Dashboard API의 snapshot 작성자만 이 테이블에 INSERT한다. `telemetry_events`·`telemetry_metric_points` 등 분석 원본은 Dashboard API에 계속 읽기 전용이다.

하나의 snapshot은 공유·영속 metadata인 **snapshot manifest**와 그 manifest가 가리키는 `snapshot_usage` build로 구성한다. manifest는 RDS의 **`dashboard_cache.snapshots`**에 둔다. 작은 상태·버전·권한·만료 metadata이며 사용량 원본을 RDS에 복제하지 않는다. SQL·마이그레이션 세부는 후속 구현 명세로 정한다. API 프로세스 재시작이나 다른 API 인스턴스의 요청에도 같은 상태를 읽어야 하며 프로세스 메모리만을 manifest의 진실원으로 삼지 않는다.

RDS `dashboard_cache`의 서비스 수준 writer는 Dashboard snapshot 작성자다. 자기 캐시 스키마에 한정된 I-6 예외를 제안하며 `enrollment`의 기존 계정·조직 설정 쓰기 경계는 바꾸지 않는다. ClickHouse와 RDS의 `dashboard_cache`는 이름이 같은 별개 논리 DB·스키마다.

manifest는 최소한 다음을 담는다.

| 범주 | 고정할 값 |
|---|---|
| 식별·소유 | 불투명 `snapshot_id`, `build_id`, tenant, 요청자 또는 재사용이 허용된 권한 범위 식별자 |
| 분석 범위 | 현재·비교 기간, timezone, compare, 적용한 데이터 범위와 조회 계약 버전 |
| 참조 데이터 | 가격 프로파일·모델 별칭, 계약, 팀 귀속·표시 및 필요한 로스터의 불변 버전 또는 복제본 식별자, tenant 삭제 정책 epoch |
| 수명·공개 | `building` / `ready` / `failed`, 생성 시각, `ready_at`, `expires_at`, 취소·무효화 정보 |
| 검증 | 복사 완료 행 수, 품질 판정에 필요한 카운터·가격 버전 집합, 고정한 데이터 시점의 설명 |

행이 0개인 정상 snapshot도 `ready` manifest를 갖는다. `snapshot_usage`의 행 존재 여부로 snapshot의 유효성이나 만료를 판정하지 않는다.

### 2. 전체 분석 입력을 한 번 고정한다

두 분석 테이블에서 각각 `FINAL`로 최신 active 행을 선택하고 유효한 `source_time`을 가진 관측을 **하나의 고정 결과 스트림**으로 수집한다. 여기서 매핑 완료·primary 이벤트만 사용량 payload로, metrics·안전한 generic 관측까지 포함한 날짜는 관측 일자 metadata로 함께 산출한다. 또는 그 전체 base 결과를 먼저 불변 staging에 고정한 뒤 둘을 파생한다. 테이블 출처는 구분하고 metrics·generic 값을 사용량·비용에 합산하지 않는다. payload를 복사한 후 관측 일자 metadata를 얻으려고 원본을 두 번째 조회하지 않는다. 그래야 두 값 사이에 들어온 재처리·삭제로 사용량과 관측 상태가 갈라지지 않는다.

첫 원본 선택 범위는 현재 기간과 비교 기간의 **전체 권한 범위**다. top3·첫 페이지·현재 선택 팀으로 줄이지 않는다. 기간이 겹치면 같은 관측을 중복 복사하지 않는다. `compare=none`은 비교 범위를 만들지 않는다.

복사본은 이후 변경하지 않는다. 같은 snapshot을 사용하는 조직 합계, 모델 구성, 팀 목록, 팀 상세, 사용자 목록은 이 복사본에서 계산한다. 사용자·세션 distinct를 다시 계산할 식별자, 이벤트 당시 대표 팀, 일자·모델 축, Decimal 비용, null 성분과 품질 판정 근거를 보존한다. 일별·팀별 distinct 수를 더해 조직 사용자·세션 수를 만들지 않는다.

가격과 토큰이 미확정인 행을 버리거나 0으로 채우지 않는다. 생성 당시 가격 버전 집합·누락 여부를 보존하고, 같은 snapshot에서 최신 가격표를 다시 적용하지 않는다. 모델 별칭이나 팀 이름도 매 요청의 최신 사전으로 바꾸지 않는다. 참조 데이터가 버전 조회를 지원하지 않으면 필요한 값을 함께 복제한다.

현재·비교 기간의 참조 데이터 선택과 사용량 복사가 모두 끝난 뒤에만 snapshot을 공개한다. 여러 저장소를 하나의 전역 트랜잭션으로 읽었다고 주장하지 않으며, 고정된 원본 선택 결과와 참조 버전으로 일관성의 범위를 정의한다.

`snapshot_usage`만으로 모든 제품 endpoint를 구현하지는 않는다. 무사용 구성원 로스터, 선택 기간 밖의 `lastUsedAt`, 계약·좌석·설정에는 별도 고정 입력이 필요하다. 현재 운영 현황인 `ingest.asOf`와 명령 실행 시 자격 검증은 snapshot 밖의 최신 상태다.

`dataThrough`의 의미와 개요가 공통 snapshot을 선택하는 요청 방식은 프론트 계약에 함께 명시한다. 수신 최댓값을 고정된 사용 데이터 cutoff로 부르지 않는다. snapshotId 없이 임의 시각의 과거 데이터 상태를 재현한다고 약속하지 않는다.

### 3. 완료 공개와 만료를 분리한다

생성 상태는 `building → ready` 또는 `building → failed`다. RDS에 building과 시작 시 삭제 정책 epoch를 기록하고 ClickHouse 복사·참조 고정·검증을 끝낸 뒤 RDS manifest를 조건부 갱신해 `ready`로 공개한다. 공개 CAS는 **같은 build가 아직 building이고 무효화되지 않았으며 tenant 삭제 정책 epoch가 시작 시점과 같음**을 함께 검사한다. 삭제 경계·정책이 바뀌었으면 그 build는 폐기하며 새 경계로 다시 생성한다. 복사 완료 직전에 무효화된 build를 ready로 되살리지 않는다.

실제 조회를 처리할 ClickHouse replica에서 build의 완결성과 가시성을 확인해야 한다. 다른 replica로 읽기를 넘기려면 그 replica에서도 같은 조건을 확인하거나 검증된 replica로 라우팅한다. RDS ready 하나만으로 모든 replica에 복사가 끝났다고 가정하지 않는다.

실패하거나 결과가 불확실한 build의 일부 행은 조회에 사용하지 않는다. 재시도는 **새 `build_id`**로 수행하고, 이전 build에 추가 INSERT해 중복을 만드는 경로를 허용하지 않는다. 공개된 build는 다른 build로 갈아끼우지 않는다. 유효한 ready snapshot을 저장소 장애로 읽지 못하는 상황은 정상 빈 데이터나 TTL 만료와 구분한다.

`expires_at = ready_at + 10분`을 기본으로 한다. 같은 build의 모든 행에 manifest의 동일한 API 만료 경계를 적용한다. 매 API 요청은 manifest의 상태와 만료를 검사하고, 만료·무효화되었거나 존재하지 않는 snapshot이면 `409 snapshot_expired`를 반환한다. `building`이나 `failed`를 정상 빈 결과로 응답하지 않는다. 최초 생성 중 상태·실패의 HTTP 표현은 대시보드 오류 계약에서 정한다.

ClickHouse TTL은 물리 정리 수단이다. TTL merge가 아직 실행되지 않아 행이 남아 있어도 만료된 snapshot을 반환하지 않는다. payload의 물리 보존 시각은 생성 제한 시간·10분 API 수명·진행 중 조회 유예를 포함해, ready 전에 복사한 행이 API 만료보다 먼저 지워지지 않게 정한다. API 유효기간을 물리 TTL에 맞춰 연장하지 않는다. manifest 공개 전 orphan build는 별도 생성 제한 시간과 정리 규칙을 갖는다.

### 4. 권한과 요청 조건을 매번 검증한다

snapshotId는 권한 증명이 아니다. 매 요청에서 인증, tenant 접근권한, snapshot의 권한 범위, 팀·개인 목록 조회 권한을 다시 확인한다. 생성 뒤 권한이 바뀌면 기존 snapshot을 무효화하고 새 범위로 다시 조회한다. 기존 권한을 보관했다는 이유로 회수된 접근을 10분간 유지하지 않는다.

공유 분석 snapshot의 범위와 endpoint별 응답 캐시·cursor의 조건은 구분한다. cursor는 snapshotId, endpoint, 팀, 검색 필터, 정렬에 묶는다. 다른 endpoint에서 같은 분석 snapshot을 처음 읽는 것은 허용하되 그 endpoint의 현재 권한을 검사한다. 필터·정렬 변경 시 cursor를 재사용하지 않는다. 공통 snapshot 재사용 범위와 새 snapshot을 만드는 조건은 프론트 계약으로 확정한다.

조직별 snapshot 동시 생성 수·보관 수, 복사 행 수·바이트 수, 쿼리 메모리·시간, 전체 임시 저장량에 상한을 둔다. 상한 초과 시 데이터를 잘라 정상 총액으로 반환하지 않는다. 수치와 초과 오류는 대표 규모의 측정으로 정한다. 임시 조회 부하가 수집·적재를 방해하지 않도록 계정·쿼리 자원 제한을 분리한다.

### 5. 조직 수집 요약은 별도 RDS 운영 스키마에 둔다

RDS에 별도 **`telemetry_ops` 스키마**를 두고 **`telemetry_ops.tenant_ingest_summary`**에 tenant별 수집 생애주기 요약을 보관한다. 원본 사용량·계약·역할을 저장하는 스키마가 아니다. `enrollment`와 같은 RDS 인스턴스를 사용할 수 있지만 스키마·권한을 분리하며, 이 ADR은 별도 물리 DB 생성을 요구하지 않는다. DDL 진실원은 backend의 버전 관리 마이그레이션이며, ingest 프로세스 기동이 임의 DDL을 실행하는 경로를 만들지 않는다. 구체적인 모듈·테이블 SQL은 후속 구현 결정이다.

요약의 서비스 수준 writer는 **telemetry ingest**다. Dashboard API는 읽기 전용이다. ingest의 `enrollment` 접근은 기존 D-2대로 읽기 전용을 유지하며, `installations.last_seen_at` 등을 이 요약의 대용으로 쓰지 않는다. I-6의 도메인 쓰기 경계와 새 운영 상태의 쓰기 경계를 채택 시 명시적으로 구분한다.

| 값 | 의미 |
|---|---|
| `first_received_at` | 이 tenant의 최초 유효한 수집 receipt 수신 시각. 발생 시각이 아니다 |
| `first_observed_at` | 유효한 원본 시각 중 최솟값. 늦게 도착한 데이터로 더 과거가 될 수 있고, 시각을 얻지 못했으면 null |
| `last_received_at` | 새 live 수집 receipt의 마지막 수신 시각. 아카이브 재처리의 실행 시각이 아니다 |

receipt의 처리 멱등 키와 최초 `received_time`을 보존한다. 같은 receipt의 저장 재시도는 min/max에 같은 입력을 적용하고 재시도 시각을 새 수신 시각으로 쓰지 않는다. HTTP 재전송과 별개의 새 수신을 구분하는 identity는 ingest 계약에서 정하며, 본문 hash만으로 다른 정상 수신을 합치지 않는다. 아카이브 재처리는 원래 receipt·시각을 유지하고 수집 신선도를 갱신하지 않는다.

receipt·요약의 저장 실패를 성공으로 숨기지 않는다. 기존 ADR 0006의 영구·일시 실패 계약을 유지하면서 재시도로 중복 효과가 생기지 않는 순서·복구 경로를 정의한다. 요약을 조회할 수 없을 때는 상태를 unknown으로 내고, 조직이 수집한 적 없다는 뜻으로 바꾸지 않는다. 요약 부재를 `never_observed`로 판정하려면 요약 도입 이전 데이터의 초기화가 완료됐다는 근거도 필요하다.

### 6. 수명과 삭제 책임을 구분한다

수신 ledger는 공통 운영 보존 정책으로 정리한다. 조직의 `aggregateRetentionMonths`를 ledger에 그대로 적용하지 않는다. 운영 보존 기간의 수치는 운영 정책에서 정하며, tenant 생애주기 요약은 ledger·분석 기록의 정리와 독립적으로 남긴다. 보존 범위 밖 날짜의 coverage는 복원할 근거가 없다고 답하고, 요약이 있다는 이유로 과거 날짜를 complete로 만들지 않는다.

| 데이터 | 쓰기 책임 | 삭제 책임 |
|---|---|---|
| 분석 원본 | 기존 수집 파이프라인의 적재 단계 | 정책을 검증하는 보존 정리 worker. Dashboard API는 작업 요청·상태 조회만 수행 |
| `dashboard_cache.snapshot_usage` | Dashboard snapshot 작성자 | TTL 및 snapshot 정리 worker. 만료·무효·실패 build만 대상 |
| RDS `dashboard_cache.snapshots`·고정 참조 데이터 | Dashboard snapshot 작성자 | snapshot 정리 worker. 유효한 요청을 처리할 상태와 만료 응답 규칙을 보존 |
| 수신 ledger | 수집 단계 | 공통 운영 보존 worker |
| RDS `telemetry_ops.tenant_ingest_summary` | telemetry ingest | tenant 삭제 절차의 전용 worker. 일반 분석·ledger 보존 정리 대상이 아님 |
| RDS `telemetry_ops`의 tenant별 삭제 경계 | 전용 보존 정리 worker. ingest·조회 서비스는 읽기 전용 | tenant 삭제 절차에서 전체 정리 완료 조건에 따라 처리. 일반 보존 연장으로 제거하지 않음 |

보존 정리는 tenant와 시각 경계로 같은 관측의 모든 버전을 삭제한다. tenant를 공유하는 월 파티션을 한 조직의 삭제에 쓰지 않는다. 운영 정리 worker는 삭제 대상에 한정된 권한을 가지며, Dashboard API에 원본 삭제 권한을 부여하지 않는다.

tenant 전체 삭제는 분석 데이터, ledger, 요약, snapshot·고정 참조, 아카이브의 각 소유자에게 전파되는 별도 절차다. 정책·권한·삭제로 기존 snapshot을 더 이상 보여 줄 수 없으면 즉시 무효화하고 물리 정리는 worker가 수행한다.

분석 기록을 지운 뒤 아카이브 재처리나 늦은 도착이 같은 기록을 되살리지 않도록 RDS `telemetry_ops`에 tenant별 **`deleted_before`** 경계를 영속 관리한다. 전용 보존 정리 worker만 경계를 갱신하며 일반 적재·재처리는 sink 직전에, 조회는 원본 읽기와 snapshot 검증 때 현재 경계를 검사한다. 경계가 바뀌면 기존 snapshot을 무효화한다.

보존 worker는 삭제 정책 epoch와 경계를 먼저 반영하고, 이전 epoch로 검사한 진행 중 쓰기를 정리한 뒤 모든 관측 버전에 DELETE를 적용한다. **이전 epoch writer의 결과 미확정 INSERT까지 모두 종료됐거나 더는 적용되지 못하도록 차단됐다는 근거가 생기기 전에는 cleanup 완료를 선언하지 않는다.** 요청 timeout이나 취소 신호 전송만으로 INSERT가 종료됐다고 가정하지 않는다. 뒤늦게 끝난 쓰기가 물리 행을 다시 남기지 않게 fence·조정·검증한다.

operation의 논리 삭제 완료와 저장장치에서 물리 제거되는 SLA는 구분한다. 경계 검사·쓰기 fence·물리 정리 완료 조건은 정리 구현의 필수 명세이며, DELETE 요청을 접수했다는 것만으로 전체 삭제 성공을 선언하지 않는다.

보존 기간을 연장해도 이미 완료된 삭제 경계를 자동으로 되돌리지 않는다. 복원은 보존된 아카이브와 권한·정책을 확인한 별도 명시적 절차로만 허용한다. tenant 전체 삭제는 일반 복원 대상으로 취급하지 않는다. 삭제 경계의 테이블 SQL과 동시성 제어 방식은 후속 명세로 정한다.

## Alternatives Considered

### A. API 프로세스 메모리에 공통 snapshot을 보관한다

별도 ClickHouse 쓰기 권한 없이 구현할 수 있다. 그러나 여러 API 인스턴스 사이 공유, 재시작·eviction, 큰 기간의 행 보관을 처리해야 한다. 같은 불변 snapshot 계약은 여전히 필요하다. 이번 제안은 프로세스 수명과 분리된 공유 저장을 선택한다.

### B. endpoint별 첫 응답만 캐시한다

각 endpoint의 페이지 이동은 고정할 수 있지만, 팀 목록에서 아직 호출하지 않은 사용자 endpoint의 입력 상태는 고정하지 못한다. 공통 snapshot을 읽은 뒤 응답을 캐시하는 최적화로만 사용한다.

### C. 원본의 `received_time` 필터로 과거 상태를 재현한다

추가 저장이 없지만 교체·merge 후 사라진 버전을 되살릴 수 없다. 분석 원본의 과거 버전 보관 계약을 별도로 도입하지 않는 한 채택하지 않는다.

### D. 분석 테이블·ledger의 min/max를 수집 생애주기 요약으로 사용한다

운영 저장소를 추가하지 않지만 보존 삭제와 재처리가 수집 이력·신선도를 바꾼다. tenant 요약과 세부 ledger의 수명을 분리한다.

## Consequences/Tradeoffs

### Positive

- 같은 snapshot의 목록·상세·사용자 집계가 동일한 분석 입력과 참조 버전을 사용한다.
- 만료와 물리 삭제를 분리해 TTL 실행 시점에 API 동작이 좌우되지 않는다.
- 분석 보존 정리 뒤에도 조직의 수집 이력과 최초·마지막 수신 의미를 유지한다.
- 원본 분석과 `enrollment`의 쓰기 경계를 보존하면서 새 쓰기 권한을 한정한다.

### Negative

- 동일 원본을 snapshot별로 복사하는 쓰기·저장 비용이 생기고 수집과 자원을 경쟁한다.
- manifest 공개, 실패 build 정리, 참조 버전 고정, 여러 저장소의 복구 순서가 추가된다.
- tenant 요약과 삭제 경계는 별도 마이그레이션·권한·운영 책임이 필요하다.
- snapshot 밖의 현재 운영 상태와 snapshot 안의 분석 상태가 다를 수 있어 API가 기준 시점을 구분해야 한다.

## Follow-up

- 채택 여부를 backend·frontend·infra·RDS 설계 담당이 검토한다. 채택 시 아키텍처 I-4·I-5·I-6과 저장소 표, dashboard 계약을 같은 변경에서 개정한다. D-2의 `enrollment` 읽기 전용은 유지한다.
- Dashboard API 소재, backend ADR 0018, 대표 팀 정책, 공급자·가격표 관리 주체는 별도 결정으로 남긴다.
- RDS manifest의 SQL·원자적 공개와 실제 읽기 replica 검증, snapshot 행 스키마·인덱스·TTL 유예, 마이그레이션 실행 주체, receipt 멱등 identity와 요약 복구를 구현 명세로 구체화한다.
- 프론트 API의 snapshot 공유 범위, 개요 요청, `dataThrough`, 생성 중·실패·상한 초과 오류를 합의한다.
- `telemetry_ops` 삭제 경계의 테이블 SQL, 적재와 삭제의 경쟁 제어, operation의 논리 완료·물리 제거 SLA, tenant 삭제와 아카이브 복원 절차를 정한다.
- 대표 1·28·366일 조회와 동시 생성 부하를 측정해 복사량·메모리·시간·보관량 상한과 ledger 공통 보존 기간을 정한다.

## Acceptance Criteria

- snapshot 생성 뒤 원본 정정·가격표·팀 이름·계약 변경이 있어도 같은 snapshot의 조직·팀·사용자 합계와 순서가 달라지지 않는다.
- 비교 기간에만 사용한 팀, 겹치는 비교 기간, 여러 팀·날짜에 걸친 세션, 가격·토큰 null이 복사 뒤에도 같은 의미를 갖는다.
- 행 0개의 ready snapshot은 정상 빈 결과이고, 부분 INSERT 실패·응답 유실 뒤 재시도한 build는 중복·부분 결과를 공개하지 않는다.
- 유효기간 10분 뒤 물리 행이 남아 있어도 409이며, API 재시작·다른 인스턴스 조회가 만료 전 유효 snapshot을 잃지 않는다.
- 다른 tenant·개인 목록 권한 없는 사용자·권한 회수된 사용자는 기존 snapshotId로 데이터를 읽지 못한다.
- 오래된 원본의 지연 도착은 `first_observed_at`만 과거로 확장할 수 있고, 저장 재시도·아카이브 재처리가 새 수신으로 집계되지 않는다.
- 분석·ledger 보존 정리 후에도 tenant 요약이 유지된다. tenant 삭제 시에는 요약·snapshot을 포함해 지정된 범위가 정리된다.
- 삭제 완료 경계보다 오래된 아카이브 재처리·지연 도착은 일반 경로로 분석 기록을 되살리지 않는다.
- payload 생성과 관측 일자 metadata 산출은 같은 고정 원본 결과를 사용한다. primary가 아닌 관측만 있는 날짜도 동일 snapshot의 metadata에 남는다.
- 이전 정책 epoch의 INSERT 결과가 불확실한 동안 cleanup은 완료되지 않으며, 생성 중 정책 변경·무효화와 경합한 build는 ready 공개 CAS에 실패한다.
- Dashboard 계정은 분석 원본에 쓰지 못하고, ingest 계정은 `enrollment`에 쓰지 못한다. 각 worker는 소유한 정리 대상만 삭제한다.

## References

- [아키텍처 §3·§4](../architecture/overview.md), [dashboard API 계약](../contracts/dashboard-api.md), [공유 도메인 계약](../contracts/data-model.md)
- [ADR 0006 — OTLP ingest의 재시도·상태 계약](0006-otlp-ingest-retry-and-status-contract.md)
- 형제 레포 `pulsemetry-frontend/docs/api/README.md` 「공통 조회 계약」, `teams-api.md` 「집계 기준」, `members-api.md` 「숫자의 의미」
- 형제 레포 `pulsemetry-backend/docs/telemetry-normalization-final.md` §2.5·§2.6·§4·§10
- [ClickHouse TTL 공식 문서](https://clickhouse.com/docs/concepts/features/operations/delete/ttl) — TTL 정리는 merge 시 실행되므로 API 만료와 구분한다
