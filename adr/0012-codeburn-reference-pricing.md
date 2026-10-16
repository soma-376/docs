# 0012. Codeburn의 모델 단가와 명시적 별칭을 참조 환산에 사용한다

## Status

Accepted — 사용자의 명시 요청으로 ADR 0010의 공개 단가만 허용하는 모델 범위와 자동 검토 null 정책을 부분 대체한다. 이전 가격판 불변성, 원천 보존, 환산액과 실제 청구액의 분리는 유지한다.

## Context

설치된 Codeburn 0.9.24는 `codex-auto-review`를 `gpt-5.5`의 참조 가격으로 계산한다. 사용자는 Codeburn의 Codex·Claude 관련 단가 전체를 프로젝트에 적용하도록 요청했다. backend의 가격 선택과 frontend에 전달되는 사용 환산액의 의미에 영향을 준다.

## Decision

- Codeburn 0.9.24의 번들 가격표, 보완 가격표, 명시적 별칭을 추출한 고정 카탈로그를 새 참조 가격판으로 추가하고 기본 배선에 사용한다. 파일 해시·원본 버전·추출 기준·제외 범위를 기록한다. 실행 중 네트워크 조회나 사용자 Codeburn 설정에 의존하지 않는다.
- Claude 및 OpenAI/Codex의 텍스트 토큰 단가와 명시적 별칭을 반영한다. 이미지·음성·실시간 전용 과금과 Batch 전용 행은 현재 Standard 텍스트 평가에 섞지 않는다. 전체 후보와 제외 사유는 추출 명세에 남긴다.
- `codex-auto-review → gpt-5.5`는 환산 기준 선택이다. 실제 backend 모델의 동일성이나 실제 청구액을 확인했다고 표시하지 않는다. 원본 `model`은 유지하고 가격판과 카탈로그의 `reference_model`로 근거를 추적한다.
- 기존에 직접 검증한 v4의 정확한 모델 단가와 요청별 장문 요금은 유지한다. 이외 모델은 Codeburn의 토큰별 요율을 적용한다. Claude 캐시 생성은 기존 사용자 요청대로 5분 참조 평가다.
- 입력·캐시·출력 의미 검증, 누락 토큰의 null, 완전히 모르는 모델의 null 정책은 유지한다. Codeburn의 임의 접두사 매칭과 미가격 0원 처리는 가져오지 않는다.
- 기존 가격판은 수정하지 않는다. 새 규칙은 정규화 판 9로 구분한다. 기존 저장 관측 재평가는 별도 실행이며 기존 관측 식별자·날짜·토큰·원문 보존 규칙을 따른다.

## Alternatives Considered

- 1개 자동 검토 별칭만 추가: 관련 모델 전체를 반영하라는 요청을 충족하지 못한다.
- Codeburn 실행 중 캐시를 서버에서 직접 읽기: 사용자 환경·캐시 갱신에 따라 같은 판의 결과가 달라진다.

## Consequences/Tradeoffs

### Positive

- 자동 검토를 포함해 훨씬 넓은 모델 범위를 재현 가능한 참조 가격으로 계산한다.
- 기존 원본과 이전 평가판을 보존한다.

### Negative

- Codeburn의 대체 모델 선택은 추정 기준이며 실제 사용 모델·구독 비용을 증명하지 않는다.
- 새 모델과 요금 변경은 새 고정 가격판으로 갱신해야 한다.

## References

- [Codeburn 0.9.24](https://github.com/getagentseal/codeburn/tree/v0.9.24)
- backend ADR 0065 및 `docs/pricing/codeburn-reference.md`
