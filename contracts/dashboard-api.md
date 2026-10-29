# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | **frontend(레포 미생성)** ↔ **backend(소재 미정)** |
| 상태 | **부분 확정.** §3 사용 환산액은 PROJ-290의 [backend #53](https://github.com/soma-376/pulsemetry-backend/pull/53)·[frontend #31](https://github.com/soma-376/pulsemetry-frontend/pull/31)에 반영됐다. 나머지 API 명세는 골격이다 |

> **§3을 제외한 API 명세는 자리표시자다.** 여기에는 [`../architecture/overview.md`](../architecture/overview.md)에서
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
