# 0005. 텔레메트리 파이프라인 전 계층을 단일 Spring 애플리케이션으로 띄운다

## Status

Accepted — [ADR 0001](0001-otlp-authentication-model.md)의 신원 헤더 전파 결정(4종 부여 후 collector가
전파)을 대체한다. 같은 ADR의 나머지 결정(불투명 토큰, 앱 계층 검증, ALB 인증 미채택)은 유효하다.

## Context

걸리는 레포는 셋이다 — `pulsemetry-backend`(도착지), `infra`(태스크 구성), `ai-telemetry-pipeline`
(이관 원본). [ADR 0004](0004-telemetry-pipeline-repo-merge.md)가 병합을 정한 뒤 남은 것은 그 안의
토폴로지, 곧 **인증을 어디에 두고 OTel Collector를 존치할 것인가**다.

**인증 계층이 RDB를 읽어야 한다.** 이 계층은 토큰 검증·재발급에 더해, AT에 실린 manifest revision을
RDB의 현재 값과 대조해 요청을 승인하거나 반려한다(backend ADR-0007). 인증을 ALB·Cognito로 빼면
사원 정보를 읽기 위한 데이터 모델이 인증 쪽에 따로 생긴다 — ADR 0004가 없애려던 "읽기 때문에 양쪽에
엔티티가 생긴다"를 인증에서 그대로 재생산한다. 따라서 인증은 데이터 모델과 같은 애플리케이션에 있어야 한다.

**그런데 Spring Security를 Collector에 심을 수 없다.** Collector는 Go 단일 바이너리이고 공식 이미지로
뜬다. 공식 이미지를 유지하면 인증 결과를 Collector 너머로 실어 나르는 장치가 필요해지는데, 그것이
`include_metadata`·`headers_setter`·`batch.metadata_keys` 세 요소의 결합이다. 이 결합은 한 요소만
빠져도 신원이 조용히 소실되며, 실제로 그렇게 터진 것이
[`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §5 B4다.

**순서를 뒤집는 것도 성립하지 않는다.** Collector를 앞에 두면 인증 전 요청이 먼저 처리된다. 그런데
Collector 단계에는 마스킹 후 원본을 외부 저장소에 적재하는 책임이 있으므로, manifest revision 대조로
**반려될 요청의 데이터가 이미 영속화된 뒤** 반려된다. 낡은 정책으로 수집된 데이터를 받지 않겠다는
대조의 목적이 무력화된다.

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
  그러나 위 Context의 적재 문제로 manifest revision 대조가 무력화된다. 기각.
- **ALB·Cognito 인증** — 인증이 manifest revision 대조를 위해 RDB를 읽어야 하므로 성립하지 않는다.
  [ADR 0001](0001-otlp-authentication-model.md)이 이미 같은 결론으로 기각했다.

## Consequences/Tradeoffs

### Positive

- 데이터 모델·설정·인증이 한 레포·한 언어로 모인다. 정합성을 규율이 아니라 컴파일과 CI가 강제한다.
- B4를 일으킨 신원 전파 3요소가 소멸한다. 지켜야 할 규칙이 줄어드는 것이 아니라 없어진다.
- 컨테이너가 하나로 줄어 네트워크 홉과 태스크 비용이 함께 준다.
- 이식할 분량이 설정 전체가 아니다. `headers_setter`·`batch.metadata_keys`·`otlphttp` exporter는
  홉을 잇는 접착제라 홉과 함께 사라진다.

### Negative

- OTLP 수신의 엣지 케이스(protobuf·JSON 인코딩, gzip, PartialSuccess)와 이후 프로토콜 변화를 직접
  떠안는다. contrib의 상위 유지보수를 잃는다.
- **재시도와 백프레셔를 직접 구현해야 한다.** 현재 ClickHouse 오류를 4xx까지 503으로 돌려보내는
  편향은 Collector가 재시도한다는 전제 위에 있다. 그 전제가 사라진다.
- 나중에 contrib 프로세서가 더 필요해지면 그것도 직접 구현해야 한다.
- 이식 분량이 ADR 0004가 든 재작성량에 더해진다.

## Follow-up

- 재시도·백프레셔 정책을 정하고 503 계약을 다시 쓴다. 현행 동작은 특성화 테스트로 고정돼 있으므로
  그것이 기준선이다.
- 마스킹 이후 원본의 적재 대상을 정한다 — 파일과 `awss3` 중 후자를 infra ADR-0017이 이미 선호한다.
- infra ADR-0007(공식 이미지라 ECR 제외)·0017(config 주입)·0022(dev config 미포크)·0023(dev
  auth-proxy)을 정리한다. 네 결정 모두 collector 컨테이너의 존재를 전제한다.
- dev compose에서 collector 서비스를 제거한다.
- [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §3·§4의 검증 주체와 신원
  전파 서술은 전환이 실제로 끝난 시점에 고친다. 그전까지 현행 서술이 사실이다.

## References

- [ADR 0001](0001-otlp-authentication-model.md)(인증 모델, 부분 대체됨) ·
  [ADR 0004](0004-telemetry-pipeline-repo-merge.md)(병합 결정)
- backend ADR-0007(인증 계층·manifest revision 대조) · ADR-0008(모듈 경계)
- infra ADR-0007 · 0017 · 0022 · 0023
