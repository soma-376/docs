# 대시보드 계약 fixture

[auto-trend.fixture.json](auto-trend.fixture.json)은 backend·frontend가 공유하는 PROJ-317의 가상 명시 이벤트와 기대값이다. 실제 응답 캡처나 수집 정책 통합 완료 증거가 아니다.

- `events`: 정밀한 발생 시각·금액·토큰·세션을 명시한다. 일별 합계를 분배하지 않는다.
- `expected`: 24개 개요 interval point와 팀 cumulative point, 부분 metadata, 기간 합계, 제외 이벤트를 제공한다.
- `metricCases`: 일부·전체 미환산, 세션 ID 결손의 지표별 영향, 기록 없음의 기대값이다. 각 사례는 events를 독립 입력으로 사용한다.
- `overviewMeta`와 `teamMeta`의 fixture snapshotId는 예시 식별자다. 개요·팀 실제 snapshot을 통합하라는 뜻이 아니다.

첫 사례의 정확한 금액 합은 `0.300000000003`, 토큰은 100, 세션은 4다. 01~02시에는 새 기록이 없어 개요는 no_data이지만 팀 누적은 유지한다. 03시의 명시적 0은 확인된 값이며 asOf인 03:30 정각과 그 뒤 이벤트는 제외한다. product 또는 namespace가 다른 같은 세션 ID는 각각 센다.

`proj-315-pending` annotation은 미연동 상태를 나타낸다. 실제 정책 버전·영향 범위·근거 참조를 확보한 통합 검증은 별도로 수행한다. backend의 `docs/api/examples/auto-trend.fixture.json`과 프론트 테스트가 복사본을 사용하면 이 파일과 동일한 값·상태·사유를 유지한다.
