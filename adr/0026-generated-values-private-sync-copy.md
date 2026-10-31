# 0026. 생성된 Helm values만 private 레포에 동기화한다

## Status

Proposed — PROJ-276의 선행 사본 공급 제안. [ADR 0030](0030-dev-gitops-delivery.md)이 private 사본·infra 원본 소유·역동기화 금지를 계승하며, generated만 보관·모든 직접 수정 금지·수동 배포에 한정한 범위는 네 image.ref 예외·완료 manifest·V/P 선택으로 부분 대체·확장한다.

이 파일명과 링크는 보존한다. 같은 번호의 [Kafka ADR 0026](0026-dev-telemetry-kafka-strimzi.md)과는 별개 결정이며 번호만으로 인용하지 않는다. 아래 Decision은 최초 사본 제안의 범위이고 현행 적용 규칙은 [단일 공급 계약](../contracts/generated-values-sync.md)을 따른다. 실제 GitOps 인수 전에는 수동 절차의 적용 경계를 유지한다.

## Context

`infra`는 Helm values 생성기와 입력·chart·배포 절차를 소유한다. 생성된 values에는 Secret 원문을
넣지 않지만 계정 식별자를 포함한 ARN과 내부 endpoint가 들어갈 수 있다. Argo CD가 읽을 저장소를
준비하면서 이 결과물을 공개 원본 레포와 구분해 보관할 필요가 있다.

이 결정은 공급자인 `infra`, 사본을 보관하는 `infra-generated-values`, 계약을 소유하는 `docs`에
걸린다. 사본의 수정 경로와 대응 revision이 없으면 원본과 값이 어긋나거나 다른 chart revision과
조합될 수 있고, 파일 동기화를 배포 완료로 오인할 수 있다.

## Decision

1. `infra-generated-values`는 private 레포로 두며 생성된 Helm values의 동기화 사본만 소유한다.
   입력·생성기·chart·배포 정책과 절차의 소유권은 `infra`에 유지한다.
2. dev 사본은 `dev/generated/`에 두고 대응 infra 전체 commit SHA를 `dev/infra-commit.txt`에
   기록한다. 같은 생성 실행의 허용 파일과 SHA를 한 사본 커밋으로 공급한다.
3. 사본을 직접 편집하거나 원본으로 역동기화하지 않는다. 수정은 infra 원본 변경 → 재생성 →
   검증 → 사본 동기화 순서로 한다. 사용 시 사본 commit과 infra SHA를 함께 고정한다.
4. 파일 허용목록·접근 경계·제외 자료는 [공급 계약](../contracts/generated-values-sync.md)이
   소유한다. 사본 레포의 README·AGENTS는 계약을 참조하고 상세 규칙을 별도로 만들지 않는다.
5. 동기화는 배포 완료를 나타내지 않는다. Argo CD 연결·자동 sync·Job 실행 정책을 이 결정으로
   도입하지 않으며 기존 infra의 수동 설치·검증·복구 절차를 유지한다.

## Alternatives Considered

### A. 공개 infra 레포에 실제 generated values를 함께 보관한다

생성기와 결과를 한 곳에서 볼 수 있지만 ARN·내부 endpoint가 공개된다. 원본 구현과 접근 제한이
필요한 실행 참조를 구분하기 위해 선택하지 않는다.

### B. private 레포에서 values를 직접 수정한다

값을 즉시 바꿀 수 있지만 생성기와 별도의 원본이 생기고 다음 생성에서 변경이 사라질 수 있다.
원본 변경과 재생성으로 수정 경로를 하나로 유지한다.

### C. 입력·생성기·chart·배포 책임까지 private 레포로 옮긴다

사본 소비가 한 레포 안에서 가능해지지만 인프라 소유권을 분리하고 기존 운영 절차를 이동해야 한다.
이번 요구는 생성 결과의 접근 제한 공급이므로 이 범위 확대를 택하지 않는다.

## Consequences/Tradeoffs

### Positive

- 공개 인프라 원본을 유지하면서 실제 생성 결과의 접근을 제한할 수 있다.
- 사본 commit과 infra SHA로 values와 생성기·chart의 조합을 식별할 수 있다.
- 원본 수정 경로와 파일 공급·배포의 책임을 분명히 한다.

### Negative

- 두 레포의 revision을 맞춰야 하며 사본 갱신이 늦으면 원본과 차이가 생긴다.
- infra SHA는 외부 조회 입력까지 복원하지 않으므로 완전한 재현성을 보장하지 않는다.
- private 저장소도 Secret 원문 보관소가 아니다. 허용목록 검토와 접근 관리가 계속 필요하다.
- Git checkout은 `0600`을 보존하지 않으므로 로컬 파일 권한을 별도로 적용해야 한다.

## Follow-up

- 공급 계약과 세 레포 안내를 함께 리뷰하고, 채택 후 ADR·계약 상태를 갱신한다.
- 실제 values 공급과 Argo CD 인증·연결·sync 정책, live 배포 검증은 별도 운영 작업으로 기록한다.

## Acceptance Criteria

계약의 파일 목록이 infra 생성기 출력과 일치하고, 같은 생성 묶음과 infra SHA를 함께 공급한다는
규칙이 세 레포 문서에 일치해야 한다. 사본 직접 수정·역방향 동기화 금지와 배포 상태의 구별을
확인한다. 이 검증은 실제 values 게시나 Argo CD 배포 증거를 대신하지 않는다.

## References

- [PROJ-275 — ArgoCD 동기화용 비공개 레포지토리 구축](https://team376.atlassian.net/browse/PROJ-275)
- [생성된 values 공급 계약](../contracts/generated-values-sync.md)
- [infra `eks-values.mjs`](https://github.com/soma-376/infra/blob/0b37a008b9ce10ac31156437000d5e538ec10c24/scripts/eks-values.mjs)
- [infra EKS 수동 배포 런북](https://github.com/soma-376/infra/blob/develop/docs/runbooks/dev-eks-manual-deployment.md)
