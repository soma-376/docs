# 0010. 공개 단가가 확인된 Luna 모델만 새 참조 가격판에 추가한다.

## Status

Accepted — ADR 0009의 현재 기본 가격판 v3를 v4로 개정한다. 이전 가격판 불변성, 현재 참조 평가와 실청구의 분리, 원천 보존, 미확인 모델의 null 정책은 유효하다.

## Context

사용자는 테스트 데이터와 분리한 실제 로컬 9월 원본을 날짜·토큰 수 그대로 적재하고 현재 단가로 평가하도록 요청했다. 원본의 모델 중 `gpt-5.6-luna`·`gpt-6-luna`는 공개 단가가 있지만 v3에 없고, `codex-auto-review`는 공개 가격이 확인되지 않는다. backend가 가격과 분석 판을 저장하고 frontend는 저장된 값과 가용성을 표현하므로 두 레포의 해석 경계에 걸린다.

## Decision

1. Standard/global/USD 참조 평가의 불변 v4에 정확한 `gpt-5.6-luna`와 `gpt-6-luna`만 추가한다. 공식 모델 문서와 가격표를 읽은 기준 시각은 `2026-10-16T08:03:52Z`다. 이전 v1/v2/v3의 ID·시각·정책·단가는 바꾸지 않는다.
2. 백만 토큰당 input/read/write/output 단가는 각각 GPT-5.6 Luna `0.20/0.02/0.25/1.20`, GPT-6 Luna `0.10/0.01/0.125/0.50` USD다. 단일 요청의 전체 입력이 272,000을 초과하면 전체 입력/cache 단가에 2, output 단가에 1.5를 적용한다.
3. 현재 공개 단가에 의한 토큰 환산액이며 구독·실청구·사용 당시 가격을 복원했다고 주장하지 않는다. 원본 모델 ID·시각·토큰을 가격표에 맞춰 바꾸지 않는다. 검증된 비중복 토큰 의미가 별도로 충족되어야 한다.
4. `codex-auto-review`는 정확 ID의 공개 단가나 동일 모델 별칭 근거가 확인될 때까지 null이다. API-key 환경에서 같은 역할에 Luna가 선택된다는 사실을 모델 동일성으로 해석하지 않는다. 이 행을 삭제하거나 0원으로 만들지 않는다. 부분합을 제공할 때는 제외 범위가 드러나야 한다.
5. backend 정규화 판은 7로 올린다. 기존 테스트 조직은 재처리하지 않고 실제 9월 원본을 새 tenant에 적재한다. 적재·가격·화면 검증은 원본 내용을 모델 입력으로 보내지 않는 로컬 집계로 대조한다.

## Alternatives Considered

### 기존 v3에 두 모델을 추가한다

같은 가격판으로 다른 지원 범위가 생기므로 채택하지 않는다.

### 자동 검토 모델을 Luna로 간주한다

클라이언트가 인증 경로별로 다른 ID를 고른다는 근거만 있고 같은 backend 모델이나 단가라는 근거가 없어 채택하지 않는다.

## Consequences/Tradeoffs

### Positive

- 공개 단가가 확인된 실제 모델을 원천 변경 없이 계산한다.
- 과거 평가판을 재현하면서 새 지원 범위를 구분한다.

### Negative

- 자동 검토 사용량이 포함된 전체 환산액은 여전히 미확인일 수 있다.
- 9월 실제 청구 금액·구독 소진액과 현재 토큰 환산액은 다를 수 있다.

## References

- [ADR 0009](0009-current-reference-cache-benchmark-and-revaluation.md)
- [GPT-5.6 Luna](https://developers.openai.com/api/docs/models/gpt-5.6-luna)
- [GPT-6 Luna](https://developers.openai.com/api/docs/models/gpt-6-luna)
- [OpenAI 가격표](https://developers.openai.com/api/docs/pricing)
