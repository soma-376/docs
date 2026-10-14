# 0022. 사용자 JWT 키는 별도 Secrets Manager Secret의 버전을 파일로 공급한다

## Status

Accepted (ADR 0021의 사용자 JWT 공개키 전달 계약을 보충한다. 실제 키 생성·Secret 등록·CDK/Helm 공급 구현·
AWS 배포·live 인증 검증 완료를 뜻하지 않는다.)

## Context

`pulsemetry-backend`의 enrollment-api는 RS256 사용자 access token(AT)을 발급하고 검증한다.
dashboard-api는 같은 AT의 서명과 현재 RDS 세션 상태를 검증한다. `UserJwt`는 RSA 2048bit 이상,
PKCS8 개인키와 X.509 공개키를 파일에서 읽고, 발급 개인키와 `active-kid` 공개키의 RSA modulus가 같은지 확인한다.
AT 유효기간은 5분이며 검증의 시간 허용 오차는 30초다.

기존 계약은 파일 경로와 정확한 `kid` 맵을 정의하지만 배포 키 원본·최초 생성·버전 보존·공급 주체는 정하지 않았다.
앱 재배포마다 키를 생성하거나 ECS/EKS가 다른 원본을 쓰면 기존 AT를 검증할 수 없다.
개인키와 공개키를 하나의 Secret으로 묶으면 공개키만 필요한 dashboard에도 개인키 읽기 권한이 전파된다.
이 결정의 당사자는 `pulsemetry-backend`와 `infra`이며 frontend의 AT 소비 계약도 유지한다.

## Decision

### 1. 원본과 형식

- **환경별 고정 ARN 두 개**를 유지한다. 개인키 Secret과 공개키 Secret은 분리하고 교체는 같은 Secret의
  새 버전으로 수행한다. 환경 간 키를 공유하지 않으며 **dev ECS와 dev EKS는 같은 두 원본**을 소비한다.
- 두 Secret은 AWS 관리형 KMS key **`aws/secretsmanager`**로 암호화한다.
  현재 같은 계정의 공급 경계를 전제로 하며 자동 회전은 설정하지 않는다.
- 개인키 Secret의 `SecretString`은 **RSA 2048bit 이상 PKCS8 PEM**이다.
  공개키 Secret의 `SecretString`은 **`kid → X.509 PEM` JSON 키링**이다. 각 값은 인증서가 아니라
  `PUBLIC KEY` 형식의 RSA 공개키다. `kid`는 비어 있지 않은 고유 식별자이며 새 키쌍에는 새 `kid`를 부여한다.
- RS256·issuer·audience·현재 세션 조회 계약은 유지한다. 원격 `jku`/`x5u`나 키 자동 탐색을 도입하지 않는다.

### 2. 명시적 최초 초기화와 부분 실패

1. 명시적인 초기화 작업에서 최초 키쌍과 `kid`를 한 번 생성한다. 앱 시작·일반 재배포·Secret의 누락을
   이유로 키를 자동 생성하지 않는다.
2. 같은 키쌍의 개인키·공개키를 두 Secret에 등록한다. 각 등록의 ARN·`VersionId`와 작업 상태를 제한된 배포 기록에 남기되
   키 원문을 코드·문서·CI 출력·로그·앱 이미지·일반 환경변수에 남기지 않는다.
3. 두 등록이 끝나면 선택한 버전의 형식·RSA 크기·`kid` 대응과 시험 서명/검증으로 키쌍 일치를 확인한다.
   모두 성공한 뒤 배포 참조를 확정한다. 두 Secret 갱신은 하나의 원자적 트랜잭션이라고 가정하지 않는다.
4. 한쪽 등록만 성공했으면 배포를 중단하고 **동일한 키쌍과 `kid`**로 미완료 등록을 복구한다.
   완료한 쪽을 재생성한 다른 키로 덮어쓰지 않는다. 원래 쌍을 복구할 수 없으면 해당 초기화를 폐기하고
   미배포 상태에서 별도 초기화를 시작한다.

### 3. 소비자와 파일 인터페이스

| 소비자 | 허용 원본 | 앱에 전달하는 설정 |
|---|---|---|
| enrollment-api | 개인키·공개키 Secret | `private-key-file`, `active-kid`, 정확한 `public-key-files` 맵 |
| dashboard-api | 공개키 Secret만 | 정확한 `public-key-files` 맵; 개인키·`active-kid` 설정 없음 |
| telemetry-ingest | JWT Secret 접근 없음 | 사용자 JWT 키 설정 없음 |

infra는 선택한 버전의 개인키 PEM과 키링의 각 공개키 PEM을 **별도 파일**로 공급한다.
앱 설정은 `pulsemetry.user-auth`의 파일 경로·`kid` 맵이며 PEM 원문을 넣지 않는다.
Secret의 JSON 키링 자체를 공개키 파일 하나로 전달하지 않는다. `PULSEMETRY_USER_AUTH_PUBLIC_KEY_FILE`은
`public-key-files.local`의 경로만 바꾸므로 임의 `kid`의 배포에는 YAML 또는 `SPRING_APPLICATION_JSON`의
명시적 맵을 쓴다. `kid`를 파일명으로 사용할 때는 안전한 파일명을 매핑하고 경로 이탈을 허용하지 않는다.

backend는 형식·파일 로딩·서명/검증·기동 검증을 소유한다. infra는 Secret 원본의 초기 등록·보존과
IAM·runtime 파일 공급·배포 버전 선택을 소유한다. ECS/EKS 전달 구현은 infra ADR 0030과 런북이 정한다.
로컬 Compose의 개발 키는 로컬 실행용이며 배포 원본으로 승격하지 않는다.

### 4. 배포 단위와 보존

- 배포 기록에는 환경·개인키 ARN/`VersionId`·공개키 ARN/`VersionId`·enrollment `active-kid`·
  양쪽 앱의 공개키 파일 맵을 **하나의 배포 단위**로 남긴다. dashboard가 개인키 참조를 소비한다는 뜻은 아니다.
  두 `VersionId`는 서로 독립적이며 같은 문자열일 필요가 없다.
  실제 full ARN은 계정 ID를 포함하므로 접근이 제한된 배포 입력·기록에서 관리한다.
  저장소 문서·예시·공개 검증 출력에는 placeholder를 쓰며 실제 계정 ID와 키 원문을 기록하지 않는다.
- ARN은 원본 식별자, `VersionId`는 배포 값 선택자다. mutable `AWSCURRENT`만으로 배포를 재현하지 않는다.
  ECS/EKS의 같은 단계는 같은 버전 선택과 같은 허용 `kid` 집합을 사용한다.
- 선택한 활성 버전과 승인된 롤백에 필요한 개인키·공개키 버전에는 staging label을 붙여 보존한다.
  `VersionId`를 기록하는 것만으로는 보존되지 않는다. AWS는 label 없는 버전을 폐기 대상으로 취급하므로
  label 이동·제거는 활성 배포와 롤백 기록을 확인한 뒤 수행한다.
  두 Secret의 `AWSPREVIOUS`가 자동으로 맞는 롤백 쌍이 된다고 가정하지 않는다.
- Secret 자체도 앱·검증 클러스터의 폐기와 분리해 보존한다. 고정 ARN을 삭제·재생성하면 원본 계약이 바뀐다.
  원본 수명과 label 정리 책임은 infra 운영 절차에 둔다.

### 5. 키 교체와 롤백

1. 새 키쌍·새 `kid`를 등록·검증하고 **구·신 공개키를 함께 가진 키링**의 버전을 준비한다.
2. 모든 enrollment·dashboard 검증 인스턴스에 겹치는 공개키 맵을 적용한다. dev ECS/EKS에서 남아 있는
   구 replica도 포함해 새 `kid` 검증을 확인한다. 일부라도 미적용이면 signer 전환을 시작하지 않는다.
3. 모든 enrollment signer를 새 개인키 `VersionId`와 새 `active-kid`로 전환한다. 공개키는 겹치는 맵을 유지한다.
   새 AT의 양쪽 앱 검증과 현재 세션 판정을 확인하고 마지막 구 키 발급 시점을 기록한다.
4. **마지막 구 키 발급 후 330초 이상** 기다린다. `UserJwt.lifetime`의 300초와 검증 허용 오차 30초를 합친 값이다.
   전환 시작 시각으로 대신하지 않으며 구 signer가 다시 발급했거나 마지막 발급 시점을 입증할 수 없으면
   구 공개키를 제거하지 않는다.
5. 구 공개키를 뺀 키링 버전을 모든 verifier에 배포한다. 구 개인키·겹치는 키링의 롤백 버전 보존과
   현재 verifier의 허용 키 제거는 별개다. 자동 reload 없이 명시적 재배포로 각 단계를 적용한다.

부분 실패 시 겹치는 공개키 상태를 유지하고 전환을 멈춘다. 구 signer로 롤백할 때도 그 공개키가
**모든 verifier에 먼저** 있어야 한다. 이미 구 공개키를 제거했다면 겹치는 키링을 먼저 복구한 뒤
구 개인키·`active-kid`로 전환한다. 신 키로 발급된 AT도 마지막 신 키 발급 후 330초 이상 검증할 수 있게
신 공개키를 유지한다. private 버전만 되돌리거나 신 AT가 남은 상태에서 구 공개키만 남긴 맵으로 되돌리지 않는다.

## Constraints

- 사용자 인증을 켠 배포는 선택한 파일과 `kid` 맵을 완전하게 준비한다. 누락·빈 파일·읽기 불가·잘못된 PEM·
  RSA 크기 미달·활성 키 불일치에서 임의 키 생성이나 인증 비활성화로 우회하지 않는다.
- 공개키만 필요한 dashboard 역할에 개인키 Secret 권한을 주지 않는다. 기존 token hash HMAC key와
  enrollment 관리자 token은 이 두 JWT Secret과 별도이며 telemetry-ingest의 기존 인증 계약은 유지한다.
- Secrets Manager의 staging label 수에는 한도가 있으므로 활성·승인된 롤백 집합을 관리한다.
  자동 회전·자동 reload가 필요해지면 버전 선택·전환 순서와 backend 로딩 방식을 새 결정으로 검토한다.

## Alternatives Considered

### A. Parameter Store SecureString과 기본 KMS

- 장점: 키 값을 보관하고 암호화할 수 있다.
- 단점: 기존 Secret 공급과 별도로 IAM·버전 선택·파일 전달·보존 운영 경로를 관리해야 한다.
- 탈락 이유: 이 범위에서 **운영 경로 추가 부담 대비 비용 절감이 작다**는 판단으로 Secrets Manager에 통일한다.
  가격 수치나 환경 전체의 비용 우위를 확정하는 결정은 아니다.

### B. 한 Secret에 개인키와 공개키 저장

- 장점: 키쌍 등록과 버전 선택이 단순해진다.
- 단점: 공개키 소비자에 개인키를 읽을 권한을 열거나 별도 추출 권한 경로가 필요하다.
- 탈락 이유: dashboard의 공개키 전용 권한을 유지한다.

### C. 시작 시 생성하거나 공개키를 앱 이미지에 포함

- 장점: 외부 파일 공급이 줄어든다.
- 단점: 재배포로 키가 바뀌거나 공개키 교체에 이미지 재빌드가 필요해 동일 artifact와 별도 키 버전을 묶게 된다.
- 탈락 이유: 명시적 키 수명과 ECS/EKS 같은 원본·버전 계약을 유지한다.

## Consequences/Tradeoffs

### Positive

- 앱 재배포가 키 수명을 바꾸지 않으며 ECS/EKS가 같은 AT를 검증한다.
- dashboard에 개인키 권한을 전달하지 않고 이미지와 키 버전을 독립적으로 관리한다.
- 겹치는 공개키·명시적 배포 기록과 label 보존으로 교체·롤백 조건을 검토할 수 있다.

### Negative

- 두 Secret 등록과 버전 선택에는 원자성이 없어 부분 실패 복구와 키쌍 검증이 필요하다.
- 파일 공급·권한·기동 검증과 전체 verifier/signer 상태 확인이 배포 관문으로 추가된다.
- 수동 교체는 여러 배포와 마지막 발급 시점 확인을 요구하며 자동 회전·reload의 편의를 얻지 못한다.
- label과 고정 ARN을 보존하는 운영 책임 및 Secrets Manager 사용 비용을 감수한다.

## Follow-up

- infra: ADR 0030에 따라 최초 등록 절차·ECS writer/volume·EKS mount/IAM과 명시적 버전 선택을 구현한다.
- backend: 배포 키 파일·활성 `kid`·키링의 실패 조건을 검증하고 현재 파일 로더와 공급 구현의 책임을 연결한다.
- **현재 상태**: 파일 기반 RS256 로더는 존재한다. 이 ADR 작업은 결정 채택이며 실제 키 생성·Secret 등록·
  버전 고정 공급 코드·AWS 배포·live 교체/롤백 검증은 수행하지 않았다.

## Acceptance Criteria

- 두 고정 ARN의 선택 버전·형식·`kid` 대응과 시험 서명/검증이 모두 성공하기 전 배포 참조를 확정하지 않는다.
  한쪽 등록 실패의 재시도는 같은 키쌍을 사용하며 새 키쌍을 섞는 시나리오를 차단한다.
  현행 `UserJwt`의 modulus 일치 검사를 시험 서명/검증 완료 증거로 취급하지 않는다.
- 선택한 키링 항목의 누락·빈 파일·읽기 불가·잘못된 PEM·RSA 2048bit 미달·`active-kid` 부재·키쌍 불일치가
  공급 또는 인증 활성화 기동을 실패시킨다. 현행 빈 경로 필터링이 이 관문을 대신한다고 가정하지 않는다.
- enrollment는 두 Secret, dashboard는 공개키만 읽고 ingest는 JWT Secret에 접근할 수 없다.
  실제 앱 UID에서 파일 접근을 확인하며 키 원문은 로그·출력·이미지·일반 설정에 나타나지 않는다.
- 공개키 겹침→전체 signer 전환→마지막 구 발급 후 330초→구 공개키 제거와, 각 단계의 부분 실패·
  제거 후 롤백을 검증한다. 구·신 AT 검증과 현재 세션 폐기 판정을 별도로 확인한다.
- 활성/롤백 label과 배포 기록을 대조하고, 같은 dev 단계의 ECS/EKS가 동일 원본·버전을 소비함을 확인한다.
  문서·template·readiness 검증을 live 키 공급·인증 증거로 기록하지 않는다.

## References

- [ADR 0021 - dashboard runtime·배포 경계](0021-dashboard-runtime-and-deployment-contract.md)
- [dashboard API 계약 - 인증키와 현재 세션](../contracts/dashboard-api.md#인증키와-현재-세션)
- [레포 책임](../architecture/repos.md), [시스템 아키텍처](../architecture/overview.md)
- backend `UserJwt.privateKey()`, `UserJwt.publicKey()`, `UserJwt.lifetime`, `UserJwt.verify()`
- backend `UserAuthConfig.userJwt()`, dashboard `SecurityConfig.dashboardAuthenticator()`, `UserAccessVerifier.verify()`
- infra ADR 0030(Secrets Manager 파일 공급), backend `docs/user-auth-operations.md`
- [AWS - Secrets Manager Secret와 버전 보존](https://docs.aws.amazon.com/secretsmanager/latest/userguide/whats-in-a-secret.html)
- [AWS - Secrets Manager 암호화와 aws/secretsmanager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/security-encryption.html)
