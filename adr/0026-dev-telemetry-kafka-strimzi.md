# 0026. dev 텔레메트리 비동기 수집은 Strimzi Kafka와 mTLS를 사용한다

## Status

Accepted — PROJ-282의 사용자 승인 결정. 실제 배포·전환·장애 검증 완료를 뜻하지 않는다.
ADR 0001의 dev OTLP 불투명 토큰 전용 결정, ADR 0005의 단일 배포 단위·네트워크 큐 금지,
ADR 0006의 dev 동기 적재 성공 응답·서버 배치 큐 금지 부분을 대체한다. 기존 HTTP 복구 경로와
마스킹·신원·재시도 계약의 나머지 부분은 유지한다.

## Context

샌드박스 제공 측이 MSK 사용 불가를 확인했다. MSK 클러스터는 생성됐지만 조직 정책이
연결을 거절해 Gateway → Kafka → ingest 검증이 완료되지 않았다. 이 결정은
`infra`의 배포·인증서·ACL과 `pulsemetry-backend`의 Kafka 연결·수집 성공 의미를 함께 바꾼다.

## Decision

- dev EKS의 단일 AZ에서 Strimzi 1.3.0, Kafka 4.2.2 KRaft dual-role Pod 3개를 서로 다른 전용 ARM64 노드에 둔다.
  각 broker는 암호화 gp3 100Gi PVC를 사용한다. 노드 1대 장애에 대응하며 AZ 장애 내성은 제공하지 않는다.
- 경로는 JWT 검증 Gateway → Kafka → 자격 재검증 ingest → 마스킹·S3·ClickHouse다.
  Gateway의 200은 Kafka ACK이며 최종 저장 완료가 아니다. 기존 envelope와 at-least-once 처리를 유지한다.
- topic partition 6, RF 3, min ISR 2, 보존 72시간, auto-create 및 unclean leader election 비활성화를 유지한다.
  HTTP 10MiB, producer 12MiB, broker/topic/replica·consumer fetch 13MiB다.
- MSK IAM 대신 내부 9093 mTLS와 Kafka ACL을 쓴다. stage별 Gateway 쓰기, ingest 읽기/group 처리,
  별도 offset 관리 자격을 분리한다. ingest와 offset-admin은 지정 topic·group의 Read/Describe를 공유한다.
  정상 commit과 관리용 offset reset이 모두 Group Read + Topic Read를 사용하는 `OffsetCommit` 요청이므로,
  별도 offset-admin Secret·수동 Job은 자격 공급과 운영 절차의 분리다. KafkaTopic만 토픽 설정을 소유한다.
  offset-admin의 Topic Read는 FETCH도 허용하므로 같은 stage의 지정 topic에 보존되는 마스킹 전 envelope를 읽을 수 있다.
  이 기존 능력을 제한된 수동 offset 운영 담당자의 권한 범위로 수용하며 일반 앱에는 그 Secret을 mount하지 않는다.
- Strimzi가 같은 `pulsemetry-dev` namespace에서 CA·사용자 Secret을 생성·관리한다.
  이 Secret만 앱에 읽기 전용 mount하며 기존 AWS Secret의 CSI 공급은 바꾸지 않는다.
- 인증서 365일·갱신 창 30일을 사용한다. 앱은 Secret metadata revision을 배포 annotation에 반영해
  명시적으로 재배포한다. 기존 replica 1/Recreate의 짧은 중단을 허용하며 자동 무중단 reload를 주장하지 않는다.
- HTTP 경로를 보존한 내부 검증 → main Kafka 전환 → 30분 안정화 → MSK와 전용 리소스 순차 삭제를 따른다.
  사용자는 미확인 MSK 데이터 폐기를 허용했다. 데이터·offset 이관은 하지 않는다.
- prod는 이 결정의 배포 대상이 아니며 PROJ-257에서 용량·다중 AZ·controller 분리를 따로 결정한다.

## Alternatives Considered

MSK 정책 해제를 계속 기다리는 방법은 제공자 확인과 충돌한다. 무인증 Kafka는 Gateway만 발행할 수 있다는
envelope 신뢰 경계를 없앤다. 단일 broker는 RF 3/min ISR 2 계약을 충족하지 못한다.
3 AZ와 전용 controller는 이번 dev 비용·가용성 선택에 포함하지 않는다.

## Consequences/Tradeoffs

### Positive

AWS MSK 데이터 권한에 의존하지 않고 기존 비동기 계약을 검증한다. 토픽·사용자를 선언적으로 관리한다.

### Negative

Kafka·스토리지·인증서·업그레이드의 운영 책임이 팀으로 이동한다. 단일 AZ 장애와 앱 인증서 재배포 중단을 감수한다.
마스킹 전 envelope가 Kafka에 최대 72시간 저장되므로 접근 제한·디스크 암호화·보존 검증이 필수다.
ingest에 정상 commit 권한을 주면서 같은 group의 offset reset만 ACL로 막는 경계는 제공하지 않는다.
운영자의 reset은 앱과 분리된 자격·수동 Job으로 수행하고 대상 확인, group 중지, 변경 전후 offset 대조를 요구한다.
offset-admin은 72시간 보존 정책의 마스킹 전 원문에 접근할 수 있다. 원문 조회를 별도 ACL로 금지한다고 주장하지 않으며,
운영 절차에서는 필요한 메타데이터만 남기고 envelope 본문을 콘솔·로그·파일로 덤프하지 않는다.

## Acceptance Criteria

실제 JWT 요청의 ACK→consume→commit→ClickHouse, 역할별 ACL 거부, 인증서 갱신,
단일 broker 장애·두 broker 장애 시 거짓 성공 방지, PVC/offset 보존과 HTTP 회귀를 검증한다.
실제 결과는 PROJ-285에 기록하고 코드·계약 승인과 구분한다.

## References

- [Kafka 수집 배포 계약](../contracts/telemetry-kafka.md)
- [PROJ-282](https://team376.atlassian.net/browse/PROJ-282)
- [Strimzi 호환표](https://strimzi.io/downloads/)
- [Kafka Group·Topic 권한 매핑](https://kafka.apache.org/42/security/authorization-and-acls/#operations-and-resources-on-protocols)
- [Kafka 4.2.2 offset 변경 구현](https://github.com/apache/kafka/blob/4.2.2/clients/src/main/java/org/apache/kafka/clients/admin/internals/AlterConsumerGroupOffsetsHandler.java)
