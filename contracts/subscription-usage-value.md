# 구독료 대비 사용 환산액 — 후속 구현 계약

## 현재 상태

이 문서는 [ADR 0032](../adr/0032-limit-team-tabs-and-design-subscription-usage-ratio-separately.md)의 후속 구현 인계 명세다. 제품 의미·계산·실패 정책은 채택했으며 아래 데이터 모델과 API는 구현안이다. PROJ-318에서는 카드, API 필드, 계약 이력 명령, migration을 구현하지 않는다. 현재 개요 카드·세션 필드·계약 API의 의미를 바꾸지 않는다.

당사자는 frontend와 backend다. 계약 이력 쓰기는 backend enrollment, 분석과 불변 계산 근거는 dashboard, 표시와 상세 근거 열람은 frontend가 소유한다. 원천의 `monthlySeatFeeUsd`, 계약 기간, 수정 버전만으로 과거 계약액을 복원할 수 있다고 가정하지 않는다.

## 1. 의미와 대상

카드 제목은 **구독료 대비 사용 환산액**이다. 조직 전체·선택 기간에 대해 다음 비율을 제공한다. 팀·구성원·모델별 계약액 배분과 이전 기간 증감은 v1에서 제공하지 않는다.

```text
비율(%) = 대상 제품의 관측된 사용 환산액
          ÷ 같은 제품·기간의 배분 계약액 × 100
```

분자는 관측 행에 저장된 USD 환산액의 합이다. 현재 계약 단가나 현재 참조 단가표로 과거 관측을 다시 평가하지 않는다. 분모는 구매한 **전체 좌석** 계약 월 총액을 기간에 배분한 금액이다. 사용·배정·활성 인원이 적어도 분모를 줄이지 않는다. 기존 `monthlySeatFeeUsd`는 `Σ(좌석별 월 금액 × 좌석 수)`이므로 좌석 수를 다시 곱하지 않는다.

이 비율은 실제 청구액·절감률·수익률·생산성이 아니다. 같은 도구에서 관측한 구독 포함 사용과 별도 API 과금 사용을 함께 포함한다. 웹 등 수집하지 않는 사용은 포함하지 않는다. 100%를 상한으로 제한하지 않는다.

### 같은 제품 범위

| 관측 도구 kind | 계약 제품 kind |
|---|---|
| `claude_code` | `claude_team` |
| `codex` | `openai_biz` |

이 명시적 카탈로그 매핑의 판을 snapshot에 고정한다. 모델 제작사, provider 문자열, 모델 이름으로 계약 제품을 추측하지 않는다. 제품에 속한 모든 관측은 구성원·팀·세션 식별자 유무와 무관하게 포함한다.

대상 제품 집합은 ① 계산 기간의 지원 도구 관측이 있는 제품, ② 기간과 겹치는 확인된 구매 계약이 있는 지원 제품, ③ 기간 내 구매 여부나 계약 이력이 아직 확인되지 않은 조직 등록 지원 제품의 합집합이다. 같은 제품은 한 번만 넣는다. 현재 보관·만료·미등록 표시만으로 ①·②를 제외하지 않는다.

지원 제품에 사용이 없더라도 기간 내 유효 구매 계약액은 분모에 포함한다. 지원 제품의 계약 누락은 대상에서 제외하는 사유가 아니다. 필요한 이력 전체를 확인할 수 없으면 조직 비율은 null이다. 확인된 다른 제품의 사용 환산액과 계약 구간은 근거에 남긴다.

관측 제품 열거·등록/구매 이력 대상 열거·카탈로그 매핑을 모두 확정하면 `scopeStatus=complete`다. 이번 조회가 실패해도 **같은 snapshot에 고정된 독립 근거**가 전체 대상 집합을 확인한다면 complete를 유지하고 가능한 반대편 조직 금액도 보존한다. 다른 snapshot이나 현재 설정을 대체 근거로 섞지 않는다.

필요한 대상 열거를 조회하지 못하고 독립 근거도 없어 집합을 확정할 수 없으면 `scopeStatus=unavailable`, `scopeReasons=["scope_incomplete"]`다. 이 경우 조직의 분자·분모·행 수·비율은 모두 null이며 조회된 제품의 금액·행 수·계약 조각만 `scopeProducts`에 보존한다. 사용 조회 실패 시 미등록 Codex 관측을 발견하지 못했을 가능성이 있다면 확인된 Claude 계약액만 조직 전체의 complete 분모로 제공하지 않는다. 대상 집합이 고정된 뒤 특정 집계만 실패한 경우와 구분한다.

관측을 지원하지 않는 제품은 분자·분모 양쪽에서 제외하고 `excludedProducts`에 `observation_not_supported`를 남긴다. 제품을 매핑할 수 없는 관측은 `tool_mapping_missing`으로 제외한다. 이는 대상 제품의 미환산 행과 다르다. 대상 집합이 비면 `no_supported_products`이며 0%가 아니다.

## 2. 계약 이력 입력과 시간 의미

현재 계약 JSON의 `effectiveFrom`·`effectiveTo`는 계약 기간이다. 현재 버전 기록에는 좌석·금액이 실제로 바뀐 효력일, 실제 변경과 입력 정정의 구분, 과거 확인 근거가 충분히 없다. `recorded_at`은 저장 시각이며 효력일이 아니다.

후속 모델은 제품별 **불변 timeline revision**과 그 revision 안의 효력 구간을 보존한다. 물리 테이블명이나 migration 번호는 이 명세에서 예약하지 않는다.

| revision 필드 | 의미·제약 |
|---|---|
| `revisionId`, `organizationId`, `productKind`, `vendorId` | 조직·계약 제품의 불변 식별자. 제품 전체의 계약을 한 timeline으로 정규화 |
| `previousRevisionId`, `sequence` | 직전 판과 단조 증가 판. 낙관적 동시성 검사에 사용 |
| `changeType` | `initial`, `contract_change`, `correction`, `historical_confirmation` |
| `changeEffectiveDate` | 실제 변경의 KST 효력일. `contract_change`에 필수, 저장 날짜로 자동 채우지 않음 |
| `recordedAt`, `recordedBy` | 지식이 입력된 시각·관리자. 계약 효력과 독립 |
| `reason`, `evidenceRefs` | 변경·정정 이유와 확인 근거의 비밀 아닌 내부 참조. 과거 확인에 필수 |
| `segments` | 해당 판의 전체 효력 구간. 이전 판을 수정하거나 지우지 않음 |

| segment 필드 | 의미·제약 |
|---|---|
| `segmentId` | 판 안에서 불변인 근거 식별자 |
| `effectiveFromDate`, `effectiveUntilDateExclusive` | KST 날짜의 `[시작, 끝)` 구간. 끝 null은 종료일 미정 |
| `kind` | `purchased` 또는 근거로 확인된 `verified_no_purchase` |
| `planId`, `tiers` | 구매 구간의 요금제·좌석·좌석별 월 USD 문자열. 구매 좌석 전체 포함 |
| `monthlySeatFeeUsd` | tiers에서 검증한 월 총액. 저장된 총액과 다시 계산한 합이 다르면 입력 거부 |
| `confirmedAt`, `confirmedBy`, `evidenceRefs` | 구간의 사실 확인과 근거 |
| `legacyVersionRefs` | 참조한 기존 계약 판. 이것만으로 효력을 확정하지 않음 |

`verified_no_purchase`는 해당 구간에 구매 계약이 없었다는 명시 확인이며 금액은 정확히 0이다. 무료 구매 계약은 `purchased`와 0달러 tiers로 남긴다. 두 경우 모두 이력 공백과 다르다. 아무 기록이 없는 구간을 자동으로 미구매·무료로 바꾸지 않는다.

실제 계약 변경은 효력일부터 새 좌석·금액을 적용하도록 이전 구간을 나누어 새 revision을 만든다. 입력 오류 정정은 원래 효력 구간의 잘못된 값을 새 revision에서 바로잡으며 정정 입력일을 새 효력일로 삼지 않는다. 과거 확인은 확인한 구간과 근거를 명시하며, 확인되지 않은 나머지 구간을 연장 추정하지 않는다.

같은 판의 겹치는 구간, 동일 제품의 중복 활성 timeline, 음수 금액·좌석, 잘못된 날짜는 저장을 거부한다. 구간 공백을 보존할 수는 있으나 그 공백을 포함한 비율은 계산하지 않는다. 인접한 `[A,B)`, `[B,C)`는 겹치지 않는다. 실제로 동시에 존재하는 동일 제품의 여러 구매 계약은 하나의 구간 안에서 모든 구매분을 합산하여 확인하고, 중복 버전을 동시에 더하지 않는다.

기존 계약의 `effectiveTo`는 **포함 날짜**다. 명시적으로 확인하여 이관할 때만 다음 날 자정을 새 exclusive 끝으로 변환한다. 기존 버전별 `recorded_at`을 구간 시작으로 이관하지 않는다. 현재 보관 상태·현재 만료 여부는 과거의 확인된 구매 구간을 삭제하지 않는다.

후속 명령은 인증된 조직 관리자만 append할 수 있고 `expectedRevisionId` 불일치는 409로 재조회한다. 감사 기록과 증빙 원문 접근은 조직 권한을 적용하며 API에는 비밀 없는 근거 ID만 반환한다. 기존 계약 수정 경로가 새 timeline을 우회할 수 없도록 전환한다. 효력·정정 정보 없이 기존 경로로 금액·좌석·기간·계약 삭제가 기록되면 같은 트랜잭션에서 timeline을 `needs_confirmation`으로 표시하고 새 snapshot의 분모를 제한한다. 이미 발행된 snapshot은 바꾸지 않는다.

## 3. 기간·snapshot·일할 배분

요청은 기존 KST 날짜 범위 1~366일을 사용한다. `requestedPeriod`는 `[startDate의 00:00, endDate 다음 날 00:00)`이다. 실제 계산 구간은 `[requestedStart, min(requestedEnd, asOf))`이며 `asOf <= requestedStart`이면 `effectivePeriod=null`, `temporalState=upcoming`이다.

`asOf`는 같은 개요 snapshot의 기준 시각이다. 사용 관측은 `source_time >= effectiveStart AND source_time < effectiveEnd`를 적용한다. 정확히 asOf에 발생한 행은 다음 snapshot에 들어간다. 현재 날짜 계약액도 정확히 같은 asOf까지만 배분한다. `asOf < requestedEnd`이면 `in_progress`, 아니면 `elapsed`다. 달력상 진행 중이라는 이유만으로 계산 가능한 값의 `status`를 partial로 만들지 않는다.

snapshot에는 조직·요청/실효 기간·asOf, 사용 관측 입력, 카탈로그 매핑 판, 선택한 계약 revision·구간, 공통 관측 정책의 판·근거를 함께 고정한다. 계약 원천의 한 읽기 트랜잭션에서 선택한 revision ID와 내용을 snapshot에 복사한다. 조회 도중 계약 정정이 들어와도 기존 snapshot/상세/CSV에는 새 판을 섞지 않는다. 새 snapshot만 정정된 판을 반영한다.

`source_time < asOf`는 발생 시각 상한이며 수신 시각을 이용한 과거 DB 복원이 아니다. 늦게 적재된 관측은 새 snapshot에만 반영된다. 카드의 현재값에 이전 snapshot의 분자나 분모를 끼워 넣지 않는다. 카드 활성화에는 day/auto 양쪽에 같은 불변 근거를 전달하는 후속 배선이 필요하며, PROJ-318에서 기존 day API에 snapshot 필드를 추가하지 않는다.

각 제품 구간을 **KST 월 경계·계약 효력 경계·실효 기간 끝**에서 자른다. KST 하루는 86,400초다. 한 조각이 속한 월의 실제 일수로 나눈다. 2월은 해당 연도의 28일 또는 29일을 사용한다.

```text
조각 계약액 = 월 좌석 계약 총액
              × 조각의 경과 나노초
              ÷ (해당 월의 실제 일수 × 86,400 × 10^9)

기간 계약액 = 모든 대상 제품·조각의 정확한 계약액 합
비율(%)    = 확인된 사용 환산액 ÷ 정확한 기간 계약액 × 100
```

진행일의 경과 나노초에는 snapshot asOf의 소수 초를 보존한다. 11월 월 300달러인 계약의 자정~정오는 300 × 0.5 / 30 = 5달러다. 분모만 당일 전체 또는 월 전체로 늘리지 않는다.

### 정밀도와 0 판정

입력 USD 문자열을 정수 계수와 `10^scale`의 분수로 변환한다. 모든 배분·합산·비율은 임의 정밀도 정수의 기약분수로 계산하며, 중간 나눗셈과 조각별 합산에서 반올림하지 않는다. `1/31`처럼 끝나지 않는 소수도 분수로 보존한다.

금액 API 대표값은 최종 합을 소수 12자리 `HALF_EVEN`으로 직렬화한 USD 문자열이다. 전체·제품·조각은 각자의 정확한 합에서 독립적으로 직렬화한다. **직렬화한 조각을 다시 더해 제품/전체 금액을 만들거나, 직렬화한 분모로 비율을 계산하지 않는다.** 상세의 `exactAmounts`와 조각 `exactAllocatedUsd`는 기약분수의 정수 문자열 `{numerator, denominator}`이며 단위는 USD다. 분모는 항상 양수다.

계약액이 0인지 여부는 정확한 분수의 numerator로만 판정한다. 월액 0.000000000001달러를 31일 중 하루에 배분한 값은 `1/31000000000000`달러다. 대표값이 `0.000000000000`이어도 유료·양수이며 `zero_denominator`가 아니다. 사용 환산액이 0.000000000001달러면 비율은 **3100.0%**다.

backend는 정확한 금액으로 `amountMagnitude`를 `zero | positive_below_cent | at_least_cent`로 판정한다. null 금액은 null이다. frontend는 `positive_below_cent`를 **‘$0.01 미만 (양수)’**로 표시하고, 무료 계약·0달러로 설명하지 않는다. 상세에는 USD 대표값과 정확 분수, 월액·시간 비율을 함께 보여준다. 금액이 정확히 0일 때만 `$0.00`을 0의 의미로 사용한다.

비율은 정확한 금액에서 계산한 뒤 소수 한 자리 `HALF_EVEN`을 적용한다. JSON `number`의 한 자리 표시를 보존하기 위해 최종 반올림 비율의 절댓값이 `2^49 = 562,949,953,421,312` **미만**일 때만 number를 반환한다. 이 범위의 binary64 반올림 오차는 0.05 미만이다. 경계 이상이면 `ratioPercent.value=null`, `ratio_out_of_range`로 제공하고 정확 비율 분수는 상세에 보존한다. 비율의 10배가 안전한 정수인지 여부만으로 허용하지 않는다. 값을 잘라 상한에 맞추거나 Infinity를 보내지 않는다. 표시 금액 두 자리 반올림도 정확 분수에서 직접 하며, 12자리 대표값을 다시 반올림해 이중 반올림하지 않는다. API의 `displayAmountsUsd`는 이 최종 두 자리 문자열을 제공한다.

## 4. 관측 입력·상태

이 카드의 공통 관측 adapter는 snapshot에 고정된 PROJ-315 정책 입력을 받는다. 필요한 입력은 `observed_usage`(실제 행 존재), `observed_empty`(해당 제품·실효 기간을 정상 조회하고 정책상 빈 관측으로 표현 가능), `no_evidence`, `query_failed`와 정책 판·근거다. 이 이름은 카드 adapter의 제안 어휘이며 PROJ-315 API가 이미 제공한다는 뜻이 아니다.

빈 SQL 결과만으로 `observed_empty`를 만들지 않는다. 현재 heartbeat·현재 설치 상태·최신 수신 시각으로 과거 사용 없음이나 수집 완전성을 독자 추정하지 않는다. 정책 연동이 pending이면 빈 범위는 `observation_policy_pending`으로 제한한다. 실제 행이 있으면 보고 공백 때문에 확인된 금액을 숨기지 않으며 수집 annotation을 별도로 함께 제공한다. 168시간 손실값을 이 카드 기간에 나누어 배분하지 않는다.

대상 행 수는 `pricedUsageRows + unpricedUsageRows`다. 값이 0인 priced 행도 환산 완료 행이다. 세션 ID·인원·토큰의 결손을 금액 계산 사유로 전파하지 않는다. 부분 미환산의 분자는 priced 행의 저장 금액 합이다. 전부 미환산이면 분자 null이며 부분합 0을 만들어 넣지 않는다.

### 복수 제품의 상태 합성

scope가 확정되어도 대상 제품 중 하나가 `no_evidence`, `query_failed`, 또는 빈 범위의 정책 `pending`이면 **조직 분자·행 수·비율은 null**이다. 성공한 제품의 값과 행 수는 scopeProducts에 보존한다. 모든 대상 계약이 확인됐으면 조직 분모는 complete로 유지할 수 있다. 예를 들어 A 사용 25달러·B no_evidence, 양쪽 계약액 10달러씩이면 조직 분자/비율 null, 조직 분모 20달러이며 A의 사용 25달러는 제품 근거에 남는다. B를 0으로 더하거나 A만의 비율을 조직 비율로 표시하지 않는다.

모든 제품이 `observed_usage | observed_empty`로 판정되면 전체 priced/unpriced 행 수를 합한다. 전체 priced 행이 양수이고 unpriced 행도 있으면 priced 금액의 합을 partial 분자로 사용한다. 한 제품이 전부 미환산이어도 다른 제품에 priced 행이 있으면 이 규칙을 따른다. 전체 priced=0·unpriced>0일 때만 전부 미환산으로 분자 null이다. 모든 제품이 observed_empty이면 0이며, 구매 없는 제품이라고 관측 판정을 생략하지 않는다.

계약 쪽은 모든 대상의 전체 실효 기간이 확인됐을 때만 조직 분모가 있다. 한 제품이라도 미확인 구간이 있으면 조직 분모·비율은 null이며 관측이 정상인 조직 분자는 유지한다. 이는 미환산 부분합을 허용하는 분자 정책과 다르다. scope 자체가 미확정이면 §1에 따라 양쪽 조직 합계를 모두 제한한다.

| 입력·상황 | 분자 | 분모 | 비율 `Value`와 문구 |
|---|---|---|---|
| 전부 환산, 계약 이력 확인 | complete 금액 | complete 양수 | complete 비율 |
| 일부 미환산 | partial 확인 금액, `pricing_missing` | complete 양수 | partial 비율, ‘확인된 금액 기준’과 미환산 건수 |
| priced 행의 금액 합 0, unpriced 행도 존재 | partial 0 | complete 양수 | partial 0%, ‘확인된 금액 기준’. **관측된 AI 사용 없음 문구 금지** |
| 모든 사용 행 미환산 | unavailable null, `pricing_missing` | 확인 가능한 금액 유지 | unavailable null, ‘사용 환산액을 확인할 수 없습니다’ |
| 계약 미입력·이력 공백·효력 불명·중복·재확인 필요 | 확인된 금액 유지 | unavailable null | unavailable null, ‘같은 기간의 계약액을 확인할 수 없습니다’ |
| 정확한 계약액 합 0 | 확인된 금액 유지 | complete 0 | unavailable null, `zero_denominator`, ‘계약액이 0이므로 비율을 계산하지 않습니다’ |
| `observed_empty`, 모든 대상 계약 확인·양수 | complete 0, 행 수 0 | complete 양수 | complete 0%, ‘관측된 AI 사용 없음’ |
| 관측 근거 없음·정책 연동 대기·사용 조회 실패 | unavailable null | 확인 가능한 금액 유지 | unavailable null. 0% 대체 금지 |
| 계약 조회 실패 | 확인된 금액 유지 | unavailable null | unavailable null, `contract_query_failed` |
| 전체 미래 | unavailable null | unavailable null | unavailable null, `upcoming`, ‘조회 기간이 아직 시작되지 않았습니다’ |

‘관측된 AI 사용 없음’은 관측값 설명이며 실제 비사용을 완전히 증명했다는 뜻이 아니다. 사용 행이 있으나 환산액이 정확히 0인 경우에는 ‘관측된 사용의 환산액 $0.00’으로 표현한다. 금액 0과 사용 행 0을 혼동하지 않는다.

`complete`는 정의된 관측·계약 금액으로 계산 가능하다는 의미다. 실제 전체 AI 사용·구독 청구의 완전성을 뜻하지 않는다. 보고 공백 등의 공통 annotation과 `in_progress`는 값의 상태와 독립적으로 표시한다.

카드 내부 제안 사유는 `pricing_missing`, `contract_missing`, `contract_history_gap`, `contract_effectivity_unknown`, `contract_history_overlap`, `contract_needs_confirmation`, `zero_denominator`, `no_observation_evidence`, `observation_policy_pending`, `usage_query_failed`, `contract_query_failed`, `scope_incomplete`, `no_supported_products`, `upcoming`, `ratio_out_of_range`다. 복수 문제가 있으면 모든 해당 사유를 중복 없이 이 목록 순서로 반환한다. scope가 미확정이면 조직 Value에 scope_incomplete와 확인된 원천 실패 사유를 함께 남긴다. 이 어휘를 기존 API 사유 목록에 즉시 추가하지 않는다.

## 5. 후속 API 안

후속 개요 응답에 독립 영역 `subscriptionUsageValue`를 추가한다. 기존 `Value<T>`의 `complete | partial | unavailable`과 null 규칙을 사용한다. 금액은 USD 문자열을 감싼 `Value<string>`으로 하여 계약액을 못 구해도 확인된 사용 금액을 보존한다. 별도 비교 필드를 만들지 않는다.

```ts
type ExactFraction = { numerator: string; denominator: string };
type AmountMagnitude = "zero" | "positive_below_cent" | "at_least_cent";
type Period = { start: string; endExclusive: string };
type SubscriptionUsageValue = {
  ratioPercent: Value<number>;
  observedEquivalentCostUsd: Value<string>;
  allocatedContractCostUsd: Value<string>;
  pricedUsageRows: number | null;
  unpricedUsageRows: number | null;
  requestedPeriod: Period;
  effectivePeriod: Period | null;
  asOf: string;
  temporalState: "elapsed" | "in_progress" | "upcoming";
  allocationMethod: "calendar_month_daily";
  scopeStatus: "complete" | "unavailable";
  scopeReasons: string[];
  exactAmounts: {
    observedEquivalentCostUsd: ExactFraction | null;
    allocatedContractCostUsd: ExactFraction | null;
    ratioPercent: ExactFraction | null;
  };
  amountMagnitude: {
    observedEquivalentCostUsd: AmountMagnitude | null;
    allocatedContractCostUsd: AmountMagnitude | null;
  };
  displayAmountsUsd: {
    observedEquivalentCostUsd: string | null; // 정확 분수에서 최종 2자리 반올림
    allocatedContractCostUsd: string | null;
  };
  basis: {
    snapshotId: string;
    calculationVersion: "subscription-usage-value-v1";
    productMappingVersion: string;
    observationPolicyVersion: string;
    observationEvidenceRefs: string[];
  };
  scopeProducts: ScopeProduct[];
  excludedProducts: {
    productKind: string | null;
    toolKind: string | null;
    reason: "observation_not_supported" | "tool_mapping_missing";
  }[];
  sourceErrors: { source: "usage" | "contracts"; code: string; retryable: boolean }[];
};
type ScopeProduct = {
  productKind: string;
  toolKinds: string[];
  vendorId: string | null;
  observedEquivalentCostUsd: Value<string>;
  allocatedContractCostUsd: Value<string>;
  pricedUsageRows: number | null;
  unpricedUsageRows: number | null;
  timelineRevisionId: string | null;
  historyStatus: "verified" | "incomplete" | "needs_confirmation";
  reasons: string[];
  allocations: {
    segmentId: string;
    kind: "purchased" | "verified_no_purchase";
    effectiveFromDate: string;
    effectiveUntilDateExclusive: string | null;
    period: Period;
    month: string; // YYYY-MM
    daysInMonth: number;
    elapsedNanoseconds: string;
    seats: number;
    monthlySeatFeeUsd: string;
    allocatedUsd: string;
    exactAllocatedUsd: ExactFraction;
    legacyVersionRefs: string[];
    evidenceRefs: string[];
  }[];
};
```

null의 사용량 카운터는 조회·근거가 없어 판정하지 못한 경우다. 확인된 빈 관측은 두 카운터가 0이다. 전체 미래는 카운터 null, scopeStatus=unavailable·scopeReasons=[upcoming]이며 아직 계산할 대상 집합을 조회하지 않아도 된다. 일부 제품의 계약 이력이 불완전하면 전체 `allocatedContractCostUsd`는 null이며, 확인된 조각만 총액처럼 표시하지 않는다. 해당 제품의 알려진 조각은 `allocations`에 남기고 빠진 구간 사유를 함께 제공한다.

관측 원천과 계약 원천의 실패는 독립적으로 처리한다. scope가 확정되어 있으면 가능한 반대쪽 조직 금액을 보존한다. scope도 확정하지 못하면 조직 금액은 null이며 확인된 제품별 근거를 보존한다. sourceErrors에는 실패 원천만 넣는다. 인증·인가·공통 snapshot 생성 실패는 기존 HTTP 오류다. 카드의 계산 불가나 원천별 실패는 독립 영역의 상태이며 기존 summary/trend/session 필드를 null로 바꾸지 않는다. 원문 SQL·호스트·증빙 내용·예외 메시지는 반환하지 않는다.

frontend는 비율·배분액을 다시 계산하지 않고 `ratioPercent`, `displayAmountsUsd`, `amountMagnitude`를 사용한다. 상세 근거는 같은 snapshot의 `scopeProducts`와 exact 분수를 표시한다. 비율이 null이어도 확인된 분자·분모와 모든 사유를 숨기지 않는다. scope 미확정으로 조직 금액이 모두 null이어도 **‘확인된 제품별 근거 보기’**를 제공하여 남은 제품별 값과 계약 조각을 열람할 수 있게 한다. 이 부분합을 조직 총액으로 표시하지 않는다. export를 제공하는 후속 구현은 같은 snapshot의 원값·상태·사유·정확 분수를 사용한다.

### API 값 예시 — 11월 하루

```json
{
  "ratioPercent": { "value": 250.0, "status": "complete", "reasons": [] },
  "observedEquivalentCostUsd": { "value": "25.000000000000", "status": "complete", "reasons": [] },
  "allocatedContractCostUsd": { "value": "10.000000000000", "status": "complete", "reasons": [] },
  "pricedUsageRows": 2,
  "unpricedUsageRows": 0,
  "requestedPeriod": { "start": "2026-10-31T15:00:00Z", "endExclusive": "2026-11-01T15:00:00Z" },
  "effectivePeriod": { "start": "2026-10-31T15:00:00Z", "endExclusive": "2026-11-01T15:00:00Z" },
  "asOf": "2026-11-01T15:00:00Z",
  "temporalState": "elapsed",
  "allocationMethod": "calendar_month_daily",
  "scopeStatus": "complete",
  "scopeReasons": [],
  "exactAmounts": {
    "observedEquivalentCostUsd": { "numerator": "25", "denominator": "1" },
    "allocatedContractCostUsd": { "numerator": "10", "denominator": "1" },
    "ratioPercent": { "numerator": "250", "denominator": "1" }
  },
  "amountMagnitude": { "observedEquivalentCostUsd": "at_least_cent", "allocatedContractCostUsd": "at_least_cent" },
  "displayAmountsUsd": { "observedEquivalentCostUsd": "25.00", "allocatedContractCostUsd": "10.00" }
}
```

위 JSON은 계산 필드만 발췌했다. 완전 응답에는 `basis`, 같은 제품의 `scopeProducts`와 `excludedProducts`, `sourceErrors`도 필수다. 신규 필드나 endpoint가 현재 구현됐다는 예시가 아니다.

## 6. 계산·상태 인수 예제

별도 표기가 없으면 계약 근거·관측 조회가 정상이며 분자와 분모의 제품 집합은 같다. 날짜는 양 끝을 포함하는 요청 날짜다.

| 사례 | 입력·정확한 배분 | 결과 |
|---|---|---|
| 기본 | 기간 계약액 100, 사용 250 | complete 250.0% |
| 11월 하루 | 월 300, 11/01: 300 × 1/30 = 10, 사용 25 | complete 250.0% |
| 월중 실제 변경 | 11/01~15 월 300, 11/16~30 월 600: 150 + 300 = 450, 사용 900 | complete 200.0%; 11/16 00:00부터 새 금액 |
| 윤년·월 경계 | 2028/02/28~03/01, 2월 월 290·3월 월 310: 290 × 2/29 + 310 × 1/31 = 30, 사용 75 | complete 250.0% |
| 평년 2월 | 2027/02/28 하루, 월 280: 280/28 = 10, 사용 25 | complete 250.0% |
| 현재일 정오 | 11/01, asOf=11/01 12:00 KST, 월 300: 300 × 0.5/30 = 5, 같은 상한 이전 사용 12.5 | complete 250.0%, in_progress·‘집계 중’ |
| 정확히 시작 | asOf=11/01 00:00 KST, 요청 11/01 | upcoming, effectivePeriod·금액·비율 null |
| 월초의 asOf | 요청 10/31~11/01, asOf=11/01 00:00 KST, 10월 월 310 | 10월 하루 10만 포함; 11월 배분 0초 |
| 부분 미환산 | priced=2, unpriced=3, 확인 금액 25, 계약액 10 | partial 250.0%, ‘확인된 금액 기준 · 미환산 3건’ |
| 부분 미환산의 확인 0 | priced=1(금액 0), unpriced=1, 계약액 10 | partial 0.0%; 사용 없음 문구 금지 |
| 전부 미환산 | priced=0, unpriced=3, 계약액 10 | 분자·비율 null, pricing_missing; 계약액 10 유지 |
| 미구매 제품의 사용 | 명시 verified_no_purchase가 기간 전체를 덮고 관측 사용 존재 | 계약액 0, 비율 null/zero_denominator |
| 무료 구매 계약 | purchased 월액 0, 사용 25 | 계약액 0, 비율 null/zero_denominator |
| 구매했으나 사용 없음 | claude 계약 10·사용 25, codex 계약 10·정상 관측 행 0 | 분모 20, complete 125.0%; codex 제외 금지 |
| 정상 빈 관측 | observed_empty, 구매 계약액 10, 행 0 | complete 0.0%, ‘관측된 AI 사용 없음’ |
| 근거 없음 | no_evidence, 구매 계약액 10 | 분자·비율 null/no_observation_evidence |
| 제품별 관측 근거 차이 | scope 확정, A 사용 25·계약 10, B no_evidence·계약 10 | 조직 분자·행 수·비율 null, 분모 20; A의 값은 scopeProducts에 보존 |
| 제품별 미환산 | A priced 1건·사용 25, B unpriced 1건, 각 계약 10 | 조직 partial 사용 25·비율 125.0%, priced=1/unpriced=1 |
| 대상 열거 실패 | usage 전체 조회 실패, 확인된 Claude 계약액 10, 미등록 Codex 사용 존재 여부 미확정 | scope unavailable, 조직 분자·분모·행 수·비율 null; Claude 계약 근거만 보존 |
| 계약 누락 | claude 사용 25·계약 10, codex 사용 5·계약 미입력 | 확인 분자 30 유지, 전체 분모·비율 null/contract_missing |
| 이력 공백 | 11/01~30 요청, 확인 계약은 11/16부터이며 이전 구매 여부 미확인 | 전체 분모·비율 null/contract_history_gap; 15일로 기간 축소 금지 |
| 정정·보관 | 월액 300을 600으로 입력 정정하고 현재 제품 보관 | 새 snapshot은 과거 확인 구간 600 적용; 기존 snapshot 300 유지, 과거 제외 금지 |
| 미지원 제품 | 지원 제품 계약 10·사용 25, 관측 미지원 제품 계약 100 | 양쪽 대상에서 미지원 제품 제외, 250.0% 및 제외 사유 |
| 작은 양수 | 31일 월의 월액 0.000000000001, 하루 배분 1/31000000000000, 사용 0.000000000001 | complete 3100.0%; 분모 대표값은 0.000000000000, 표시 ‘$0.01 미만 (양수)’ |

쿼리 실패는 `usage_query_failed` 또는 `contract_query_failed`와 반대편 확인 금액 유지까지 검증한다. 정확히 구간 끝/asOf에 있는 사용의 배제, leap year, 월말·윤년 종료일의 exclusive 변환, 시간의 소수 초, 반올림 전 합산, 안전한 number 범위를 넘는 비율도 후속 구현의 경계 테스트다.

## 7. 문구·텍스트 시안

```text
구독료 대비 사용 환산액
250.0%
관측된 사용 환산액 $250.00
기간 배분 계약액   $100.00
달력 월 기준 일할 배분 · 대상 제품 및 계산 근거 보기
```

```text
구독료 대비 사용 환산액                           집계 중
250.0%  확인된 금액 기준 · 미환산 3건
관측된 사용 환산액 $12.50
기간 배분 계약액   $5.00
11월 1일 12:00 KST까지 · 대상 제품 및 계산 근거 보기
```

```text
구독료 대비 사용 환산액
— 같은 기간의 계약액을 확인할 수 없습니다
관측된 사용 환산액 $30.00
기간 배분 계약액   확인 불가
Codex 계약 정보 미입력 · 계약 근거 확인
```

```text
대상 제품 및 계산 근거
기간: 11월 1일 00:00~24:00 KST · 기준 시각: 11월 2일 00:00 KST
Claude Code → Claude Team
월 구매 계약액 $300.00 · 구매 좌석 전체 · 30일 중 1일 = $10.00
관측된 사용 환산액 $25.00 · 환산 2건 · 미환산 0건
계약 revision / 효력 구간 / 확인 근거 ID / 정확 분수
제외 제품: 관측 지원 없음 (해당하는 경우)
```

공통 설명은 다음과 같다.

> 지원 도구에서 관측된 전체 사용을 같은 제품·기간의 좌석 계약액과 비교합니다. 구독 포함분과 별도 API 과금을 구분하지 않으며, 웹 사용 등 미관측 사용은 포함되지 않습니다. 실제 청구액·절감률·생산성 지표가 아닙니다.

비율은 소수 한 자리, 일반 금액은 소수 두 자리다. 작은 양수는 별도 문구를 쓴다. 100% 초과를 성공·절감으로 표현하는 색상이나 진행 막대를 쓰지 않는다. 수집 annotation은 값의 설명과 별도로 제공하며 보고 공백을 곧바로 사용 손실액으로 바꾸지 않는다.

## 8. 후속 구현 인계·미해결

인수 순서는 효력 이력 명령과 검증 → 과거 구간의 근거 있는 확인 → 공통 관측 정책 adapter → 불변 snapshot·정확 계산 → API와 카드다. legacy 기록의 효력 추정 백필, 현재 active 월액을 과거 분모에 사용하는 임시 구현, 미확인 구간의 0 채우기는 허용하지 않는다.

후속 티켓에서 물리 스키마·명령 endpoint·권한·보존 기간과 PROJ-315 adapter의 실제 인터페이스를 구현하고 양쪽 계약 테스트를 추가해야 한다. 기존 계약 UI의 이력 명령 전환과 우회 쓰기 시 needs_confirmation 처리를 함께 인수한다. 운영의 과거 계약 확인은 관리자의 근거 입력이 필요하며 이 문서의 계산 예제가 그 사실 확인을 대신하지 않는다.

PROJ-318 완료는 이 설계와 세션 탭 정리·기존 계약 회귀 검증까지다. 카드 출시·이력 migration·기존 계약 데이터의 사실 확인·운영 배포는 별도 완료 조건이다.
