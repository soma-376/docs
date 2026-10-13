# 0017. 개요는 화면에 필요한 값과 지표 상태를 단일 API로 제공한다

## Status

Accepted — backend ADR 0045의 개요 `productUsage` 응답을 부분 대체한다. snapshot 제품 매핑과 다른 분석 API는 유지한다.

## Context

pulsemetry-backend와 pulsemetry-frontend의 개요 화면이 변경됐지만 좌석·알림·제품별 상세 등 사용하지 않는 필드와 조회가 남아 있다. 기간의 완전성만으로는 개별 지표의 식별·단가·토큰 누락을 설명하지 못한다.

## Decision

- 기존 개요 GET 경로와 단일 snapshot을 유지한다. 응답은 meta, comparison, summary, trend, modelMix, teamUsage, errors로 한정한다.
- 각 값은 value/status/reasons를 갖는다. 비교 지표는 비교 상태·불가 사유·이전 값·증감도 서버에서 계산한다. 불완전한 기간 또는 근거로 비교를 만들지 않는다.
- 값 없음(null), 확인된 0, 부분 관측, 조회 실패를 구분한다. 데이터가 없는 기간에는 네 영역 모두 null이고 errors는 빈 배열이다.
- 공통 snapshot·인증 실패는 HTTP 오류다. 독립 집계의 저장소 실패는 해당 영역 null과 안전한 오류 코드로 반환하며 성공한 영역은 유지한다.
- 부분 실패의 재시도 버튼은 페이지 공통으로 하나만 둔다. 단일 API를 다시 호출하며 기존 성공 영역은 로딩으로 덮지 않는다. 새로운 응답에서 실패한 영역에 이전 snapshot의 값을 끼워 넣지 않는다.
- CSV는 현재 개요의 지표·추이·화면에 묶인 모델·상위 팀과 미배정만 내보내고 상태와 사유를 보존한다.

## Alternatives Considered

영역별 API는 독립 재시도가 가능하지만 snapshot과 초기 로딩의 조율이 추가된다. 지금은 단일 API를 유지한다. 전체 실패만 반환하면 정상 영역까지 숨겨지므로 채택하지 않는다.

## Consequences/Tradeoffs

### Positive

불필요한 조회를 제거하고 화면·CSV에서 같은 지표 상태를 사용한다. 한 영역의 실패가 정상 영역을 가리지 않는다.

### Negative

부분 실패 재시도도 모든 영역을 다시 조회한다. 호환되지 않는 계약 변경이므로 프론트와 백엔드를 함께 배포해야 한다.
