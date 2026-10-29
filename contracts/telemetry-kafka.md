# 계약 — dev Kafka 비동기 수집과 mTLS 배포

당사자는 `pulsemetry-backend`와 `infra`다. 상태는 **변경 중 — dev 배포·전환 검증 전**이다.
[ADR 0026](../adr/0026-dev-telemetry-kafka-strimzi.md)이 dev 비동기 경로에 적용되며
기존 `telemetry-ingest.md`의 동기 HTTP 경로는 복구 경로로 유지한다.

## 데이터와 성공 의미

Gateway는 telemetry JWT를 검증하고 기존 Protobuf envelope와 `tenant_id:installation_id` key를 발행한다.
JWT 원문·Authorization/Bearer 값은 envelope, Kafka record header, producer metadata, 로그에 넣지 않는다.
HTTP 200은 `acks=all`의 broker ACK다.

telemetry JWT의 발급자는 enrollment-api이며 `iss=pulsemetry-enrollment`,
단일 `aud=pulsemetry-telemetry-ingest`, `scope=telemetry:write`, `typ=pulsemetry-telemetry+jwt`를 고정한다.
서명은 RS256과 최소 2048-bit RSA 키를 사용하며 사용자 로그인 JWT와 키를 분리한다.
개인키는 발급자에만 공급하고 Gateway와 HTTP 호환 ingest에는 운영자가 지정한 `kid → 공개키 파일` 맵을 공급한다.
Gateway는 알려진 `kid`의 공개키로 서명을 검증하며 토큰 안의 키·원격 키 URL은 신뢰하지 않는다.
최대 수명은 3시간, 시계 오차 허용은 30초이며 본문 수신 완료 시점에도 유효성을 확인한다.

`jti`는 RDB `telemetry_tokens.id`인 자격 UUID, `tenant_id`는 tenant UUID,
`sub`는 installation UUID다. Gateway는 이 세 값과 `iss`, `kid`, `iat`, `exp`를 envelope의
auth context에 담고 접수 시각은 envelope의 `accepted_at`으로 전달한다.
ingest는 매 처리 시도마다 `jti`로 RDB를 조회하여 JWT 형식,
tenant·installation·서명 키·발급/만료 시각의 일치와 고정 issuer를 재검증한다.
유효 시각은 consumer의 현재 시각이 아닌 Gateway 접수 시각으로 판단한다.
정상 rotation은 폐기 전 접수분만 허용하며 보안 폐기, 설치·회원·조직 차단은 거부한다.
처리 코어에 전달할 신원은 이 검증을 통과한 RDB 행에서 얻는다.

ingest는 RDB 자격을 재검증한 뒤 기존 마스킹·신원·보강·보존 경계·적재 코어를 실행한다.
auto commit은 끄며 기존 영구 오류 commit, 일시 오류 1·5·30초 재시도와 at-least-once 처리를 유지한다.
자격 거부는 영구 실패이고 RDB 조회 장애는 일시 실패다. dev JWT 경로는 기존 ptt 전용 계약을 대체하지만
HTTP 복구 경로의 ptt 호환은 유지한다. HTTP 경로에서 JWT를 받을 때에도 서명 검증과 RDB 재검증을 수행한다.

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
ingest는 지정 topic과 group의 Read/Describe를 갖는다. 별도 `offset-admin` 사용자도
같은 stage의 지정 topic과 group에 Read/Describe를 갖는다. topic 설정은 KafkaTopic/Topic Operator가 소유한다.
앱 ServiceAccount의 AWS JWT·DB·S3 권한은 Kafka ACL과 독립적으로 유지한다.

Kafka 4.2.2의 정상 consumer commit과 `Admin.alterConsumerGroupOffsets`를 통한 offset reset은
모두 `OffsetCommit` 요청을 사용하며 Group Read와 Topic Read를 요구한다. 따라서 ingest의 정상 commit을
허용하면서 같은 group의 offset reset만 Kafka ACL로 거부할 수 있다고 보장하지 않는다.
근거는 [Kafka 권한 매핑](https://kafka.apache.org/42/security/authorization-and-acls/#operations-and-resources-on-protocols)과
[4.2.2 offset 변경 구현](https://github.com/apache/kafka/blob/4.2.2/clients/src/main/java/org/apache/kafka/clients/admin/internals/AlterConsumerGroupOffsetsHandler.java)이다.

운영자가 실행하는 offset reset은 별도 `offset-admin` 자격과 수동 관리 Job으로 제한한다.
이는 자격 공급과 실행 절차의 분리다. 앱에는 `offset-admin` Secret을 공급하지 않으며,
수동 Job은 대상 stage·topic·group과 변경할 offset을 확인하고 group 중지·변경 전후 offset 대조 절차를 따른다.
ACL 검증은 지정 외 topic/group 접근, Gateway 읽기와 ingest 쓰기의 거부를 확인한다.
ingest의 같은 group reset 거부를 별도 ACL 보장이나 검증 완료 조건으로 삼지 않는다.

CA·사용자 인증서는 유효기간 365일, 갱신 창 30일이다. Secret UID/resourceVersion으로 만든
`kafka.tlsRevision` annotation을 명시적 Helm 배포에 반영한다. 자동 갱신만으로 외부 앱이
새 인증서를 읽었다고 판정하지 않는다. 현재 replica 1/Recreate의 중단 가능성을 인계한다.

## 전환과 재전달

`PULSEMETRY_INGEST_MODE=dual`은 HTTP 수신기와 Kafka consumer를 동시에 켜는 전환 모드다.
같은 요청을 두 경로로 복제 발행하는 기능은 없다. ALB를 Gateway로 전환하는 동안 기존 HTTP 요청과
Kafka 처리가 겹칠 수 있으므로 두 경로 사이의 처리 순서는 보장하지 않는다.
Kafka 안에서는 `tenant_id:installation_id` key에 따른 partition 순서를 사용한다.

재전송·재전달은 at-least-once이므로 중복을 허용한다. Kafka envelope의 `message_id`는 재시도에도
같은 receipt ID로 유지되고 HTTP 재전송은 새 receipt ID를 만든다. 분석 행의 중복 수렴 기준은
요청 ID가 아닌 아래 분석 테이블의 복합 정렬 키와 `row_version`이며 ClickHouse 조회에는 `FINAL`을 적용한다.
이 규칙은 전송 간 exactly-once 또는 전역 순서를 보장하지 않는다.

| 항목 | HTTP·Kafka가 공유하는 현재 적재 규칙 |
|---|---|
| 테이블 | logs·선별된 spans는 `telemetry_events`, metric points는 `telemetry_metric_points` |
| 엔진·버전 | 두 테이블 모두 `ReplacingMergeTree(row_version)`, `row_version UInt64` |
| partition·정렬 키 | `PARTITION BY toYYYYMM(source_time)`, `ORDER BY (tenant_id, source_time, installation_id, observation_id)` |
| 컬럼 매핑 | 공통 `IngestPipeline`의 sink가 `ObservationEnvelope`의 tenant·source time·installation·observation ID를 같은 이름의 컬럼에 쓰고, `RowVersioning.rowVersion`을 `row_version`에 쓴다 |
| 버전 산식 | `(normalizer_rev << 32) \| ingest_seq`; live `ingest_seq`는 archive receipt 수신 시각의 epoch 초다. Kafka는 보존된 `accepted_at`, HTTP는 해당 요청 접수 시각을 사용한다 |

따라서 동일 관측의 복합 정렬 키가 같을 때 높은 버전의 행으로 수렴한다. 같은 Kafka record의 재시도는
접수 시각과 버전이 같고, HTTP 재전송은 새 접수 시각에 따른 버전을 가질 수 있다.
현재 두 수신 경로는 같은 처리 코어와 이 컬럼 매핑을 사용한다.
근거는 backend의 [V2 분석 DDL](https://github.com/soma-376/pulsemetry-backend/blob/develop/libs/telemetry-persistence/src/main/resources/clickhouse/V2__telemetry_analysis_tables.sql),
[공통 행 매핑](https://github.com/soma-376/pulsemetry-backend/blob/develop/libs/telemetry-persistence/src/main/kotlin/com/team376/pulsemetry/persistence/telemetry/AnalysisEnvelopeRow.kt),
[버전 산식](https://github.com/soma-376/pulsemetry-backend/blob/develop/libs/telemetry-adapter/src/main/kotlin/com/team376/pulsemetry/telemetry/adapter/observation/ObservationEnvelope.kt)이다.

기존 `enriched_events`는 별도 V1 테이블로 `ReplacingMergeTree ORDER BY event_id`이며
그 경로의 `event_id = record_id` 계약을 보존한다. 현재 분석 테이블의 `observation_id`로 이름을 바꾸거나
두 식별자를 서로 변환하지 않는다. Kafka 전환이 기존 `enriched_events` 데이터를 이관하거나 삭제하지 않는다.

HTTP→dual→Kafka 전환 실패 시 ALB와 ingest를 기존 HTTP 모드로 복구하며 Strimzi PVC와 offset을 유지한다.
사용자 승인에 따라 기존 MSK 데이터·offset 이관 없이 새 경로 안정화 후 MSK와 전용 리소스를 삭제한다.

## 현재 상태와 검증

문서·코드 변경은 실제 연결 성공 증거가 아니다. PROJ-283·284의 구현과 PROJ-285의 실제 배포·검증을
구분하며, HTTP 복구 경로를 유지한 채 아래 미해결 검증을 수행한다.

## 미해결

dev 실제 설치·수집 전환은 아직 검증되지 않았다. PROJ-285에서 validation/main의 실제 JWT 요청,
Kafka ACK→consume→commit→ClickHouse와 로그·trace 연결, 30분 안정화, 장애·ACL·크기 경계,
PVC 재연결 및 인증서 갱신 후 재접속을 확인해야 한다. HTTP·로그인·기존 ClickHouse 보존 결과는
Kafka 검증 결과와 구분해 기록한다. 이 완료 증거를 확보한 뒤 기존 MSK를 순차 삭제한다.
