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
- 사람 계정 로그인·세션 — backend Spring Security가 AT·RT를 직접 발급한다(backend ADR-0007 Accepted, 구현은 아직 없다).
  Cognito는 쓰지 않는다([허브 ADR-0001](../adr/0001-otlp-authentication-model.md) — infra ADR-0008은 Superseded).
  `members.cognito_user_sub`는 폐기 예정 컬럼이다([`data-model.md`](data-model.md) §3).
- 인가 모델 — 현재 로그인 사용자는 사실상 관리자뿐이다. 팀장 권한 분리는 Non-goal이다.
- 응답 스키마의 버저닝 정책 — enrollment API가 `DisallowUnknownFields`로 겪는 문제
  ([`enrollment-api.md`](enrollment-api.md) §6 M7)를 반복하지 않는다.
- 지표 레지스트리와 시나리오 카탈로그의 배포 형태(전역 참조 데이터를 어떻게 내려주는가).

## 3. 화면

MVP 화면 구성은 `archive/IA.md`에 초안이 있으나 **미증류**다. frontend 착수 시 증류해
[`../product/`](../product/prd.md)로 옮긴다.
