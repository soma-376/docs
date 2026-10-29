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
| 0001 | [OTLP 인증은 불투명 토큰을 애플리케이션 계층에서 검증한다](0001-otlp-authentication-model.md) | Accepted (부분 대체 — [ADR 0005](0005-single-app-telemetry-topology.md)); dev Kafka 부분 대체: [ADR 0026](0026-dev-telemetry-kafka-strimzi.md) |
| 0002 | [manifest 배정 단위는 tenant다](0002-manifest-assignment-unit-is-tenant.md) | Accepted |
| 0003 | [텔레메트리 파이프라인은 별도 레포로 유지하고 collector만 backend로 이관한다](0003-telemetry-pipeline-repo-boundary.md) | Superseded by [ADR 0004](0004-telemetry-pipeline-repo-merge.md) |
| 0004 | [텔레메트리 파이프라인을 backend 레포로 병합한다](0004-telemetry-pipeline-repo-merge.md) | Accepted |
| 0005 | [텔레메트리 파이프라인 전 계층을 단일 Spring 애플리케이션으로 띄운다](0005-single-app-telemetry-topology.md) | Accepted; dev Kafka 부분 대체: [ADR 0026](0026-dev-telemetry-kafka-strimzi.md) |
| 0006 | [OTLP ingest 는 영구 실패를 4xx 로, 일시 실패를 503 으로 돌려주고 배치 큐를 두지 않는다](0006-otlp-ingest-retry-and-status-contract.md) | Accepted; dev Kafka 부분 대체: [ADR 0026](0026-dev-telemetry-kafka-strimzi.md) |
| 0007 | [분석 snapshot과 수집 생애주기 요약을 원본 저장소와 분리한다](0007-dashboard-snapshots-and-telemetry-lifetime-summary.md) | Proposed |
| 0008 | [사용 환산액은 버전을 고정한 참조 단가로 평가한다](0008-current-public-reference-valuation.md) | Accepted |
| 0011 | [계산 가능한 관측 부분합을 전체 확정값과 분리해 표시한다](0011-known-usage-subtotals.md) | Accepted |
| 0022 | [프론트 이미지는 브랜치 머지 후 ECS에 자동 배포한다](0022-frontend-ecs-image-delivery.md) | Accepted |
| 0023 | [dev 과거 시드는 독립 파일과 제한 속도로 적재하고 검증 후 SSO와 알림을 활성화한다](0023-dev-historical-dashboard-dataset.md) | Accepted |
| 0024 | [dev 시드의 지연 감시는 왕복 시간과 HTTP 응답 요약 시간을 구분한다](0024-dev-seed-http-summary-latency.md) | Accepted |
| 0025 | [초대 설치 링크와 사전 검증은 최종 enroll 소비와 분리한다](0025-invitation-installation-delivery.md) | Accepted |
| 0026 | [dev 텔레메트리 비동기 수집은 Strimzi Kafka와 mTLS를 사용한다](0026-dev-telemetry-kafka-strimzi.md) | Accepted (배포 검증 별도) |

| 0027 | [설치 과정에서 데스크톱 앱을 함께 설치하고 정책 적용 상태를 표시한다](0027-install-desktop-and-report-policy-status.md) | Accepted |

새 ADR은 `0028`부터. [`0000-adr-template.md`](0000-adr-template.md)의 구조를 따른다.

ADR 0007은 검토 중인 제안이다. ClickHouse snapshot 쓰기와 RDS 운영 요약의 경계는 아직 현행 계약을 대체하지 않는다.

**크로스레포 ADR 후보** (아직 결정되지 않았거나 결정이 문서화되지 않은 것):

- Dashboard API 서버의 소재 — `pulsemetry-backend` 모듈인가 별도 레포인가 ([`../contracts/dashboard-api.md`](../contracts/dashboard-api.md) §2)
- 계약 응답 스키마의 버저닝 정책 — tolerant reader로 갈 것인가 ([`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §6 M7)

**후보에서 결정으로 옮겨간 것:**

- 참조 단가의 평가 기준·출처·버전 관리와 저장 사용량 재평가 → [ADR 0008](0008-current-public-reference-valuation.md)
- 텔레메트리 파이프라인의 소유 레포 → [ADR 0004](0004-telemetry-pipeline-repo-merge.md)
  (backend 레포로 병합. **대체됨(ADR 0004)** — 0003의 전체 병합 기각 결정은 더 이상 유효하지 않다)
- OTLP 인증 모델 → [ADR 0001](0001-otlp-authentication-model.md)
- 파이프라인 내부 토폴로지(인증 위치·Collector 존치) → [ADR 0005](0005-single-app-telemetry-topology.md)
- OTLP ingest 의 재시도·백프레셔 정책과 503 계약 → [ADR 0006](0006-otlp-ingest-retry-and-status-contract.md)
  (ADR 0005 Follow-up 이 열어 둔 항목)
- manifest 배정 단위 → [ADR 0002](0002-manifest-assignment-unit-is-tenant.md)
- enrollment 스키마의 부트스트랩 주체 → 결정 완료(backend Flyway — backend ADR-0009,
  [`../contracts/data-model.md`](../contracts/data-model.md) §5). 남은 것은 ECS 실행 자리(infra 새 ADR 예정)

---

## 전 레포 카탈로그

ADR 0006의 `Retry-After`는 서버의 최소 대기 제안이며, 최종 대기 시간은 데몬의 백오프와 상한을 따른다.

**레포 단위 링크만 둔다.** 개별 ADR을 여기 미러링하지 않는다 — 개별 ADR의 단일 출처는 각 레포의 인덱스다.
이 표를 개별 ADR 수준으로 늘리면 스테일이 구조적으로 발생한다.

| 레포 | ADR 위치 | 인덱스 | 건수 | 번호 규칙 · 파일명 관례 |
|---|---|---|---|---|
| `infra` | `docs/adr/` | ✅ `docs/adr/README.md` | 23 (0001–0019, 0021–0024) | **영어 슬러그.** `0020`은 로그 그룹 정책용 **예약**. 새 ADR은 `0025`부터 |
| `pulsemetry-backend` | `docs/adr/` | ✅ `docs/adr/README.md` | 64 (0001–0064), PR #37 기준 | **한국어 슬러그.** 다음 번호는 대상 브랜치 인덱스에서 확인 |
| `telemetryctl` | `docs/adr/` | ✅ `docs/adr/README.md` | 18 (0001–0018), [PR #37의 `a55e944` 기준](https://github.com/soma-376/telemetryctl/blob/a55e944/docs/adr/README.md) | **한국어 슬러그.** 해당 ref의 다음 번호는 `0019`; 작성 시 대상 브랜치 인덱스에서 다시 확인 |
| `ai-telemetry-pipeline` | `docs/adr/` | ❌ 없음 | 6 (0001–0006) + 템플릿 | **한국어 슬러그**(레포 `AGENTS.md`·템플릿 선언). 새 ADR은 `0007`부터 |
| `team-376-llm-wiki` | `wiki/decisions/` | `index.md` | 다수 | **ADR이 아니다** — 회의에서 나온 결정의 기록. 코드 구조를 구속하지 않는다 |
| `docs` (이 레포) | `adr/` | 위 표 | 14 (템플릿 제외) | 크로스레포·제품 결정만. **영어 슬러그** |

`rdb-schema`·`otel-collector`·`.github`·`agent-skills`에는 ADR이 없다.

### 인덱스가 없는 레포에 ADR을 추가할 때

인덱스가 없는 레포는 `ai-telemetry-pipeline` 하나다
(`pulsemetry-backend`·`telemetryctl`은 PROJ-79에서 infra 형식의 인덱스를 만들었다).
그 레포에 ADR을 추가할 때 인덱스를 새로 만들지는 **사용자에게 물어본다** — infra 형식을 따르면 된다.

### ADR을 각 레포에 분산해 두는 이유

[`../README.md`](../README.md)의 "ADR을 각 레포에 분산해 두는 이유" 참조.

### 파일명 관례

레포마다 다르다. **그 레포의 기존 관례를 따른다** — infra와 이 허브는 영어 슬러그, backend·telemetryctl·ai-telemetry-pipeline은 한국어 슬러그다.
전 레포 통일은 하지 않는다. 파일명을 바꾸면 기존 ADR 간 상호 링크가 깨진다.
