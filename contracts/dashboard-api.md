# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | [pulsemetry-frontend](https://github.com/soma-376/pulsemetry-frontend) ↔ [pulsemetry-backend](https://github.com/soma-376/pulsemetry-backend) |
| 상태 | **부분 확정.** 사용 환산액 절은 PROJ-290의 [backend #53](https://github.com/soma-376/pulsemetry-backend/pull/53)·[frontend #31](https://github.com/soma-376/pulsemetry-frontend/pull/31)에 반영됐다. 설치 보고 공백·전송 손실 절은 ADR 0030에 따라 변경 중이며 나머지 API 명세는 골격이다 |

> **사용 환산액·설치 보고 공백·전송 손실 절을 제외한 API 명세는 자리표시자다.** 여기에는 [`../architecture/overview.md`](../architecture/overview.md)에서
> 이미 결정된 구조적 제약을 정리했다. 엔드포인트별 계약은
> [`enrollment-api.md`](enrollment-api.md) 수준으로 보완한다.

## 1. 이미 정해진 제약

Dashboard API를 설계할 때 아래는 협상 대상이 아니다.

| # | 제약 | 근거 |
|---|---|---|
| 1 | **Signal Database와 User Database 두 곳을 조합해 응답한다.** 사용량 수치와 조직 구조를 조인해야 "팀별 비용"이 나온다 | overview.md 흐름 B |
| 2 | **조직 계약 단가 적용은 조회 시 계산한다(2차 가공).** 계약이 바뀌어도 재적재가 필요 없어야 한다 | overview.md §1, prd.md 제품 원칙 3 |
| 3 | **공시 기준 금액과 계약 적용 금액은 이름이 다른 두 값이다.** 응답 필드에서도 섞지 않는다 | prd.md 제품 원칙 3 |
| 4 | **파이프라인과 Signal Database에 쓰지 않는다.** 읽고, 조직 설정만 쓴다 | overview.md I-5 |
| 5 | **User Database 쓰기는 조직 설정 범위로 한정한다**(조직 구성·계약·정책·구성원). 계정·토큰은 Auth Service의 것이다 | overview.md I-6 |
| 6 | **시나리오 가용성은 저장하지 않고 조회 시 판정한다.** 카탈로그의 선행 조건을 User DB의 정책·조직 상태와 대조한다 | overview.md I-14 |
| 7 | **시나리오는 새 지표를 만들지 않는다.** 카탈로그가 참조하는 지표는 지표 레지스트리에 정의돼 Signal DB에 이미 존재하는 것뿐이다 | overview.md I-13 |
| 8 | 시나리오 적용 상태는 **`?scenario={id}`** 로 표현해 링크 공유 시 같은 화면이 복원되게 한다 | prd.md §6-1 |

## 2. 착수 전 정해야 할 것

- **Dashboard API 서버가 어디에 사는가** — `pulsemetry-backend`에 모듈로 추가할지 별도 레포로 뗄지.
  이것은 크로스레포 결정이므로 [`../adr/`](../adr/README.md)에 ADR로 남긴다.
- **웹 대시보드 로그인·세션** — backend Spring Security가 AT·RT를 직접 발급한다(backend ADR-0007 Accepted, 구현은 아직 없다).
  Cognito는 쓰지 않는다([허브 ADR-0001](../adr/0001-otlp-authentication-model.md) — infra ADR-0008은 Superseded).
  `members.cognito_user_sub`는 제거됐고 비밀번호 자리는 `members.password_hash`다([`data-model.md`](data-model.md) §3).
  사원이 CLI에서 하는 로그인과 데몬의 OTLP 인증(installation에 귀속된 `ptt_`)은 이 문서의 범위가 아니다.
- 인가 모델 — 계정은 `members` 하나지만 **웹 대시보드에 들어오는 사람은 관리자(`role`이 `owner`·`admin`)뿐이다.**
  일반 사원(`role='member'`)은 같은 계정으로 CLI·파이프라인 쪽만 쓴다. 이 구분은 role 기반 인가로 막는다.
  팀장 권한 분리는 Non-goal이다.
- 응답 스키마의 버저닝 정책 — enrollment API가 `DisallowUnknownFields`로 겪는 문제
  ([`enrollment-api.md`](enrollment-api.md) §6 M7)를 반복하지 않는다.
- 지표 레지스트리와 시나리오 카탈로그의 배포 형태(전역 참조 데이터를 어떻게 내려주는가).

## 3. 화면

MVP 화면 구성은 `archive/IA.md`에 초안이 있으나 **미증류**다. frontend 착수 시 증류해
[`../product/`](../product/prd.md)로 옮긴다.

## 사용 환산액

[ADR 0028](../adr/0028-single-reference-price-catalog.md)에 따라 서버는 단일 참조 단가표로 새 사용량을 계산한다.
사용 환산액은 각 관측에 저장된 Standard/global/USD 토큰 평가 금액의 합이다. 실제 청구액·조직 계약액·벤더 보고 금액이 아니다.

- 단가표는 공급자 공식 문서로 확인한 모델 단가를 우선하고, 나머지는 Codeburn 0.9.24 고정 카탈로그를 쓴다.
  별칭(`codex-auto-review → gpt-5.5`)은 환산 기준의 선택이며 실제 모델 동일성을 뜻하지 않는다.
  출처·평가 조건·별칭은 단일 카탈로그의 문서와 코드에서 관리한다([ADR 0008](../adr/0008-current-public-reference-valuation.md)).
- 가격판을 선택하지 않으며 관측·snapshot에 가격판 식별자를 새로 기록하지 않는다. API는 `pricingVersion`을 반환하지 않고,
  `pricing_version_mixed`를 확인 불가 사유로 사용하지 않는다. 프론트는 가격판 일치 여부를 검사하거나 표시하지 않는다.
- Claude 캐시 생성은 5분 쓰기 단가로 평가한다. 실제 TTL 판정이 아니다.
- 검증된 Claude Code(exclusive)와 Codex(inclusive) 사용량은 같은 비중복 토큰 성분으로 합산한다. 다른 의미 프로파일은
  합산하지 않는다. 캐시·reasoning을 이중 합산하거나 미보고 성분을 0으로 채우지 않는다.
- 단가표에 없는 모델, 미검증 토큰 의미, 필요한 성분의 누락은 null이다. 해당 범위에 미가격 행이 있으면 전체 환산액은 null이다.
  계산 가능한 행의 저장된 금액은 별도 부분합으로 제공한다([ADR 0011](../adr/0011-known-usage-subtotals.md)).
- 과거 가격표의 차이는 전체값·부분합·평균·비중·비교를 막지 않는다. 비교 기간이나 다른 그룹에 있던 가격표 차이도
  선택한 범위의 금액을 가리지 않는다. 기간 완전성·관측 공백·미가격 규칙은 유지한다.
- 기존 금액과 원천 사용량은 보존한다. 단가표 변경으로 과거 금액을 자동 재계산하지 않으므로 합계는 과거 사용량을 모두
  현재 단가로 다시 평가한 값이 아니다. 저장된 사용량을 명시적으로 재평가한 경우 새 snapshot에서 읽는다.


## 설치 보고 공백·전송 손실

[ADR 0030](../adr/0030-installation-report-gaps-and-transmission-loss.md)을 따른다. 당사자는 backend·frontend이며
읽기 권한과 호환 배포는 infra가 함께 관리한다. 기존 관리자 권한과 조직 격리를 유지한다.

`GET /api/v1/organizations/{organizationId}/ingest-status`는 기존 상태와 독립적으로 `transmissionLoss`를 반환한다.

| 필드 | 의미 |
|---|---|
| `payloadCount` | 최근 168시간 확인된 양수 손실 증가분 합계. 집계 불가는 null |
| `windowStart`, `windowEnd` | 서버 기준 반개구간 `[windowEnd − 168시간, windowEnd)` |
| `availableFrom` | 실제 집계 가능한 시작 시각. 집계 기반이 없으면 null |
| `availability` | `complete`, `partial`, `unavailable` |
| `reasons` | 자료 부족·중단·카운터 감소·관측 근거 없음·조회 실패 등 집계 제한 |
| `affectedInstallationCount` | 창 안에 양수 증가분이 기록된 설치 수. 집계 불가는 null |

`complete`는 보고된 증가분 집계의 근거를 의미한다. `partial`은 확인된 수량을 유지하며,
`unavailable`은 0 대신 null을 반환한다. 단순 보고 공백은 가용성 저하 사유가 아니다.
현재 상태용 ClickHouse 조회 실패와 손실 원장용 RDS 조회 결과는 독립이다.

`GET /api/v1/organizations/{organizationId}/ingest-status/loss-installations`는 헤더의 `windowEnd`를 받아
동일한 창을 cursor로 조회한다. 설치 ID·기기명·구성원·손실 수·마지막 증가분 확인 시각·현재 설치 상태를 반환한다.
집계에 포함된 폐기 설치도 남긴다. 분석 화면의 날짜·팀·벤더 필터는 두 API에 적용하지 않는다.

알림은 `category=collection`, `ruleId=installation_report_gap`, `eventCount=null`을 추가한다.
전용 evidence의 `kind`는 `installation_report_gap`이며 `installationId`, nullable `deviceName`, `basisAt`,
`thresholdReachedAt`, nullable `lastReceivedAt`, nullable `resolutionDetectedAt`, nullable `resolutionReason`을 담는다.
`resolutionReason`은 `report_resumed` 또는 `installation_revoked`다. 알림의 `open`·`closed`와 확인 여부는 독립이다.
범주별 미확인 수는 `collection`을 포함한다. `security` 필터에는 포함하지 않는다.

헤더는 “최근 7일 보고된 전송 손실 N건”을 사용하고 상세에 다음 설명을 제공한다.

> 데몬이 보고한 손실 페이로드 수입니다. 이벤트·토큰 수와 다르며, 전송 대기·정책상 비수집·단순 보고 공백은 포함하지 않습니다.

단순 보고 공백은 “보고 상태 미확인”으로 표현한다. 구버전 응답에 새 필드가 없으면 “집계 정보 없음”으로 표시하며
0으로 대체하지 않는다. 새 범주·evidence를 수용하는 프론트부터 배포하고, 손실 원장 이관·읽기 권한 검증 후 dashboard를 배포한다.


집계 사유 코드는 `tracking_not_started`, `history_incomplete`, `tracking_interrupted`, `counter_decreased`,
`no_observation`, `query_failed`다. 상세는 필수 `windowEnd`, 선택 `limit`(기본 20, 1~100), `cursor`, `snapshotId`를 받는다.
응답은 `{meta:{organizationId,snapshotId,asOf},windowStart,windowEnd,installations:{items,totalCount,nextCursor}}`이며
items는 `{installationId,deviceName,memberId,account,payloadCount,lastObservedAt,status}`다.
`deviceName`·`memberId`·`account`는 nullable, status는 `active` 또는 `revoked`다. 후속 페이지는 같은 창과 snapshotId를 유지한다.


헤더 조회 때 조직·`windowEnd`별 기기 집계와 요약을 자기 cache에 10분간 고정한다. 합계와 영향 설치 수는
동일한 기기 집계에서 계산하며 상세·후속 페이지는 같은 결과를 읽는다. 원천의 늦은 커밋은 다음 헤더 갱신에 반영한다.
헤더가 발급한 창의 결과가 없거나 만료됐으면 상세는 409 `snapshot_expired`이며 새 헤더를 조회한다.
미래 `windowEnd`는 400이다. `meta.asOf`는 해당 `windowEnd`다. 만료 cache 정리와 원천 이력 보존은 별개다.

## 자동 시간 추이

변경 중 — backend·frontend가 [ADR 0031](../adr/0031-auto-trends-use-source-time-buckets-and-server-cumulative-values.md)에 따라 합의한 계약이다. [공통 fixture](examples/auto-trend.fixture.json)는 명시적인 이벤트와 기대 point를 담으며 실제 API 캡처가 아니다.

개요 `/analytics/overview`, 팀 목록 `/analytics/teams`, 팀 상세 `/analytics/teams/{teamId}`만 `bucket=auto|day`를 받는다. 경로 앞에는 기존 `/api/v1/organizations/{organizationId}`가 붙는다. 생략 또는 day는 기존 일별 DTO·집계를 유지하고 지원하지 않는 값은 400이다. 팀 사용자 조회는 bucket 없이 전달된 snapshot의 입력을 계승한다.

자동 범위는 KST `[startDate 00:00,endDate 다음 날 00:00)`이며 최대 366일이다. 조회 시작에 고정된 반개구간 버킷을 만들고 마지막만 종료에서 자른다. 간격은 1시간·3시간·6시간·12시간·1일·3일·7일 중 `ceil(범위 길이/간격) ≤ 60`인 가장 작은 값이다. 오늘도 24개 구간이며 달력 주차에 맞추지 않는다. 원본 source_time으로 집계하고 일별 값 분배·보간은 하지 않는다.

```ts
type Value<T> = { value: T | null; status: "complete" | "partial" | "unavailable"; reasons: string[] };
type TrendAggregation = {
  mode: "auto";
  intervalSeconds: number;
  valueMode: "interval" | "cumulative";
  rangeStart: string;
  rangeEnd: string;
  asOf: string;
};
type AutoPoint = {
  bucketStart: string;
  bucketEnd: string;
  temporalState: "elapsed" | "in_progress" | "upcoming";
  usageRows: number | null;
  pricedUsageRows: number | null;
  unpricedUsageRows: number | null;
};
type AutoOverviewTrend = {
  bucket: "auto";
  points: Array<AutoPoint & {
    equivalentCostUsd: Value<string>;
    knownEquivalentCostUsd: Value<string>;
    totalTokens: Value<number>;
  }>;
};
type AutoTeamTrendPoint = AutoPoint & {
  cumulativeEquivalentCostUsd: Value<string>;
  cumulativeKnownEquivalentCostUsd: Value<string>;
  cumulativeTotalTokens: Value<number>;
  cumulativeSessionCount: Value<number>;
};
```

자동 응답의 meta.trendAggregation은 필수다. 개요는 valueMode=interval이고 기존 trend를 AutoOverviewTrend로 반환한다. 팀은 valueMode=cumulative이고 team.trend 배열을 AutoTeamTrendPoint로 반환한다. 자동 point에는 일별 date·observation을 혼합하지 않는다. 팀 pricedUsageRows·unpricedUsageRows는 누적 범위이고 usageRows는 양쪽 모두 해당 버킷의 비누적 행 수다.

`bucketEnd ≤ asOf`는 elapsed, `bucketStart ≤ asOf < bucketEnd`는 in_progress, 나머지는 upcoming이다. 미래 지표는 null/unavailable/upcoming이고 세 행 수도 null이다. 빈 자동 기간도 전체 시간축을 제공한다. 근거 없는 elapsed/in_progress 지표는 null/unavailable/no_data, 행 수는 0이다. 새 기록 없는 중간 구간도 확인된 팀 누적값은 유지한다.

명시적 0과 기록 없음은 다르다. 금액은 정확한 Decimal 문자열이며 CSV에서도 원문을 보존한다. 미환산 전체값은 null/pricing_missing이고 계산 가능한 부분합은 별도 값이다. 세션 ID 결손은 session_identity_missing으로 해당 지표만 제한하며 정상 비용·토큰의 상태에 전파하지 않는다. 세션은 `(팀,product,namespace,ID)`가 기간 안에 처음 관측된 버킷에 한 번만 더한다.

자동 snapshot은 현재·비교 기간의 사용량과 관측 metadata를 같은 원본 선택에서 `source_time < asOf`로 제한한다. 합계·추이·상세·사용자·CSV가 같은 입력을 읽는다. asOf는 발생 시각 상한이며 수신 시각 기준의 과거 상태 복원이 아니다. 자동 개요 meta.snapshotId는 필수이고 day 개요에는 추가하지 않는다. 팀의 기존 snapshotId는 유지한다.

query_contract로 day/auto를 구분한다. 목록·상세의 요청 모드가 snapshot과 다르거나 만료되면 409 snapshot_expired다. 모드를 캐시 키에 포함하고 페이지·상세는 같은 모드·snapshot을 유지한다. CSV 사용자 조회 중 만료되면 전체 내보내기를 새 snapshot으로 다시 시작한다. 개요·팀 snapshot을 통합하지 않는다.

### 수집 annotation 인계와 활성화

자동 meta.collectionPolicy는 PROJ-315 원천 정책 연결 전 아래 pending 계약을 제공한다. asOf·범위는 해당 snapshot과 같다. 이 annotation은 수집 완전성이나 실제 손실의 증거가 아니다.

```ts
type PendingCollectionPolicy = {
  policyVersion: "proj-315-pending";
  asOf: string;
  status: "pending";
  annotations: [{
    scope: "period";
    rangeStart: string;
    rangeEnd: string;
    teamId: null;
    reasons: ["policy_integration_pending"];
    evidenceRefs: [];
  }];
};
```

실제 인계는 동일 snapshot에 정책 버전·기준 시각·영향 범위·사유·근거 참조를 고정한다. 정확한 시간·팀이 확인될 때만 버킷 annotation으로 연결하고 모르면 기간 수준에 남긴다. 기존 일별 완전성을 모든 시간 버킷에 복제하지 않으며 단순 보고 공백만으로 확인된 사용값을 숨기지 않는다. PROJ-316의 최근 168시간 페이로드 손실은 버킷·팀·이벤트·토큰에 배분하지 않는다.

팀의 비용·토큰 탭만 표시하며 기존 세션 API·개요·표·CSV는 유지한다. 팀 누적을 화면에서 다시 더하지 않는다. 개요 좌표는 버킷 시작, 팀은 버킷 끝이며 진행 중 값은 asOf까지다. 실제 간격·전체 KST 시간 범위를 툴팁·키보드 탐색·접근성 설명에 제공한다. CSV는 mode·간격·시간대·asOf·버킷 경계·시간 상태·지표 상태·사유를 보존하고 null은 빈 셀이다. chart_trend는 선택한 비용·토큰 축이다.

backend를 먼저 배포하고 PROJ-315 공통 정책 통합과 실제 API 개요·팀·CSV 검증을 통과한 배포에서만 `NEXT_PUBLIC_AUTO_TREND_ENABLED=true`로 frontend를 빌드한다. 기본값은 false다. 개발·mock 검증은 사용자 활성화 증거가 아니다. 복구는 PROJ-316 파서·PROJ-318 탭 구성을 유지한 호환 frontend를 false로 다시 빌드·배포해 day 요청으로 전환한다.

### 현재 상태·미해결

PROJ-315 실제 정책 원천은 pending adapter와 분리된 통합 검증 대상이다. [공통 fixture](examples/auto-trend.fixture.json)의 빈 evidenceRefs는 그 통합 완료를 의미하지 않으며, 사용자 자동 활성화 gate는 이 증거가 확보되기 전까지 닫혀 있다. PROJ-316 알림 기능 전체를 일괄 선행 조건으로 삼지는 않는다.

## 팀 탭 선택과 세션 계약 보존

[ADR 0032](../adr/0032-limit-team-tabs-and-design-subscription-usage-ratio-separately.md)에 따라 팀 화면은 day/auto 모두 비용·토큰만 선택한다. 세션 선택에만 연결된 추이·비교·벤더 비중·툴팁·범례·안내를 제거하며 공유 데이터·파서와 세션 API·집계는 유지한다. 과거 axis=session·tab=session URL은 비용을 기본으로 보여주고 날짜·비교·team 선택은 보존한다.

개요 세션 카드와 배치, 팀 비용 표의 세션당 값, 상세·사용자·구성원 지표, 기존 CSV의 세션 값·상태·사유는 변경하지 않는다. chart_trend는 선택한 비용·토큰 축이다. sessionCount·sessionsPerUser·tokensPerSession·equivalentCostPerSessionUsd·cumulativeSessionCount·sessionMixAvailable·sessionMixReason과 API sort=session은 기존 의미를 유지한다.

미배정 사용의 관측 여부는 정상 응답의 pricedUsageRows와 unpricedUsageRows가 모두 있을 때 합계로 판정한다. 합계가 양수면 세션 ID·금액이 null이어도 행을 보인다. 합계가 0일 때만 관측 사용 없는 미배정 행을 숨긴다. 기존 구버전 호환의 optional 카운터를 필수로 바꾸지 않으며 한쪽이라도 없으면 unknown으로 행을 보존하고 실제 사용이 있었다고 단정하지 않는다. 조회 실패를 0으로 바꾸지 않는다. 구성원은 기존 periodUsage != null을 사용한다.

세션 ID 결손은 정상 비용·토큰 값의 상태를 제한하지 않는다. 원천·snapshot의 session_id·session_id_namespace, firstSessionDays·firstSessionBuckets, 마지막 사용·활동·좌석 판단과 세션 파생값을 보존한다. 이번 탭 정리에는 backend API 변경이나 migration이 없다.

구독료 대비 사용 환산액 카드는 [독립 후속 설계](subscription-usage-value.md)다. 계약 효력·정정 이력, 같은 제품·기간의 분자·분모, 정확 분수 일할 계산·상태·API 안·텍스트 시안을 해당 문서에서 정의한다. 이번 탭 정리로 개요 카드를 교체하거나 placeholder를 추가하지 않는다. PROJ-317만 복구할 때도 비용·토큰 탭과 PROJ-316 파서를 보존한 frontend를 자동 집계 false로 빌드한다.
