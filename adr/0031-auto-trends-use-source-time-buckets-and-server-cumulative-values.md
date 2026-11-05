# 0031. 자동 추이는 발생 시각 버킷과 서버 누적값을 사용한다.

## Status

Accepted — 버킷·미래 표시·metadata는 [ADR 0033](0033-auto-trends-use-six-to-twelve-visible-period-buckets.md)이 부분 대체한다. 나머지 결정은 유지한다.

## Context

개요의 사용 환산액 추이와 팀 누적 추이는 조회 기간에 맞는 같은 시간 경계가 필요하다. 하루 조회를 일별 합계로만 표시하면 시간대별 변화를 볼 수 없고, 화면이 별도로 누적하면 기간 합계·상세·CSV의 기준이 달라질 수 있다.

이 결정은 backend의 집계·snapshot·API와 frontend의 차트·CSV에 걸친다. PROJ-315는 관측·미환산 정책을, [ADR 0030](0030-installation-report-gaps-and-transmission-loss.md)은 설치 보고 공백·전송 손실을 소유한다. 자동 집계가 이 정책들을 시간대별 완전성으로 추론해서는 안 된다.

## Decision

### 1. 날짜 범위로 고정된 자동 버킷을 선택한다

조회 범위는 `Asia/Seoul`의 `[startDate 00:00, endDate 다음 날 00:00)`이고 최대 366일이다. 후보 중 `ceil(범위 길이 / 간격) ≤ 60`인 가장 작은 간격을 고른다.

| 조회 일수 | 간격 |
|---|---|
| 1~2 | 1시간 |
| 3~7 | 3시간 |
| 8~15 | 6시간 |
| 16~30 | 12시간 |
| 31~60 | 1일 |
| 61~180 | 3일 |
| 181~366 | 7일 |

최소 1시간·최대 60개는 서비스 정책이다. 모든 버킷은 조회 시작 자정에 맞춘 반개구간이며 마지막 끝만 조회 종료에서 자른다. 3일·7일 간격을 달력 주차나 UTC 자정에 맞추지 않는다. 오늘도 하루 전체의 24개 버킷을 유지한다.

원본 이벤트의 `source_time`으로 집계한다. 일별 합계를 시간별로 나누거나 보간하지 않는다. `bucketEnd ≤ asOf`는 `elapsed`, `bucketStart ≤ asOf < bucketEnd`는 `in_progress`, `asOf < bucketStart`는 `upcoming`이다.

### 2. 자동 계약과 기존 일별 계약을 구분한다

개요·팀 목록·팀 상세 조회만 `bucket=auto|day`를 받는다. 생략과 `day`는 기존 일별 응답·집계를 유지하고 지원하지 않는 값은 400이다. 팀 사용자 API에는 `bucket`을 추가하지 않고 전달된 snapshot의 입력을 계승한다.

자동 응답은 `meta.trendAggregation`에 `mode=auto`, `intervalSeconds`, `valueMode`, `rangeStart`, `rangeEnd`, `asOf`를 제공한다. point는 `bucketStart`, `bucketEnd`, `temporalState`, 해당 버킷의 비누적 `usageRows`를 제공한다. 일별 DTO의 필드를 자동 누적값으로 재해석하지 않는다.

개요는 `valueMode=interval`이며 각 버킷의 금액·토큰을 기존 `Value<T>` 형식으로 제공한다. 팀은 `valueMode=cumulative`이며 `cumulativeEquivalentCostUsd`, `cumulativeKnownEquivalentCostUsd`, `cumulativeTotalTokens`, `cumulativeSessionCount`를 각각 `Value<T>`로 제공한다. 팀의 환산 완료·미환산 행 수는 해당 누적값과 같은 범위다.

금액은 정확한 Decimal 문자열이다. 화면 좌표 계산 이외에는 부동소수점으로 변환하지 않으며 CSV에서도 문자열을 보존한다. 사용 기록 없음·미환산·수집 상태·시간 진행 상태는 독립이다. 근거 없는 0을 만들지 않고, 명시적 0은 보존한다. 중간에 새 기록이 없어도 이미 확인한 누적값은 유지한다. 미래 지표와 usageRows·pricedUsageRows·unpricedUsageRows는 null이며 현재 구간의 확인된 값은 `in_progress`와 함께 제공한다.

세션은 `(팀, product, session namespace, session ID)`가 기간 안에 처음 등장한 버킷에서 한 번만 더한다. 세션 ID 결손은 세션 값의 상태에만 영향을 준다. 비용·토큰·세션의 결손은 각 누적 범위에서 판단하며 뒤의 정상 버킷으로 앞의 결손을 지우지 않는다.

### 3. snapshot의 입력과 기준 시각을 고정한다

자동 snapshot은 현재·비교 기간의 사용량과 관측 metadata를 같은 원본 선택에서 만들고 `source_time < snapshot.asOf`를 적용한다. 합계·추이·상세·사용자·CSV가 동일 snapshot 입력을 사용한다. `asOf`는 이벤트 시각 상한과 표시 기준이며 수신 시각 기준의 과거 데이터 상태 복원이 아니다.

기존 snapshot의 `query_contract`로 자동·일별 계약을 구분한다. 목록·상세 요청 모드와 snapshot 계약이 다르거나 만료되면 409로 재조회를 요구한다. 개요와 팀의 snapshot을 하나로 통합하지 않는다. 페이지·사용자·CSV 추가 조회는 해당 화면의 snapshot을 유지하고 만료 시 전체 내보내기를 다시 시작한다.

### 4. 수집 상태는 근거의 영향 범위대로 전달한다

PROJ-315의 공통 인계 계약은 같은 snapshot에 고정된 정책 버전·기준 시각·영향 범위·사유·근거 참조다. 자동 응답의 meta.collectionPolicy는 원천 정책 연결 전 policyVersion=proj-315-pending·status=pending과 policy_integration_pending 기간 annotation으로 미연동 상태를 명시한다. 이 adapter는 수집 완전성을 판정하지 않는다. 영향 시간과 대상을 특정할 수 있을 때만 버킷에 수집 상태 annotation을 연결하고, 특정할 수 없으면 기간 수준에 둔다. 단순 보고 공백으로 확인된 사용량·누적값을 숨기지 않는다.

기존 일별 완전성 판정을 시간 버킷에 복제하지 않는다. PROJ-316의 최근 168시간 손실은 서버가 확인한 페이로드 증가분이며 이벤트 발생 시각별 손실이 아니다. 이를 버킷·팀에 균등 배분하거나 토큰·이벤트 손실 수로 바꾸지 않는다. 영향 범위를 모르면 수집 상태 안내로만 연결한다.

### 5. 호환 배포와 표시 경계를 유지한다

`NEXT_PUBLIC_AUTO_TREND_ENABLED`의 기본값은 false다. backend를 먼저 배포하고 PROJ-315 공통 정책 통합 검증을 통과한 배포에서만 true로 빌드해 사용자 자동 요청을 활성화한다. 개발·테스트의 자동 경로 검증과 사용자 활성화는 구분한다. PROJ-316 알림 기능 전체 완료를 일괄 선행 조건으로 삼지 않는다.

팀 화면은 일별·자동 모두 비용·토큰 탭만 제공하며 세션 API·내부 집계·개요·표·CSV 지표는 유지한다. 팀 누적값을 화면에서 다시 누적하지 않는다. 개요 좌표는 버킷 시작, 팀 좌표는 버킷 끝이며 진행 중 누적값은 `asOf`까지 표시한다. 툴팁·키보드 탐색·CSV는 전체 KST 범위와 실제 간격·상태·사유를 제공한다.

복구는 PROJ-316 파서와 PROJ-318 탭 구성을 유지하는 호환 frontend를 false로 다시 빌드·배포해 일별 요청으로 전환한다. 호환되지 않는 과거 frontend로 되돌리지 않는다. 기존 일별·세션 API를 유지하므로 데이터 복구는 필요하지 않다.

## Alternatives Considered

### A. 일별 합계를 균등 배분하고 화면에서 누적한다
- 장점: 기존 일별 조회를 재사용한다.
- 단점: 실제 발생 시각과 세션 최초 관측을 잃고 합계·차트·CSV가 달라질 수 있다.
- 탈락 이유: 확인되지 않은 시간대별 사용량을 만들어 낸다.

### B. 기존 일별 DTO의 의미를 누적으로 바꾼다
- 장점: 응답 형태가 적다.
- 단점: 기존 클라이언트의 누적 연산과 충돌하고 snapshot 모드를 구별할 수 없다.
- 탈락 이유: 기존 일별 호환을 보장하지 못한다.

## Consequences/Tradeoffs

### Positive
- 날짜 경계·시간축·누적 의미가 API와 화면·CSV에서 일치한다.
- 관측된 사용값과 수집 상태를 별도로 설명하고 세션 지표를 보존한다.

### Negative
- 일별·자동 DTO와 snapshot 계약을 동시에 검증해야 한다.
- PROJ-315 인계 검증이 끝나기 전에는 사용자 자동 집계를 켤 수 없다.

## Follow-up

- PROJ-315에서 공통 annotation 원천과 값 공개 정책을 연결하고 사용자 활성화 gate의 통합 증거를 기록한다.

## Acceptance Criteria

- 모든 간격 경계·KST 자정·마지막 짧은 버킷·현재 및 미래 구간·`asOf` 정각 제외를 검증한다.
- 명시적 이벤트 fixture로 버킷 합계·누적 최종값·세션 중복 제거·Decimal·지표별 결손·일별 호환·400/409·CSV 재시작을 검증한다.
- 조회 횟수가 팀 수·버킷 수에 비례하지 않으며 PROJ-315 통합 gate를 우회하지 않는다.

## References

- [대시보드 API 계약](../contracts/dashboard-api.md)
- [확인된 사용량 부분합](0011-known-usage-subtotals.md)
- [설치 보고 공백과 전송 손실](0030-installation-report-gaps-and-transmission-loss.md)
