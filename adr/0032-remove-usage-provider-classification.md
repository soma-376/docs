# 0032. 사용량의 호출 공급자 분류를 제거한다

## Status

Accepted — PROJ-320 사용자 결정. backend ADR 0020의 공급자 미확인 품질 플래그 생성과 ADR 0023의 공급자 기반 모델 매핑을 부분 대체한다. 원본 보존·집계·snapshot 불변성은 유지한다.

## Context

pulsemetry-backend와 pulsemetry-frontend가 당사자다. 수집된 도구명·모델명·고정 gen_ai.system 값만으로 실제 호출 공급자를 판별할 수 없다. 실제 호출 공급자와 모델 제작사는 서로 다른 정보이며, 모델 제작사 배지는 유지한다.

## Decision

- 실제 호출 공급자 판별·공급자 기반 별칭 통합만 제거한다. 모델 제작사를 실제 호출 공급자로 간주하지 않는다.
- 도넛의 모델 제작사(OpenAI·Anthropic) 배지·필터와 모델 선택·키보드 조작을 유지한다. 확인된 제작사가 둘 이상일 때만 전체 및 해당 제작사 배지를 표시하며, 하나 이하면 배지를 숨긴다. 미확인 모델은 전체에 남긴다.
- 새 정규화는 provider_unresolved를 만들지 않는다. 기존 기록을 읽기 위한 어휘와 원본 OTel metadata는 보존한다.
- 기존 모델 ID `unknown/<product>/<escaped model>`와 model-id-v1 판을 유지한다. unknown 접두어는 내부 식별자 호환용이며 화면 분류가 아니다.
- ClickHouse snapshot의 provider 열은 호환용 NULL 슬롯으로 남긴다. 배포된 DDL·기존 원본을 수정하지 않는다.
- 토큰·환산액·관측 수와 모델별 집계는 바꾸지 않는다. 실제 제품·좌석·계약 벤더 관리도 범위 밖이다.

## Alternatives Considered

모델명으로 호출 공급자를 추정하는 방식은 경유 경로를 구분하지 못하므로 채택하지 않는다. 컬럼 즉시 삭제는 기존 뷰·배포 호환성을 깨므로 분리한다.

## Consequences/Tradeoffs

### Positive

확인되지 않은 공급자 분류 없이 관측된 모델과 사용량을 표시한다.

### Negative

호출 공급자별 분석은 제공하지 않는다. 호환 컬럼과 과거 품질 플래그는 남는다. 재도입에는 실제 호출 근거와 새 계약이 필요하다.

## Acceptance Criteria

- 모델 제작사 배지 동작을 유지하며 전체 모델 비중과 null·0 의미를 변경하지 않는다.
- 정규화의 공급자 플래그 제거 외 토큰·비용·메타데이터를 보존한다.
- 실제 ClickHouse snapshot의 모델별 합계와 기존 모델 ID를 검증한다.
