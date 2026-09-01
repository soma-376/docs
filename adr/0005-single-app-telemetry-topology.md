# 0005. 텔레메트리 파이프라인 전 계층을 단일 Spring 애플리케이션으로 띄운다

## Status

Accepted — [ADR 0001](0001-otlp-authentication-model.md)의 신원 헤더 전파 결정(4종 부여 후 collector가
전파)을 대체한다. 같은 ADR의 나머지 결정(불투명 토큰, 앱 계층 검증, ALB 인증 미채택)은 유효하다.

## Context

걸리는 레포는 셋이다 — `pulsemetry-backend`(도착지), `infra`(태스크 구성), `ai-telemetry-pipeline`
(이관 원본). [ADR 0004](0004-telemetry-pipeline-repo-merge.md)가 병합을 정한 뒤 남은 것은 그 안의
토폴로지, 곧 **인증을 어디에 두고 OTel Collector를 존치할 것인가**다.

**인증 계층이 RDB를 읽어야 한다.** OTLP 경로의 `ptt_`는 불투명 토큰이라 클레임이 없다. 검증은 해시로
`telemetry_tokens`를 찾아 `installations`·`members`·`tenants`까지 조인해야 성립하고, 거부 사유 아홉 가지가
전부 그 조인의 컬럼에서 나온다([`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §4,
[ADR 0001](0001-otlp-authentication-model.md)). 인증을 ALB·Cognito로 빼면 그 테이블을 읽기 위한 데이터
모델이 인증 쪽에 따로 생긴다 — ADR 0004가 없애려던 "읽기 때문에 양쪽에 엔티티가 생긴다"를 인증에서
그대로 재생산한다. 따라서 인증은 데이터 모델과 같은 애플리케이션에 있어야 한다.

**Spring Security는 Collector에 심을 수 없다.** Collector는 Go 단일 바이너리이고 공식 이미지로
뜬다. 공식 이미지를 유지하면 인증 결과를 Collector 너머로 실어 나르는 장치가 필요하다 —
`include_metadata`·`headers_setter`·`batch.metadata_keys` 세 요소의 결합이다. 한 요소만 빠져도
신원이 조용히 소실된다. 실제로 터진 것이
[`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §5 B4다.

**순서를 뒤집는 것도 성립하지 않는다.** Collector를 앞에 두면 인증 전 요청이 먼저 처리된다.
Collector 단계에는 마스킹 후 원본을 외부 저장소에 적재하는 책임이 있으므로, 폐기된 토큰이나 정지된
tenant의 요청처럼 **거부될 데이터가 이미 영속화된 뒤** 거부된다. 인증을 통과하지 못한 데이터는 애초에
파이프라인에 들어오지 않아야 한다.

결정하지 않으면 병합 작업이 계층 배치를 정하지 못한 채 시작되고, 인증·마스킹·적재의 책임 경계가
구현 순서로 정해진다.

## Decision

- **파이프라인 전 계층을 단일 Spring 애플리케이션으로 띄운다.** 인증 · 수집/마스킹 · 변환 · 보강 ·
  적재가 한 배포 단위 안의 모듈이며, 그 사이에 네트워크 홉을 두지 않는다.
- **OTel Collector 바이너리를 쓰지 않는다.** 수집과 마스킹을 Kotlin으로 이식한다 — OTLP HTTP 수신,
  `redaction` 상당의 마스킹, 마스킹 이후 원본의 외부 저장소 적재.
- **인증은 Spring Security 필터 체인이고, 신원은 `SecurityContextHolder`에서 얻는다.**
  신원 헤더 4종 부여와 Collector 전파 3요소를 폐기한다.
- **클라이언트 마스킹을 신뢰해 서버 마스킹을 생략하지 않는다.** 데몬의 1차 제거와 무관하게 서버에서
  다시 마스킹한다.
- **infra는 collector 컨테이너를 띄우지 않는다.** `config/otel-collector.yaml`과 그 주입 경로
  (infra ADR-0017)가 함께 사라진다.

## Alternatives Considered

- **공식 이미지 유지 — Spring Security → Collector → 변환·보강** — 이식 비용이 없다. 그러나 신원 전파
  3요소 결합이 남고, Collector 설정 파일이 실행 주체(infra)와 소유 주체(backend)로 갈린 상태도 남는다.
  ADR 0004가 병합을 채택한 근거가 "규율로 정합성을 유지하는 데 실패했다"인데, 같은 성격의 규율을
  둘이나 남긴다. 기각.
- **Collector 선행 — Collector → Spring Security + 변환·보강** — 홉이 하나 줄고 프록시가 필요 없다.
  그러나 위 Context의 적재 문제로 인증이 걸러야 할 데이터가 먼저 영속화된다. 기각.
- **커스텀 Collector 배포판 — 인증 확장을 직접 구현해 빌드한다** — 인증이 리시버 앞에 서므로 적재
  문제가 없고, 신원을 배치 전에 데이터로 새기면 전파 3요소도 사라진다. 상위 리시버와 재시도도 그대로
  쓴다. 갈리는 지점은 확장이 무엇을 읽느냐다. RDB를 직접 조회하면 위 첫 논거가 그대로 적용되고,
  [ADR 0004](0004-telemetry-pipeline-repo-merge.md)가 없앤 `TOKEN_HASH_SECRET` 공유가 되살아난다.
  인증 API를 호출하면 OTLP 요청마다 같은 앱으로 동기 왕복이 하나 더 생기고 — 그 앱이 곧이어 같은 본문을
  다시 받는다 — 거부 사유 아홉 가지가 프로세스를 건너는 계약으로 남는다. auth-proxy가 이름만 바뀐
  것이다. 어느 쪽이든 Go 배포판을 상위 버전에 맞춰 유지하는 비용이 Kotlin 앱과 따로 남는다. 기각.
- **ALB·Cognito 인증** — `ptt_` 검증이 RDB 조인이라 ALB가 대신할 수 없다.
  [ADR 0001](0001-otlp-authentication-model.md)이 이미 같은 결론으로 기각했다.

## Consequences/Tradeoffs

### Positive

- 데이터 모델·설정·인증이 한 레포·한 언어로 모인다. 정합성을 규율이 아니라 컴파일과 CI가 강제한다.
- B4를 일으킨 신원 전파 3요소가 소멸한다.
- 컨테이너가 하나로 줄어 네트워크 홉과 태스크 비용이 함께 준다.
- 이식할 분량이 설정 전체가 아니다. `headers_setter`·`batch.metadata_keys`·`otlphttp` exporter는
  홉을 잇는 접착제라 홉과 함께 사라진다.

### Negative

- OTLP 수신의 엣지 케이스(protobuf·JSON 인코딩, gzip, PartialSuccess)와 이후 프로토콜 변화를 직접
  떠안는다. contrib의 상위 유지보수를 잃는다.
- **재시도와 백프레셔를 직접 구현해야 한다.** 현재 ClickHouse 오류를 4xx까지 503으로 돌려보내는
  편향은 Collector가 재시도한다는 전제 위에 있는데, 그 전제가 사라진다.
- 나중에 contrib 프로세서가 더 필요해지면 그것도 직접 구현해야 한다.
- 이식 분량이 ADR 0004가 든 재작성량에 더해진다.

## Follow-up

- **롤백 트리거를 둔다.** 셋 중 하나가 관측되면 Collector 존치를 재검토한다 — ① 이식본이 특성화
  테스트가 고정한 현행 동작을 재현하지 못한다 ② OTLP 수신·재시도·백프레셔에서 E2E 회귀가 나온다
  ③ contrib 프로세서가 추가로 필요해진다. 판단 기한은 **infra가 collector 컨테이너를 내리기 직전**이다.
  그전까지 구 파이프라인이 계속 서비스하므로 되돌리는 비용은 이식분 폐기뿐이고, 아래 infra ADR 넷을
  정리한 뒤로는 달라진다. 돌아갈 자리는 Alternatives 첫 항목(공식 이미지 존치)이다.
- 재시도·백프레셔 정책을 정하고 503 계약을 다시 쓴다. 기준선은 특성화 테스트가 고정한 현행 동작이다.
- 마스킹 이후 원본의 적재 대상을 정한다 — infra ADR-0017은 이미 `awss3`를 선호한다.
- infra ADR-0007(공식 이미지라 ECR 제외)·0017(config 주입)·0022(dev config 미포크)·0023(dev
  auth-proxy)을 정리한다. 네 결정 모두 collector 컨테이너의 존재를 전제한다.
- dev compose에서 collector 서비스를 제거한다.
- 데몬 → 서버 구간이 계속 불투명 `ptt_`를 쓸지, 사용자 AT로 바뀔지는 backend ADR-0007 Follow-up의
  미결이다. AT로 바뀌면 manifest revision 대조가 이 경로에도 들어오므로 인증 계층의 책임이 늘어난다.
  이 결정의 계층 순서는 어느 쪽이든 그대로다.
- [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §3·§4의 검증 주체와 신원
  전파 서술은 전환이 실제로 끝난 시점에 고친다. 그전까지 현행 서술이 사실이다.

## References

- [ADR 0001](0001-otlp-authentication-model.md)(인증 모델, 부분 대체됨) ·
  [ADR 0004](0004-telemetry-pipeline-repo-merge.md)(병합 결정)
- backend ADR-0007(인증 계층) · ADR-0008(모듈 경계) · ADR-0010(파이프라인 단계 모듈)
- infra ADR-0007 · 0017 · 0022 · 0023
