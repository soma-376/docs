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

[ADR 0035](adr/0035-dev-gitops-delivery.md)과 [공급 계약](contracts/generated-values-sync.md)은
Image Updater 도입 철회와 workflow 기반 게시·복구 설계다. backend는 원본 게시 workflow 최종 성공 뒤
별도 승격으로 같은 SHA/run/attempt 네 ARM64 이미지의 B를 선택하고 image·rollback을 게시한다.
infra config publisher는 선택 B를 보존하며 설정을 재생성한다. backend는 C와 비이미지 바이트를 보존한다.
두 publisher는 private main의 delivery-state.json·CAS·owner/generation 공통 잠금을 배포·기능 검증까지
보유한다. 상태 commit과 P를 구별하며 repository별 concurrency·timeout 잠금 탈취·force push는 금지한다.
현재 revision 실패가 확정되면 실제 schema/runtime/key/TLS 호환성을 확인한 이전 정상 B로 한 번만 자동
image rollback한다. unknown·config 실패·비호환·소유권 유실·복구 실패는 blocked다. 일반 image rollback은
Argo auto sync/self-heal을 유지하며 offset/key/인증서 유지보수는 별도 중지 절차다.
실패 B·전체 이미지 벡터 재게시를 차단하고 실패 source의 strict descendant이면서 원본 게시 run이 영속
failure fence 이후 생성된 정상 후보만 자동 재개한다. 잠금 해제 시 이미 완료된 적격 후보도 다시 조회한다.
Updater·candidate branch·tracking tag 발견·image.ref 전환·별도 overlay는 없고 repository+digest를 유지한다.
Argo 자체만 수동 Helm, 인계한 Application은 auto sync/self-heal·prune off다. 인계 전 PROJ-279는 최초
사본부터 성공한 CDK 배포 뒤 CD가 YAML 13개와 C를 공급하며 Helm 적용은 수동이다. 승인 plan에 활성 여부와
로그 기반 이미지 입력을 고정하고 원본 배포 receipt로 사본 재시도를 검증한다. 이 입력은 B가 아니다.
metadata-only 조회·14개 파일 허용목록을 유지하고 metadata/state 존재 시 중단한다. Argo 인계 전 writer를
중지하고 실제 실행 이미지의 H를 검증한다. 실제 구현은 backend·infra가 소유하며 운영 상태는 각 레포의 수락 근거로 확인한다.
다음 ADR은 `0036`부터이며 실제 작성 전 인덱스를 다시 확인한다.

## PROJ-311 이미지 승격 구현 경계

허브 GitOps ADR은 선행 예약 번호를 보존해 `0035`, private 사본 ADR은 `0034`로 이관한다.
공개 완료 manifest는 account 없는 네 repository/digest와 이미지 계약을 담는다. 별도 default-branch 승격
workflow가 원본 develop 생산 실행을 검증하며 보호된 `dev-gitops` 역할은 기존 ECR push와 분리한다.
image publisher의 비이미지 예외는 공급 계약의 deliveryVerification 여섯 leaf뿐이다. PostSync Job은 새 V마다
검사하고 runner가 현재 C/V/P·operation·Job UID를 연결한다. 원문 없는 증거는 private dev/evidence에 둔다.
자동 복구 v1은 실제 schema/runtime/key/TLS/envelope fingerprint가 마지막 정상과 모두 같은 경우 한 번이다.
PROJ-310 운영 인수·정상 B·별도 infra 설정 publisher/CAS 수락 전 자동화 스위치는 비활성이다.
