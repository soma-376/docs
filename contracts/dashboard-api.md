# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | **frontend(레포 미생성)** ↔ **backend(소재 미정)** |
| 상태 | **골격.** 착수 전이며 확정된 것이 없다 |

> **이 문서는 자리표시자다.** 여기 적힌 것은 [`../architecture/overview.md`](../architecture/overview.md)에서
> 이미 결정된 구조적 제약이며, 실제 엔드포인트·필드는 아직 하나도 정해지지 않았다.
> 착수 시 이 문서를 [`enrollment-api.md`](enrollment-api.md) 수준으로 채운다.

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

## 현재 공시 단가 기준 사용 환산액

[ADR 0008](../adr/0008-current-public-reference-valuation.md)과 [ADR 0009](../adr/0009-current-reference-cache-benchmark-and-revaluation.md)에 따라 현재 기준 환산은 버전을 고정한 공식 Standard/global/USD 토큰 단가의 평가 금액이다. v3는 Claude 캐시 생성에 5분 공시 단가를 평가 기준으로 사용하며 실제 TTL이 5분이었다고 판정하지 않는다. 과거 실제 청구액이나 조직 계약액을 뜻하지 않는다. 사건 시각·보고 금액은 보존하며, 미검증 토큰 의미나 필요한 성분이 없으면 null이다. 가격표 조회 기준 시각·출처·평가 조건은 `pricing_version`의 불변 프로파일에서 확인한다.

검증된 Claude exclusive와 Codex inclusive 사용량은 같은 비중복 canonical 토큰 성분으로 정규화한 뒤 합산한다. 호환성이 확인되지 않은 의미 프로파일은 합산하지 않는다. 원천 캐시·reasoning 포함관계를 이중 합산하거나 미보고 성분을 0으로 채우지 않는다. 서로 다른 가격 판은 기존 혼재 판정을 유지한다. 승인된 과거 재평가는 기존 관측 ID와 원본을 보존해 최신 분석 판을 교체하며, API는 새 snapshot의 저장된 환산액을 읽는다.

[ADR 0010](../adr/0010-current-reference-luna-model-pricing.md)의 v4는 정확한 `gpt-5.6-luna`·`gpt-6-luna` 공시 단가를 추가한다. 기존 v1/v2/v3는 그대로 선택 가능하다. `codex-auto-review`는 공개 단가/동일 모델 근거가 확인되지 않아 null을 유지하며, 다른 모델 가격이나 0원으로 대체하지 않는다.
