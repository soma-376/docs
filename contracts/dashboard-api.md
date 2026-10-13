# 계약 — dashboard API

| 항목 | 내용 |
|---|---|
| 당사자 | **pulsemetry-frontend** ↔ **pulsemetry-backend** |
| 상태 | 현재 화면 조회·계약 알림의 공통 계약. 필드별 상세는 각 레포 API 문서가 소유 |

현재 업무 API의 경로는 `/api/v1`이다. 경로 전환 자체는 PROJ-200의 별도 변경이며 이 문서는 현재 화면의 조회 계약을 다룬다.

> 이 문서는 구조적 제약·화면별 조회 경계를 담는다. 필드별 상세는 backend의
> `docs/api/README.md` 및 `docs/api/reference/`, frontend의 `docs/api/`에서 관리한다.
> 코드와 계약의 반영은 운영 배포 완료를 뜻하지 않는다.

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

## 2. 조회 소유와 권한

`pulsemetry-backend/apps/dashboard-api`가 분석·설정·좌석·알림 조회를 맡는다. 조직·구성원·정책·계약의 관리 명령은 같은 레포의 `apps/enrollment-api`가 맡는다. dashboard-api는 원천을 읽고 자신의 집계 캐시만 쓴다. 프론트는 동일 출처 BFF에서 두 API를 구분해 호출한다(backend ADR 0026). 이는 현재 코드의 소유 경계이며 backend ADR 0022의 Proposed 상태와 별도 캐시·배포 전 채택 조건을 변경하지 않는다.

웹 대시보드는 `owner`·`admin`만 접근한다. 로그인·서비스 토큰의 상세 계약은 backend `docs/api/auth.md`와 frontend `docs/oidc-login.md`가 소유하며, 인증 결정 정리는 PROJ-188에서 별도로 진행한다. 이 변경은 인증 방식이나 권한을 바꾸지 않는다.

## 3. 현재 화면

개요·팀 분석·구성원·운영·보안·설정을 제공한다. 아래 조회 계약과 [제품 정의](../product/prd.md)의 현재 화면 범위를 따른다.

## 4. 개요 단일 API

[ADR 0017](../adr/0017-overview-screen-contract.md)에 따라 개요는 화면 전용 응답과 지표별 상태를 반환한다. 상세 계약은 backend `docs/api/reference/overview.md`와 frontend `docs/api/overview-api.md`를 따른다. 양쪽을 함께 배포한다.

## 5. 수집 사유·계약 상태

[ADR 0018](../adr/0018-dashboard-completeness-and-contract-input.md)에 따라 개요의 reasons에 날짜별 수집 미완료 근거를 추가한다. `contract-input-status` GET은 설정 권한으로 현재 계약 미입력 벤더 목록을 조회한다. [ADR 0020](../adr/0020-expired-contract-notifications.md)에 따라 같은 응답에 만료 계약 목록도 제공한다. 수집 상태·사용 환산액과 독립적이며 프론트와 백엔드를 함께 적용한다.

## 6. 나머지 페이지의 조회 투영

[ADR 0019](../adr/0019-dashboard-page-read-models.md)에 따라 팀 분석·구성원·운영·설정 응답은 현재 화면의 값과 상태만 제공한다. 목록·상세·관리 명령의 조회 경계와 snapshot 페이지네이션은 유지한다. 팀의 증감·사용자당 값은 서버가 판정하고, 구성원은 현재 명단과 기간 사용량을 분리한다. 운영 알림은 안전한 화면용 근거와 평가 상태만 반환하고, 설정의 벤더 상세는 상세 요청에서 받는다. 헤더 수집 상태를 각 페이지 응답에 중복하지 않는다. 프론트와 백엔드 계약은 함께 반영하며 CSV도 같은 값·상태·사유를 보존한다.

## 7. 만료 계약 알림과 갱신

`GET /api/v1/organizations/{id}/contract-input-status`는 기존 `requiredCount/vendors`와 별도로 `expiredCount/expiredContracts`를 반환한다. 각 만료 항목은 `vendorId/displayName/endDate`다. 최신 등록 계약의 종료일 다음 날(KST)부터 만료이며, 보관된 벤더와 미입력·시작 예정·종료일 없는 계약은 제외한다. 계약 원천 조회 한 번으로 두 목록을 만들며 전체 설정이나 사용량을 조회하지 않는다.

공통 알림함은 미확인 규칙 알림과 현재 만료 계약을 함께 표시한다. 만료 알림의 ‘계약 갱신’은 `/settings?vendor={vendorId}`로 이동해 해당 계약 폼을 연다. 만료 계약에는 별도 확인 처리가 없으며, 갱신 저장 성공 후 조직 조회를 갱신하면 목록과 벨 건수에서 사라진다. 규칙 알림의 확인 동작은 유지한다.

벨 건수는 미확인 규칙 알림 수와 만료 계약 수의 합이다. 한 조회라도 실패하면 합계를 0으로 표현하지 않고, 알림함에서 해당 조회를 재시도한다. 계약 미입력은 기존 헤더 안내를 유지하고, 만료 안내를 설정 상단이나 운영·보안 목록·CSV에 중복하지 않는다. 세부 응답은 backend `docs/api/reference/contract-input-status.md`와 frontend `docs/api/alerts-api.md`를 따른다.
