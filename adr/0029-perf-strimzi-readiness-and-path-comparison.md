# 0029. 성능 시험은 격리된 Strimzi 환경에서 진입 경로와 저장 정합성을 분리해 검증한다

## Status

Accepted — PROJ-192의 B01–B07 수행 요청에 따른 결정이다. 코드 병합, 실제 배포, 성능 시험 통과는 각각 별도 증거로 판정한다.

## Context

`infra`의 기존 perf 구현과 `telemetry-load-tests`의 환경 v2 계약은 MSK IAM 연결을 전제로 한다.
현재 dev는 [ADR 0026](0026-dev-telemetry-kafka-strimzi.md)의 Strimzi와 mTLS를 사용한다.
기존 내부 HTTP endpoint만으로는 ALB·WAF·TLS를 포함한 사용자 진입 경로를 측정할 수 없다.
과거 소량 기능 검증과 관측 시스템의 정상 상태는 발생기 여유나 처리 용량의 근거가 아니다.

이 결정은 AWS·Kubernetes를 소유하는 `infra`, 환경 교환·발생기·결과를 소유하는
`telemetry-load-tests`, 합성 데이터와 저장 의미를 소유하는 `pulsemetry-backend`에 적용한다.
제품 OTLP payload, JWT 검증, 마스킹, 귀속 및 저장 스키마는 기존 제품 계약을 따른다.

## Decision

### 환경 교환과 격리

- 환경 계약의 원본은 `telemetry-load-tests/docs/contracts/infra-environment.md`에 둔다.
  MSK를 표현하는 기존 v2 schema·fixture·동작은 보존하고 Strimzi를 표현하는 환경 v3를 추가한다.
  문서 버전과 Kafka provider로 명시적으로 분기하며, 미지원 버전·provider 및 환경 교환 bundle의 버전 혼합을 거부한다.
  run-plan·journal·run-manifest의 독립 버전까지 일괄 변경하지 않는다.
- Strimzi 계약은 내부 9093 mTLS, Kafka CR/NodePool·노드/PVC, topic·group·partition·복제 조건,
  역할별 CA/PKCS12 Secret 참조와 metadata revision을 표현한다. 비밀 원문을 교환 파일에 넣지 않는다.
  infra의 schema 사본은 원본 도구로 export하고 출처·바이트 해시를 함께 고정한다.
- perf는 dev와 별도 환경으로 배포한다. 같은 환경의 비교 실행 동안 이미지 digest, 노드·replica,
  데이터 조건과 계측 설정을 고정한다. 단일 발생기를 SUT와 다른 노드에 둔다.
  기존 dev·prod 배포의 렌더링과 자원 수명을 perf 변경으로 바꾸지 않는다.
- Kafka CR/PVC의 보존 정책은 환경 삭제와 구분한다. 생성 자원 inventory와 환경 ID로
  정리 대상을 확인하고 Helm·Kubernetes·CloudFormation 및 잔존 EBS/Secret/로그·증거의 수명을 함께 관리한다.
  공유 dev 및 다른 실행의 자원을 이름 유사성만으로 삭제하지 않는다.

### 진입 경로와 실행 조건

- 하나의 격리 SUT에 HTTPS ALB/WAF/TLS 경로와 내부 Gateway HTTP 경로를 둔다.
  외부 경로는 유효한 hostname·인증서를 검증하며 TLS 검증을 끄지 않는다.
  두 경로 모두 기존 JWT·본문 검증을 수행하고 최종적으로 같은 Kafka·ingest·저장소를 사용한다.
- 경로별 실행 ID와 용량 C를 따로 기록한다. B는 기존 규칙인 C의 60%로 계산한다.
  두 경로의 비교 실행은 같은 절대 요청률·payload 분포를 사용하며 시작 데이터량·캐시·backlog 차이를 보존한다.
- 실행 전에 비용 상한·생성/삭제 시각, MAX_RPS, 최대 시간·요청·저장량, 디스크 하한,
  drain·최종 검증 기한을 확정한다. 미입력 값을 임의 기본값으로 채워 실행하지 않는다.
  비용 조회가 불가능하면 단가·환산율·사용량·여유분을 명시한 추정 견적을 사용하고 실제 청구액으로 표현하지 않는다.
- 공유 dev에서 enrollment를 scale-down하는 정식 prepare를 실행하지 않는다.
  결제·메일·OIDC 등 시험 경로 밖 부수 작동은 비활성으로 유지하고 PostgreSQL·ClickHouse·S3는 실제 저장 의존성으로 사용한다.

### 데이터와 저장 판정

- 배경 데이터는 합성 원본에서 perf 전용 ID로 재생성한다. dev DB·credential·기존 artifact를 복제하지 않는다.
  과거 데이터 preload와 새 k6 실행 신원을 별도 manifest로 기록하며 preload를 Gateway/Kafka/S3 성공 증거로 사용하지 않는다.
- perf seed는 환경 ID와 resolved 환경에서 유래한 대상 증거의 해시를 검증하고,
  첫 쓰기 전에 실제 PostgreSQL·ClickHouse identity를 대조한다. SSO·알림 활성화는 perf에서 허용하지 않는다.
  기존 dev/local의 충돌·크기·속도·압력·retention·재개 검증을 유지한다.
- 대표 규모·기간·tenant 편향·payload/배치 분포·캐시 조건을 명시한다.
  합성 데이터가 존재한다는 사실만으로 운영 대표성이 입증됐다고 판정하지 않는다.
- HTTP ACK, Kafka 소비/drain, ClickHouse 최종 논리 저장을 별도로 판정한다.
  drain은 정확한 topic/group과 모든 partition의 최신 offset 근거를 사용한다.
  미커밋 partition은 begin/end offset으로 비어 있음을 입증할 때만 drain 조건을 충족한다.
  누락·오래된 표본을 0으로 바꾸지 않는다. 기존 MSK probe와 Prometheus probe는 provider에 맞게 선택한다.
- 요청 journal과 ClickHouse `FINAL`의 논리 ID·귀속을 대조한다.
  아카이브 본문 표본 검증은 표본 범위로 보고하며 전수 검증으로 표현하지 않는다.

### 관측과 결과

- 정식 실행의 remote-write와 환경/run 식별자를 필수로 검증한다. 발생기의 CPU·RSS·swap·물리 NIC,
  FD·소켓·VU·journal 비용과 서버·DB 지표를 같은 시간축에 수집한다.
  CPU 약 20% 여유는 초기 기준이며 다른 자원 병목을 대신하지 않는다.
- Grafana에는 전체 요청 경과시간 p95/p99, 실패율, 계획/시작 유입·dropped, bytes/s,
  발생기·서버·Kafka·DB 지표를 둔다. 내장 `http_req_duration`과 연결·TLS를 포함한 custom 경과시간을 구분한다.
  OTLP partial rejection 검사에 필요한 응답 본문을 유지한다.
- ALB/WAF 오류, 실제 egress/route, DB client backend·예약 슬롯·각 pool과 운영 작업의 연결 예산을 기록한다.
  `pg_stat_activity` 전체 행 수를 `max_connections`로 나눈 값을 client 연결 사용률로 사용하지 않는다.
- 정상 유지 구간의 기존 기준은 전체 요청 경과시간 p95≤1초, p99≤3초, 실패율≤0.1%,
  계획 대비 시작 요청≥99%, dropped=0이다. 워밍업은 제외하고 요청 시작 phase로 귀속한다.
  발생기 포화·증거 누락은 용량 결론을 제한하며 독립적으로 확인한 저장 누락·오귀속 등 실패는 보존한다.
  PASS/FAIL/INCONCLUSIVE/NOT_RUN을 설계·실제 적용·실행 검증 단계별로 기록한다.

## Alternatives Considered

기존 v2 필드를 Strimzi 의미로 재해석하면 구 소비자가 잘못된 MSK 연결을 선택할 수 있다.
공유 dev 부하는 배경 트래픽·자동 증감과 prepare의 enrollment 축소 때문에 재현성과 서비스 상태를 함께 바꾼다.
내부 HTTP만 측정하면 외부 진입 경계의 지연·차단·대역폭을 설명할 수 없다.
단일 발생기 여유를 확인하기 전에 분산 발생기를 도입하면 ID·유입·journal 분할 과제를 동시에 추가한다.

## Consequences/Tradeoffs

### Positive

현행 Strimzi와 같은 수집 의미로 경로별 영향을 비교하고, 발생기 미달과 서버 실패를 구분할 근거를 보존한다.
기존 MSK 계약과 dev 배포를 유지하면서 새 perf 지원을 검증할 수 있다.

### Negative

격리 환경·고정 노드·실제 저장소에 비용이 들고, v2/v3 호환 검사와 Strimzi 인증서·스토리지 정리 책임이 늘어난다.
필수 관측과 journal도 자원을 사용하므로 용량 보정부터 같은 설정을 유지해야 한다.
단일 AZ·축소 사양·합성 데이터 결과를 운영 용량이나 AZ 장애 내성으로 일반화할 수 없다.

## Follow-up

- 다중 발생기(B08)는 단일 발생기 부족이 확인된 뒤 ID·요청률·journal·시계 합산을 별도로 보완한다.
- 6시간 Soak(B09)는 현재 최대 3시간 JWT 발급 방식의 수명·교체 문제를 해결한 뒤 수행한다.
- 이번 실행의 비용·기한·데이터 선택과 실제 준비도는 각 Jira 작업 및 실행 manifest에서 기록한다.

## Acceptance Criteria

v2 회귀와 v3 양쪽 계약 검증, dev/prod 렌더 회귀, 잘못된 seed 대상의 쓰기 전 거부,
빈/미커밋·누락·stale partition drain 검증을 통과한다.
그 뒤 전용 환경에서 세 시그널 소량 전송의 ACK→소비→ClickHouse를 대조하고 같은 구간의 Grafana와 발생기 여유를 확인한다.
코드 검사 통과만으로 실제 적용이나 성능 시험 완료를 선언하지 않는다.

## References

- [PROJ-192](https://team376.atlassian.net/browse/PROJ-192), 하위 PROJ-292–298
- [Kafka 수집 계약](../contracts/telemetry-kafka.md)
- [dev 역사 데이터 시드 결정](0023-dev-historical-dashboard-dataset.md)
- [성능 환경 계약 원본](https://github.com/soma-376/telemetry-load-tests/blob/main/docs/contracts/infra-environment.md)
