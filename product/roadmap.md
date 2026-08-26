# 로드맵

> **골격 문서다.** 마일스톤 구조만 확정돼 있고 날짜·담당은 비어 있다.
> 원본 4주 계획은 `archive/PLAN.md` §5-3에 있다. Phase D에서 완성한다.

## M0 — 검증 (진행 중)

[`prd.md`](prd.md) §7의 가설 4개를 푸는 단계다. 제품 코드가 아니라 답이 산출물이다.

| 작업 | 산출물 | 가설 |
|---|---|---|
| 자체 팀 수집 활성화 → 이벤트 볼륨·원가 측정 (Privacy ON/OFF 양쪽) | 좌석당 원가(원화) | 4 |
| **Codex 시그널 매핑** | 두 도구의 필드 대응표 + 공통 스키마·provider별 확장 스키마 초안 | 2 |
| Wizard of Oz — 5개사 수작업 리포트 발송 | 시나리오 카탈로그 초기 목록 + 첫 고객 5곳 | 1 |
| 정보보안 담당 3인 인터뷰 | 허용 보존 기간 하한선, 추가 증명 요구사항 | 3 |
| 기본 마스킹 룰셋 초안 + 실제 프롬프트 샘플 정탐·오탐률 측정 | 룰셋 v0 | 3 |
| [`prd.md`](prd.md) §8의 **3·5·6번 결정** | 결정 → ADR | — |

## M1 — E2E 성립

온보딩부터 ClickHouse 적재까지 신원이 붙은 채로 끊기지 않고 흐르는 상태.
현재 남은 차단 요인은 [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §5에 있다.

## M2 — 대시보드 1차

Overview · Cost & Models · Members · Activity. 조직 축 집계와 계약 단가 실비용.
frontend 레포가 아직 없다 — 생성 시 [`../architecture/repos.md`](../architecture/repos.md)에 등록한다.

## M3 — 정책·시나리오

Privacy / 마스킹 규칙 2화면, Manifest 재배포 경로, 시나리오 런처.

## 마일스톤 밖

[`prd.md`](prd.md) §6-3 Non-goals 참조.
