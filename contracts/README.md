# 레포 간 계약 (contracts)

**두 레포가 합의해야 성립하는 인터페이스**의 단일 출처다. 계약은 당사자가 둘이므로 한쪽 레포에 두지 않는다.

## 왜 여기 있는가

E2E 검증에서 확인된 차단 결함 4건 중 3건이 "구간 내부의 버그"가 아니라
**저장소 경계에서 합의가 코드로 상륙하지 않은 지점**이었다. 대표 사례:

> 백엔드는 telemetry 토큰을 무염 SHA-256으로 저장하고, auth-proxy는 HMAC-SHA256으로 조회했다.
> 각 레포의 코드는 자기 문서와 일치했고, 두 문서가 서로를 모르고 있었다.
> 결과는 **발급된 모든 토큰이 401**.

계약을 한쪽 레포에 두면 상대 레포는 그것을 자기 문서로 인식하지 않는다. 그래서 여기 둔다.

## 계약 목록

| 계약 | 당사자 | 상태 |
|---|---|---|
| [`enrollment-api.md`](enrollment-api.md) | `telemetryctl` ↔ `pulsemetry-backend` | 확정 |
| [`telemetry-ingest.md`](telemetry-ingest.md) | dev: `telemetryctl` → `pulsemetry-backend` (·`infra`); 과거 경로: `ai-telemetry-pipeline` | 변경 중 — dev EKS 실행 경로·B3 전제 정리 (ADR 0023); 실제 배선·기능 검증 별도 |
| [`data-model.md`](data-model.md) | `pulsemetry-backend` ↔ 읽기 소비자 (`telemetry-ingest`·기존 `ai-telemetry-pipeline`), 배포는 `infra`, 설계도는 `rdb-schema` | 변경 중 — dev enrollment Flyway 선행·소비자 실행 경계 (ADR 0023) |
| [`dashboard-api.md`](dashboard-api.md) | `pulsemetry-frontend` ↔ `pulsemetry-backend` ↔ `infra` | 변경 중 — dashboard ADR 0021·JWT ADR 0022 유지, dev 배포 부분은 ADR 0023으로 대체; live 배포 별도 |

PROJ-250의 세 계약은 [dev EKS 전용 ADR 0023](../adr/0023-dev-backend-eks-only.md)에 따라 정합화한다.
PROJ-248·249의 관련 구현과 계약이 머지되기 전에는 변경 중으로 둔다. 과거 조사 항목은 해당 문서의
적용 범위를 함께 읽으며, 문서 갱신이나 이미지 게시 성공을 live E2E 완료로 해석하지 않는다.

## 규칙

1. **계약 변경 PR은 상대 레포 담당자가 리뷰어다.** 한쪽 레포의 사정만으로 고치지 않는다.
2. **계약 문서와 구현이 어긋나면 계약이 기준이다.** 구현을 고치거나, 계약을 바꾸려면 먼저 이 문서를 고치고 양쪽 합의를 받는다.
3. **계약 문서는 현재 합의된 상태를 적는다.** 그 결정의 *이유*는 ADR에 있다 — 이 문서는 "무엇에 합의했는가", ADR은 "왜 그렇게 정했는가".
4. **기계 판독 가능한 원본이 있으면 그것이 우선한다.** 예: manifest의 JSON Schema는
   `telemetryctl/contracts/*.schema.json`이다. 이 문서는 그 스키마를 서술할 뿐 대체하지 않는다.
5. **미해결 항목을 감추지 않는다.** 각 계약 문서 말미에 "미해결" 절을 두고, 무엇이 아직 배선되지 않았는지 적는다.

## 계약을 바꾸는 절차

1. 이 디렉토리의 해당 문서를 고치는 PR을 연다. 리뷰어에 **양쪽 레포 담당자**를 넣는다.
2. 결정의 이유가 새로 생겼으면 크로스레포 ADR을 함께 쓴다([`../adr/README.md`](../adr/README.md)의 스코프 규칙).
3. 합의 후 각 레포의 구현 PR을 연다. 구현 PR은 이 계약 문서를 링크한다.
4. 양쪽이 머지될 때까지 계약 문서의 상태 표기를 "변경 중"으로 둔다.
