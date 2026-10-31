# AGENTS.md — Pulsemetry 문서 허브

Pulsemetry는 Claude Code·Codex 등 개발 AI 도구의 사용량과 비용을 조직 → 팀 → 구성원 축으로 모아 보여주는
사내 통합·가시화 플랫폼이다.

- 제품·아키텍처·레포 간 계약의 **단일 출처는 이 레포(`soma-376/docs`)**다. 형제 체크아웃 `../docs`를 우선 참조한다.
- 기능 작업 전에는 `spec` 스킬, 설계 관련 작업 전에는 `adr` 스킬을 쓴다.
- **코드와 ADR이 어긋나면 ADR이 기준이다.** 결정을 바꾸려면 `adr-new`로 개정 ADR을 먼저 쓴다.
- git 작업은 `CONVENTION.md`를 따른다 (`conventions` 스킬).
- 스킬이 보이지 않으면 형제 `../agent-skills` 클론 여부를 확인하고, 없으면 사용자에게 클론을 안내한다.
- 문서·주석은 한국어, 코드·파일명은 영어.
- **사용자의 명시 요청 없이 `git push` 하지 않는다.**

---

## 이 레포에서 작업할 때

### 문서 스타일

- 만연체를 쓰지 않는다. 한 문단은 하나의 주장만 담는다.
- 사실과 추론을 구분해 적는다. 확정된 것, 합의 대상인 규칙, 아직 정해지지 않은 것을 섞지 않는다.
- 최종 형태만 적는다. 개발 환경의 임시 구성, 아직 배선되지 않은 것, "향후 도입 예정"은 본문에 넣지 않는다.
  현재 상태를 적어야 할 때는 그 문서의 "현재 상태" 절에 따로 모은다.
- 작성일은 문서 본문에 적지 않는다. `git log --follow <파일>`로 확인한다.

### ADR 인덱스 동시 갱신 의무

`adr/`에 파일을 추가·개정하면 **같은 커밋에서** `adr/README.md`의 표를 갱신한다.
supersede 할 때는 기존 ADR의 `Status`도 함께 고친다. 인덱스가 파일 목록과 어긋나는 커밋은 만들지 않는다.

### 계약 문서를 고칠 때

`contracts/`의 문서는 당사자가 둘이다. 한쪽 레포의 사정만으로 고치지 않는다.
계약 변경 PR은 상대 레포 담당자가 리뷰어다. 자세한 규칙은 [`contracts/README.md`](contracts/README.md).

### `archive/`를 신뢰하지 않는다

`archive/`의 PLAN·SA·AA·IA는 증류 이전 원본이다. 현재 구조와 다른 서술이 남아 있다.
**질문에 답할 때 `archive/`를 근거로 쓰지 않는다.** 증류본(`product/`·`architecture/`·`contracts/`)이 기준이다.
증류본에 없는 내용이 필요하면 archive에서 찾되, 그 내용을 증류본으로 옮기고 나서 인용한다.

### 문서 배치

새 내용을 어디에 둘지는 [`README.md`](README.md)의 배치 판별 규칙을 따른다.
한 레포만 아는 내용을 이 허브에 쓰지 않는다.

## 브랜치

장수 브랜치는 `main` 하나다 — 코드 레포 4종과 달리 `develop`을 두지 않는다.

- **오타·링크·상태 갱신** → main 직접 커밋 (문서 운영 계획 §3).
- **계약·ADR·구조 변경** → `<타입>/<티켓키>-<슬러그>` 브랜치에서 **`main`으로 직접 PR.**
  계약은 상대 레포 담당이, 그 외는 아무나 1명이 리뷰한다.

협업 컨벤션 v2의 브랜치명·커밋 형식은 그대로 따르되, 베이스만 develop이 아니라 main이다.
`.github/workflows/`의 PR 자동화(제목 채움·담당자 지정)도 main 대상 PR에 걸려 있다.

## PROJ-281 dev GitOps 배포 계약

[ADR 0030](adr/0030-dev-gitops-delivery.md)과 [generated 공급 계약](contracts/generated-values-sync.md)은
승인된 설계다. backend의 같은 SHA·run/attempt 네 앱 완료 manifest를 검증하고, candidate/dev의
image.ref만 Updater가 수정한다. infra가 C·V·B를 검증해 main에 snapshot V를 먼저 게시한 뒤
별도 P에서 desired와 Application source를 함께 선택한다. 현재 상태·구현·실제 인수를 구분한다.
Argo/Updater 자체는 수동 Helm, 인수한 dev Application은 auto sync/self-heal·prune off다.
PROJ-281은 최신 main을 병합한 PR #16의 PROJ-276 선행 브랜치를 계승한다. 기존 generated 사본
ADR 0026과 Kafka ADR 0026은 파일명을 유지하므로 범위·파일 링크로 구분한다. 사본 원칙은
ADR 0030이 계승하고 image 필드 예외·V/P 승격을 확장한다. PR #11은 미병합 참고이며
선행 조건이 아니다. ADR 0023은 과거 시드다. 로컬 계승과 원격 PR 병합·실제 인수를 구분한다.
다음 ADR은 `0031`부터이며 실제 작성 전 인덱스를 다시 확인한다.
