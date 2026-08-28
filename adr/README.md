# ADR — 크로스레포 인덱스와 전 레포 카탈로그

## 스코프 규칙 — 어느 ADR을 어디에 쓰는가

| 결정의 성격 | 위치 |
|---|---|
| **2개 이상 레포의 코드·계약에 영향**을 주거나 **제품 수준** 결정 | **이 허브 `adr/`** |
| 단일 레포의 구현 결정 | **그 레포의 `docs/adr/`** |

판단이 애매하면 이렇게 묻는다: *"이 결정을 뒤집으려면 몇 개 레포의 PR이 필요한가?"*
둘 이상이면 허브다.

### contracts와의 관계

- [`../contracts/`](../contracts/README.md)는 **현재 합의된 상태**를 적는다 — "무엇에 합의했는가".
- ADR은 **그 결정의 이유**를 적는다 — "왜 그렇게 정했는가".
- 계약을 바꾸는 PR은 보통 둘 다 건드린다. 계약 문서를 고치면서 이유가 새로 생겼으면 ADR을 함께 쓴다.

### 인덱스 동시 갱신 의무

`adr/`에 파일을 추가·개정하면 **같은 커밋에서** 아래 표를 갱신한다.
supersede 할 때는 기존 ADR의 `Status`도 함께 고친다. 인덱스가 파일 목록과 어긋나는 커밋은 만들지 않는다.

---

## 크로스레포 ADR

| 번호 | 제목 | Status |
|---|---|---|
| 0001 | [OTLP 인증은 불투명 토큰을 애플리케이션 계층에서 검증한다](0001-otlp-authentication-model.md) | Accepted |
| 0002 | [manifest 배정 단위는 tenant다](0002-manifest-assignment-unit-is-tenant.md) | Accepted |
| 0003 | [텔레메트리 파이프라인은 별도 레포로 유지하고 collector만 backend로 이관한다](0003-telemetry-pipeline-repo-boundary.md) | Accepted |

새 ADR은 `0004`부터. [`0000-adr-template.md`](0000-adr-template.md)의 구조를 따른다.

**크로스레포 ADR 후보** (아직 결정되지 않았거나 결정이 문서화되지 않은 것):

- Dashboard API 서버의 소재 — `pulsemetry-backend` 모듈인가 별도 레포인가 ([`../contracts/dashboard-api.md`](../contracts/dashboard-api.md) §2)
- 계약 응답 스키마의 버저닝 정책 — tolerant reader로 갈 것인가 ([`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §6 M7)
- Provider 공시 단가표의 소유·갱신 주체와 저장 위치 ([`../product/prd.md`](../product/prd.md) §8-5)

**후보에서 결정으로 옮겨간 것:**

- 텔레메트리 파이프라인의 소유 레포 → [ADR 0003](0003-telemetry-pipeline-repo-boundary.md)
  (전체 병합 기각 — backend ADR-0006은 `Rejected`로 닫혔다)
- OTLP 인증 모델 → [ADR 0001](0001-otlp-authentication-model.md)
- manifest 배정 단위 → [ADR 0002](0002-manifest-assignment-unit-is-tenant.md)
- enrollment 스키마의 부트스트랩 주체 → 결정 완료(backend Flyway — backend ADR-0009,
  [`../contracts/data-model.md`](../contracts/data-model.md) §5). 남은 것은 ECS 실행 자리(infra 새 ADR 예정)

---

## 전 레포 카탈로그

**레포 단위 링크만 둔다.** 개별 ADR을 여기 미러링하지 않는다 — 개별 ADR의 단일 출처는 각 레포의 인덱스다.
이 표를 개별 ADR 수준으로 늘리면 스테일이 구조적으로 발생한다.

| 레포 | ADR 위치 | 인덱스 | 건수 | 번호 규칙 · 파일명 관례 |
|---|---|---|---|---|
| `infra` | `docs/adr/` | ✅ `docs/adr/README.md` | 23 (0001–0019, 0021–0024) | **영어 슬러그.** `0020`은 로그 그룹 정책용 **예약**. 새 ADR은 `0025`부터 |
| `pulsemetry-backend` | `docs/adr/` | ✅ `docs/adr/README.md` | 9 (0001–0009) | **한국어 슬러그.** 새 ADR은 `0010`부터 |
| `telemetryctl` | `docs/adr/` | ✅ `docs/adr/README.md` | 8 (0001–0008) | **한국어 슬러그.** 새 ADR은 `0009`부터 |
| `ai-telemetry-pipeline` | `docs/adr/` | ❌ 없음 | 6 (0001–0006) + 템플릿 | **한국어 슬러그**(레포 `AGENTS.md`·템플릿 선언). 새 ADR은 `0007`부터 |
| `team-376-llm-wiki` | `wiki/decisions/` | `index.md` | 다수 | **ADR이 아니다** — 회의에서 나온 결정의 기록. 코드 구조를 구속하지 않는다 |
| `docs` (이 레포) | `adr/` | 위 표 | 3 | 크로스레포·제품 결정만. **영어 슬러그** |

`rdb-schema`·`otel-collector`·`.github`·`agent-skills`에는 ADR이 없다.

### 인덱스가 없는 레포에 ADR을 추가할 때

인덱스가 없는 레포는 `ai-telemetry-pipeline` 하나다
(`pulsemetry-backend`·`telemetryctl`은 PROJ-80에서 infra 형식의 인덱스를 만들었다).
그 레포에 ADR을 추가할 때 인덱스를 새로 만들지는 **사용자에게 물어본다** — infra 형식을 따르면 된다.

### ADR을 각 레포에 분산해 두는 이유

[`../README.md`](../README.md)의 "ADR을 각 레포에 분산해 두는 이유" 참조.

### 파일명 관례

레포마다 다르다. **그 레포의 기존 관례를 따른다** — infra와 이 허브는 영어 슬러그, backend·telemetryctl·ai-telemetry-pipeline은 한국어 슬러그다.
전 레포 통일은 하지 않는다. 파일명을 바꾸면 기존 ADR 간 상호 링크가 깨진다.
