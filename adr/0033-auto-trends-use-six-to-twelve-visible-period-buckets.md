# 0033. 자동 추이는 전체 기간을 6~12개로 계획하고 현재까지만 표시한다.

## Status

Accepted — ADR 0031의 버킷 선택·미래 표시·집계 metadata를 부분 대체한다. 0032는 후속 PROJ-318 브랜치의 결정 번호다.

## Context

backend PROJ-317과 frontend PROJ-317은 최대 60개 고정 초 구간을 사용한다. 사용량·비용 분석에서 더 적은 구간과 달력월 집계가 필요하며 미래 공간은 현재 관측의 가독성을 낮춘다. 두 저장소의 SQL·snapshot·응답 검증·차트·CSV가 함께 변경된다.

## Decision

전체 선택 기간의 계획은 최소 6개·최대 12개다. 간격은 1일=3시간, 2일=6시간, 3~6일=12시간, 7~12일=1일, 13~24일=2일, 25~36일=3일, 37~84일=7일, 85~168일=14일, 169~366일=달력 1개월이다. KST 조회 시작을 기준으로 경계를 만든다. 월 경계는 원래 시작일에 N개월을 더하고 해당 월의 마지막 날로 보정한다. 13번째 짧은 구간은 12번째에 합친다.

간격은 전체 선택 기간으로 정하고 표시 종료는 effectiveEnd=max(rangeStart,min(rangeEnd,asOf))다. 시작이 effectiveEnd 이상인 버킷은 반환하지 않는다. 마지막 끝은 effectiveEnd로 자른다. 원래 계획 종료보다 asOf가 이르면 in_progress, 아니면 elapsed다. 정각에는 길이가 0인 다음 구간을 만들지 않는다. 미래 전체 기간은 빈 배열이며 이미 지난 빈 구간의 값은 no_data로 보존한다. 최소 6개는 전체 계획에만 적용되며 현재까지 응답은 0~12개다.

meta.trendAggregation은 policyVersion=calendar-buckets-v2, intervalSeconds(고정 간격은 초·월은 null), intervalMonths(월은 1·고정 간격은 null), effectiveEnd를 제공한다. rangeStart/rangeEnd는 전체 선택 기간을 유지한다. 차트 축의 끝은 effectiveEnd이며 구간의 정확한 끝을 툴팁과 CSV에 표시한다. 월 구간을 30일로 환산하지 않는다. 자동 snapshot 계약은 dashboard-auto-trend-v2이며 이전 자동 계약과 혼용하지 않는다. 일별 모드와 PROJ-315 활성화 gate는 유지한다.

원본 발생 시각·Decimal 금액·지표별 결손·세션 최초 등장 중복 제거·서버 누적값은 보존한다. 구간별 반복 SQL 없이 경계 목록으로 일괄 집계한다. 프론트는 서버의 구간 계약을 검증하고 재집계하지 않는다.

## Alternatives Considered

- 실제 화면에도 최소 6개를 강제: 미래 구간을 추가하거나 시간이 흐를 때마다 간격을 바꾸므로 채택하지 않는다.
- 월을 30일로 고정: 달력월과 윤년 경계가 어긋나므로 채택하지 않는다.

## Consequences/Tradeoffs

### Positive

- 전체 기간의 구간 수가 제한되고 현재 관측에 축 너비를 사용한다.
- 같은 구간 경계를 SQL·API·CSV가 공유한다.

### Negative

- 진행 중인 기간의 표시 구간은 6개 미만일 수 있다.
- 월 마지막 구간은 13번째 잔여 기간 병합으로 한 달보다 길 수 있다.
- 자동 v1 클라이언트와 호환되지 않아 양쪽 배포 및 snapshot 만료 처리가 필요하다.

## Acceptance Criteria

- 1~366일·월말·윤년에서 계획 6~12개와 현재 이전 연속 구간을 검증한다.
- 구간 경계 직전·정각·직후, asOf 제외, 팀 누적·총합·세션 보존을 실제 SQL로 검증한다.
- 프론트 파서·시간축·CSV와 일별 호환을 검증한다.

## References

- [기존 자동 추이](0031-auto-trends-use-source-time-buckets-and-server-cumulative-values.md)
- [Dashboard API](../contracts/dashboard-api.md)
