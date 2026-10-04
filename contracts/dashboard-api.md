# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | **frontend(`pulsemetry-frontend`)** ↔ **backend(`pulsemetry-backend`)** |
| 소재 | 조회는 `:apps:dashboard-api`, 관리 명령은 `:apps:enrollment-api` |
| 서버 측 상세 | `pulsemetry-backend/docs/dashboard-server-spec.md`(조회) · `pulsemetry-backend/docs/enrollment-server-spec.md` §12·§13(관리 명령) |
| 상태 | **변경 중** — 조회·관리 명령은 구현됐다. 소재 결정(backend ADR 0022)은 대시보드 snapshot 의 허브 ADR 초안(Proposed) 채택 전이다 |

이 문서는 레포 경계의 합의만 적는다. 엔드포인트·필드·오류는 서버 측 상세가 담는다 — 본문을 여기 복제하지 않는다.

## 1. 이미 정해진 제약

Dashboard API를 설계할 때 아래는 협상 대상이 아니다.

| # | 제약 | 근거 |
|---|---|---|
| 1 | **Signal Database와 User Database 두 곳을 조합해 응답한다.** 사용량 수치와 조직 구조를 조인해야 "팀별 비용"이 나온다 | overview.md 흐름 B |
| 2 | **조직 계약 단가 적용은 조회 시 계산한다(2차 가공).** 계약이 바뀌어도 재적재가 필요 없어야 한다 | overview.md §1, prd.md 제품 원칙 3 |
| 3 | **공시 기준 금액과 계약 적용 금액은 이름이 다른 두 값이다.** 응답 필드에서도 섞지 않는다 | prd.md 제품 원칙 3 |
| 4 | **파이프라인과 Signal Database에 쓰지 않는다.** 조회 앱은 원천을 읽고 자기 캐시(`dashboard_cache`)와 알림 평가 기록만 쓴다 | overview.md I-5, 대시보드 snapshot 허브 ADR 초안(Proposed), backend ADR 0022·0051 |
| 5 | **User Database 의 업무 쓰기(조직 구성·계약·정책·구성원·좌석·알림 규칙·알림 확인)는 Auth Service(enrollment-api)가 받는다.** 조회 앱은 읽기만 한다. 계정·토큰도 Auth Service의 것이다 | overview.md I-6, backend ADR 0026 |
| 6 | **시나리오 가용성은 저장하지 않고 조회 시 판정한다.** 카탈로그의 선행 조건을 User DB의 정책·조직 상태와 대조한다 | overview.md I-14 |
| 7 | **시나리오는 새 지표를 만들지 않는다.** 카탈로그가 참조하는 지표는 지표 레지스트리에 정의돼 Signal DB에 이미 존재하는 것뿐이다 | overview.md I-13 |
| 8 | 시나리오 적용 상태는 **`?scenario={id}`** 로 표현해 링크 공유 시 같은 화면이 복원되게 한다 | prd.md §6-1 |

## 2. 정해진 것

- **소재** — `pulsemetry-backend` 의 앱 둘이다. 조회는 `:apps:dashboard-api`(backend ADR 0022), 관리 명령은 `:apps:enrollment-api`(backend ADR 0026).
  비동기 명령(업데이트 안내·보존 정리·좌석 동기화·회수·복원)은 202 와 작업 ID 를 돌려주고 결과는 조회 앱의 작업 상태 조회로 본다(backend ADR 0039).
- **웹 대시보드 로그인·세션** — backend 가 AT·RT 를 직접 발급하고(enrollment-api), 조회 앱은 AT 와 현재 세션을 검증한다(backend ADR 0018·0026).
  Cognito는 쓰지 않는다([허브 ADR-0001](../adr/0001-otlp-authentication-model.md) — infra ADR-0008은 Superseded).
  비밀번호 자리는 `members.password_hash`다([`data-model.md`](data-model.md) §3). 회사 IdP 로그인(OIDC)은 아직 없다.
  사원이 CLI에서 하는 로그인과 데몬의 OTLP 인증(installation에 귀속된 `ptt_`)은 이 문서의 범위가 아니다.
- **인가** — 계정은 `members` 하나지만 **웹 대시보드에 들어오는 사람은 관리자(`role`이 `owner`·`admin`)뿐이다.**
  일반 사원(`role='member'`)은 403 이다. 팀장 권한 분리는 Non-goal이다.
- **응답 호환** — 응답 필드는 더하기만 한다. 프론트는 모르는 키를 무시하고, backend 는 프론트 요청서의 예시 JSON 사본과 선언한 가산 키로 응답의 키 구조를 대조한다
  (enrollment API 의 `DisallowUnknownFields` 문제 — [`enrollment-api.md`](enrollment-api.md) §6 M7 — 를 반복하지 않는다).

## 미해결

- 지표 레지스트리와 시나리오 카탈로그의 배포 형태(전역 참조 데이터를 어떻게 내려주는가) — 시나리오 런처는 아직 없다.
- 조회 앱의 자기 스키마 쓰기 예외(snapshot·알림 평가 기록) — 대시보드 snapshot 허브 ADR 초안의 리뷰와 번호 정리.
- 회사 IdP 로그인(OIDC) — 별도 작업(PROJ-186)이 이 저장소들에 아직 없다.

## 3. 화면

MVP 화면 구성은 `archive/IA.md`에 초안이 있으나 **미증류**다. frontend 착수 시 증류해
[`../product/`](../product/prd.md)로 옮긴다.

## 사용자 인증 계약 추가 (PROJ-107)

가입·로그인 JSON API와 CLI 코드 교환은 [사용자 인증 계약](user-auth.md)을 따른다. 기존 설치 봉투는 유지한다. 상태: 변경 중.

## revision 검증 연결 (PROJ-108)

사용자 인증 라이브러리에 verifyCurrentRevision을 제공한다. 실제 관리자 경로 연결은 PROJ-109다.
manifest_revision_mismatch는 409이며 RT 기반 재동기화로 복구한다. 인증 API 자체는 role에 따라 로그인 자격을 나누지 않는다.

## 등록 제품·계약 관리

설정의 목록은 등록 제품만 노출하며 조직당 제품 kind의 활성 등록은 하나다. 공급자 관측과 등록 UUID를 혼동하지 않는다.
기존 계약 변경은 시작일을 유지하는 입력 정정이며 이전 버전과 변경자를 보존한다.
제품 결정은 [ADR 0016](../adr/0016-registered-products-and-contract-corrections.md),
현행 HTTP 계약은 `pulsemetry-backend/docs/dashboard-server-spec.md`와 `enrollment-server-spec.md` §12를 따른다.
이 절은 위 문서 골격의 설정 관리 범위만 구체화한다.

## 등록 제품 기준 사용 알림

[ADR 0017](../adr/0017-registered-product-usage-alerts.md)에 따라 설정은 spend_spike·quota_exceeded·product_not_registered 규칙을 제공한다.
product_not_registered는 등록 제품이 있어야 켤 수 있고, 없으면 registered_products_not_configured다.
관측 product의 명시 매핑과 보관되지 않은 등록 제품 kind를 비교하며, 공급자·모델로 제품을 추정하지 않는다.
기존 model_not_allowed·tool_unapproved는 새 평가·편집 대상에서 제외하고 과거 알림의 식별자로만 보존한다.
설정 응답의 alertLists와 PUT /settings/alert-lists/{listId}를 제거한다. 규칙 PATCH와 알림 조회·확인 형식은 유지한다.
새 알림의 subject는 카탈로그 제품 ID이고 summary는 productId·productName·events·threshold다.
온보딩의 계약 벤더 등록이 비교 기준이며 모델·도구 승인 입력은 없다. 새 규칙은 관리자가 설정에서 직접 켠다.
