# NNNN. 제목 (결정을 평서문으로)

## Status

`Proposed` · `Accepted` · `Superseded by [ADR NNNN](NNNN-슬러그.md)` 중 하나.
대체 관계가 있으면 무엇이 대체됐고 무엇이 유효한지 한 줄로 함께 적는다.

## Context

무엇이 문제인가. 어떤 제약이 있었나. **결정하지 않았을 때 무슨 일이 일어나는가.**
사실만 적는다 — 여기서 결정을 미리 말하지 않는다.

크로스레포 ADR이면 **어느 레포들이 걸리는지** 여기에 명시한다.

## Decision

무엇을 하기로 했는가. 평서문 현재형으로 적는다.

## Constraints *(선택)*

이 결정이 지켜야 하는 외부 제약.

## Alternatives Considered

각 대안과 **왜 택하지 않았는지**. 대안이 하나뿐이었으면 그렇게 적는다.

## Consequences/Tradeoffs

### Positive

### Negative

감수하기로 한 대가. 여기가 비어 있으면 대개 검토가 덜 된 것이다.

## Follow-up *(선택)*

재검토 조건과 미결 질문. **"언제 이 결정을 다시 볼 것인가."**

## Acceptance Criteria *(선택)*

## References *(선택)*

관련 ADR, 계약 문서, 코드 위치.

---

**작성 규칙**

- 섹션 순서는 위와 같다. 괄호로 표시한 것은 내용이 있을 때만 둔다.
- **작성일은 문서에 적지 않는다.** `git log --follow <파일>`로 확인한다.
- 파일명은 그 레포의 관례를 따른다 (infra 영어 슬러그, backend·telemetryctl 한국어 슬러그).
- 추가·개정 시 **같은 커밋에서** 해당 인덱스 README의 표를 갱신한다.
