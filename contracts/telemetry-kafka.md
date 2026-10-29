# 계약 — dev Kafka 비동기 수집과 mTLS 배포

당사자는 `pulsemetry-backend`와 `infra`다. 상태는 **변경 중 — 구현·배포 검증 전**이다.
[ADR 0026](../adr/0026-dev-telemetry-kafka-strimzi.md)이 dev 비동기 경로에 적용되며
기존 `telemetry-ingest.md`의 동기 HTTP 경로는 복구 경로로 유지한다.

## 데이터와 성공 의미

Gateway는 telemetry JWT를 검증하고 기존 Protobuf envelope와 `tenant_id:installation_id` key를 발행한다.
JWT 원문·Authorization은 envelope와 로그에 넣지 않는다. HTTP 200은 `acks=all`의 broker ACK다.
ingest는 RDB 자격을 재검증한 뒤 기존 마스킹·신원·보강·보존 경계·적재 코어를 실행한다.
auto commit은 끄며 기존 영구 오류 commit, 일시 오류 1·5·30초 재시도와 at-least-once 처리를 유지한다.

`eks_validation`은 `pulsemetry-dev-telemetry-ingress-validation` topic 및
`pulsemetry-dev-telemetry-ingest-validation` group, `eks`는 각각 `-main`을 사용한다.
archive prefix·topic·group·ACL을 같은 stage에서 생성한다. topic은 6 partition, RF 3, min ISR 2,
72시간 delete 보존이며 자동 생성과 unclean leader election은 금지한다.
HTTP body 10MiB, producer max.request.size 12MiB, broker/topic/replica·consumer fetch 상한은 13MiB다.

## 연결 인터페이스

Strimzi cluster 이름은 `soma-376-dev-telemetry`, namespace는 `pulsemetry-dev`다.
bootstrap은 `soma-376-dev-telemetry-kafka-bootstrap.pulsemetry-dev.svc.cluster.local:9093`이다.
`PULSEMETRY_KAFKA_SECURITY=mtls`는 SSL/PKCS12와 hostname 검증을 사용하며 IAM/STS/SASL fallback을 하지 않는다.

| 환경변수 | 파일 |
|---|---|
| `PULSEMETRY_KAFKA_TLS_TRUSTSTORE_LOCATION` | `/mnt/kafka-tls/cluster/ca.p12` |
| `PULSEMETRY_KAFKA_TLS_TRUSTSTORE_PASSWORD_FILE` | `/mnt/kafka-tls/cluster/ca.password` |
| `PULSEMETRY_KAFKA_TLS_KEYSTORE_LOCATION` | `/mnt/kafka-tls/user/user.p12` |
| `PULSEMETRY_KAFKA_TLS_KEYSTORE_PASSWORD_FILE` | `/mnt/kafka-tls/user/user.password` |

Strimzi cluster CA public Secret과 사용자 Secret의 필요한 네 항목만 읽기 전용 mount한다.
CA 개인키는 앱에 주지 않는다. 비밀번호는 generated values·환경변수 원문·로그에 넣지 않는다.
설정·파일·PKCS12 검증 실패는 시작 실패이며 오류에는 비밀 원문을 포함하지 않는다.
기존 IAM/plaintext 모드는 호환용으로 보존하되 dev Strimzi는 mTLS만 사용한다.

## 권한과 수명

stage별 Gateway는 지정 topic의 Write/Describe 및 cluster IdempotentWrite,
ingest는 지정 topic Read/Describe 및 지정 group 처리만 허용한다.
topic 설정은 KafkaTopic/Topic Operator, offset reset은 별도 수동 관리 Job만 수행한다.
앱 ServiceAccount의 AWS JWT·DB·S3 권한은 Kafka ACL과 독립적으로 유지한다.

CA·사용자 인증서는 유효기간 365일, 갱신 창 30일이다. Secret UID/resourceVersion으로 만든
`kafka.tlsRevision` annotation을 명시적 Helm 배포에 반영한다. 자동 갱신만으로 외부 앱이
새 인증서를 읽었다고 판정하지 않는다. 현재 replica 1/Recreate의 중단 가능성을 인계한다.

## 현재 상태와 검증

문서·코드 변경은 실제 연결 성공 증거가 아니다. PROJ-283·284 구현 후 PROJ-285에서
실제 ACK→consume→commit→ClickHouse, ACL·인증서·크기 경계·장애·PVC 보존을 검증한다.
HTTP→dual→Kafka 전환 실패 시 기존 HTTP를 복구하며 Strimzi PVC와 offset을 유지한다.
사용자 승인에 따라 기존 MSK 데이터·offset 이관 없이 새 경로 안정화 후 MSK와 전용 리소스를 삭제한다.
