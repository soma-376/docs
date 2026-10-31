# 계약 — 생성된 Helm values의 private 동기화 사본

| 항목 | 내용 |
|---|---|
| 당사자 | **`infra`** (생성·검증·공급·배포 책임) → **`infra-generated-values`** (접근 제한 동기화 사본) |
| 사본 저장소 | [`soma-376/infra-generated-values`](https://github.com/soma-376/infra-generated-values), private, 기본 브랜치 `main` |
| 생성기 | `infra/scripts/eks-values.mjs` |
| 관련 ADR | [허브 ADR 0030 — 생성된 values만 private 레포에 동기화한다](../adr/0030-generated-values-private-sync-copy.md) |
| 상태 | 변경 중 — ADR과 공급 계약 리뷰, 관련 레포 문서 반영 후 확정 |

이 계약은 생성 결과의 공급 경계와 revision 연결을 정한다. 입력·생성기·chart·배포 정책과 절차는
`infra`가 소유하며, private 레포의 값이 새 원본이 되지 않는다.

## 1. 소유권과 접근

`infra` 운영 담당자가 원본 변경, 생성 결과 검증, 사본 동기화와 배포 판단을 맡는다.
`infra-generated-values`는 생성된 values와 대응 infra revision만 보관한다. 상세 계약은 이 문서에
두고, 사본 레포의 README와 AGENTS는 목적·허용 파일·수정 경로·담당 주체를 안내한다.

사본은 비공개로 유지하고 운영 담당자와 필요한 읽기 주체만 접근한다. Argo CD가 사본을 소비할 때도
읽기 경계만 갖는다. 저장소 읽기 권한은 원본 수정 권한이나 배포 승인으로 해석하지 않는다.
자격증명 발급·보관·연결 설정은 이 계약의 generated 파일에 포함하지 않는다.

## 2. 공급 파일과 revision

`dev/generated/`에는 다음 12개 YAML만 공급한다. 이 목록은 현재 `eks-values.mjs`의 출력 파일명과
일치해야 하며, 생성기 출력 범위를 바꿀 때 이 계약과 사본 안내를 함께 갱신한다.

```text
base.yaml
lbc.yaml
runtime.yaml
enrollment-api.yaml
telemetry-ingest.yaml
telemetry-gateway.yaml
telemetry-kafka-admin.yaml
dashboard-api.yaml
clickhouse.yaml
fluent-bit.yaml
ascp.yaml
observability.yaml
```

`dev/infra-commit.txt`에는 생성에 사용한 infra의 전체 commit SHA 한 줄을 기록한다. 생성기·chart·
추적된 입력이 그 commit과 일치하는 상태에서 생성한다. 같은 실행의 12개 파일과 SHA를 한 사본
커밋으로 게시하며, 파일 누락이나 서로 다른 실행의 혼합이 있으면 동기화하지 않는다.

이 SHA는 생성기·chart·추적된 입력의 revision을 식별한다. AWS 조회 결과와 이미지 digest 등
외부 입력 전체를 복원하는 증거는 아니므로 SHA 하나로 완전한 재현성을 보장한다고 표현하지 않는다.

루트 `README.md`와 `AGENTS.md`는 안내 문서로 허용한다. 원본 입력, 생성기, chart, 수동 override,
Secret 원문·토큰·개인키, kubeconfig, AWS 조회 원본 JSON, 로그·검증 증거와 개인 경로는 공급 대상이
아니다. 실제 ARN·endpoint 같은 생성된 참조는 private 사본에만 두며 공개 문서나 PR 본문에 옮기지 않는다.

## 3. 생성·동기화·사용 절차

1. `infra`에서 원본 입력·생성기·chart의 변경을 검토하고 commit을 확정한다.
2. 그 revision의 생성기로 values를 만들고 기존 infra 절차에 따라 검증한다.
3. 허용한 12개 YAML과 대응 SHA만 복사한다. 실행 디렉터리 전체를 복사하지 않는다.
4. 파일 목록·누락·내용 일치와 Secret 원문 부재를 검토한 뒤 한 묶음을 한 커밋으로 동기화한다.
5. 사본을 사용할 때는 사본 자체의 commit도 고정하고 `dev/infra-commit.txt`의 SHA와 일치하는
   infra checkout의 chart·정적 values를 사용한다. SHA가 다르거나 필요한 파일이 없으면 중단한다.

상세 명령과 기존 설치·검증·복구 순서는
[infra EKS 수동 배포 런북](https://github.com/soma-376/infra/blob/develop/docs/runbooks/dev-eks-manual-deployment.md)을
따른다. Git은 파일의 `0600`을 보존하지 않으므로 checkout 뒤 로컬 사본 디렉터리는 `0700`,
values와 SHA 파일은 `0600`으로 설정한다.

사본을 고쳐 원본으로 되돌리는 역방향 동기화는 하지 않는다. 값 수정은 infra 원본 수정과 재생성을
거친다. 이전 values를 사용할 때도 해당 사본 commit과 infra SHA를 함께 선택하고 기존 복구 절차를
따른다. 서로 다른 사본 revision의 runtime·앱 YAML을 혼합하지 않는다.

## 4. 동기화와 배포의 경계

사본의 push·merge 성공은 저장소에 파일을 공급했다는 뜻이다. Argo CD 연결·sync, Kubernetes
적용, workload 정상 상태와 기능 검증을 완료했다는 뜻이 아니다. 배포 판단과 증거는 infra가 관리한다.

이 계약으로 Argo CD Application, 자동 sync, 이미지 자동 갱신이나 one-off Job 자동 실행을
도입하지 않는다. 기존 수동 설치 순서, JWT verifier → signer 순서, Kafka 관리 Job의 별도 실행
절차는 유지한다. `telemetry-kafka-admin.yaml`을 보관하는 행위가 Job 실행 요청이 되지 않는다.

## 5. 현재 상태와 미해결

PROJ-275의 범위는 허브 계약·ADR, infra 운영 안내, private 레포의 README·AGENTS와 기본 브랜치
구성이다. 최초 문서 구성에는 실제 generated values를 게시하지 않는다.

계약 확정에는 infra·사본 저장소 담당자의 리뷰와 관련 문서 반영 확인이 필요하다. 실제 values
동기화, Argo CD 인증·연결·sync 정책, 배포와 live 기능 검증은 별도 운영 작업이며 이 문서의
작성이나 저장소 생성으로 완료 처리하지 않는다.
