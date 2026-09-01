# 0001. OTLP 인증은 불투명 토큰을 애플리케이션 계층에서 검증한다

## Status

Accepted — infra ADR-0008(인증 이원화: 대시보드 Cognito + OTLP ALB jwt-validation)을 전면 대체한다.
이 ADR이 OTLP 인증 모델을 소유하고, infra ADR-0008은 `Superseded by`로 닫는다.
부분 대체: [ADR 0005](0005-single-app-telemetry-topology.md)가 Decision 5(신원 헤더 4종 부여 후
collector가 전파)를 대체한다 — 파이프라인이 단일 앱이 되어 전파할 홉이 없어진다. 나머지 결정
(불투명 토큰 2단 모델, 앱 계층 검증, ALB 인증 미채택)은 유효하다.

## Context

걸리는 레포는 넷이다 — `telemetryctl`(토큰 전송), `ai-telemetry-pipeline`(현행 검증자 auth-proxy),
`pulsemetry-backend`(토큰 발급, 검증의 종착지), `infra`(엣지 구성). 이 결정을 뒤집으려면 네 레포의 PR이
필요하므로 스코프 규칙상 허브 소유다.

infra ADR-0008은 Cognito JWT를 ALB가 검증하는 모델을 제안했다. 실제로 구현·운영되는 모델은 다르다 —
enrollment가 발급한 불투명 `ptt_` 토큰을 auth-proxy가 해시 조회로 검증한다. 토큰 종류(JWT ↔ 불투명),
검증 주체(ALB ↔ 앱), 검증 방식(서명 ↔ 해시 조회), 발급 주체(Cognito ↔ enrollment API) 네 축이 전부
달랐고, 세 레포 어디에도 Cognito 사용처가 없다. 이 모델을 결정으로 기록한 ADR이 어느 레포에도 없어
계약 문서([`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md))만이 실상을 담고 있었다.

결정하지 않으면 infra 문서를 읽는 사람은 Cognito 복귀를 살아 있는 계획으로 오독하고,
검증 로직의 이관처(backend)와 폐기 대상(auth-proxy)이 문서에 남지 않는다.

## Decision

1. **신원은 클라이언트 자칭이 아니라 토큰에서 파생한다.** 디바이스·사용자 식별 헤더나 신원 리소스
   속성을 신뢰하지 않는다.
2. **토큰은 2단 불투명 토큰이다** — `pit_`(설치 자격) / `ptt_`(텔레메트리 전송). OTLP `Authorization`에
   실리는 값은 `ptt_`다. 해시는 `telemetry_token`만 HMAC-SHA256(`TOKEN_HASH_SECRET`), 초대 코드와
   `installation_token`은 무염 SHA-256이다([`../contracts/enrollment-api.md`](../contracts/enrollment-api.md) §3·§4).
3. **인증 종결 지점은 애플리케이션이다.** ALB는 TLS 종단과 라우팅만 한다 — ALB 단 인증
   (`authenticate-cognito` · `jwt-validation`)은 `/v1/*`·`/api/*` 양 경로 모두 채택하지 않는다.
   dev·prod 동일하고, 이중 인증도 두지 않는다. 근거는 도메인·인증서 부재가 아니라 **검증 판단을
   앱 계층 한 곳에 두는 결정**이다(backend ADR-0007) — 도메인을 확보해도 ALB 인증으로 돌아가지 않는다.
4. **검증 지점의 종착지는 backend Spring Security 계층이다.** 현행 검증자 auth-proxy
   (`ai-telemetry-pipeline`, TypeScript)는 과도기 구성이며 폐기 대상이다 — infra ADR-0023의
   "한시적 구성" 자기 규정이 맞다.
5. **검증 통과 시 신원 헤더 4종**(`x-pulsemetry-token-id` · `-tenant-id` · `-installation-id` ·
   `-member-id`)을 부여하고 `Authorization`을 제거한다. 소비 지점은 collector 신원 전파 3요소를 거친
   processor 스탬핑이다([`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §3·§4).

## Constraints

- **`TOKEN_HASH_SECRET`은 회전할 수 없다.** 키가 바뀌면 발급된 모든 토큰의 `token_hash`가 매칭 불가가
  되어 전 클라이언트가 401을 받는다. 회전하려면 토큰 전량 재발급 또는 이중 키 검증이 선행돼야 한다.
  발급자(backend)와 검증자(auth-proxy → 이관 후 backend)가 같은 시크릿을 받아야 한다.
- 불투명 토큰에는 클레임이 없다. manifest revision 대조 같은 클레임 기반 검사는 텔레메트리 경로에
  적용할 수 없고, 사용자 세션 AT가 실리는 관리자 API 경로의 몫이다(backend ADR-0007 —
  그 경로와 AT 발급은 아직 구현되지 않았다).

## Alternatives Considered

- **Cognito JWT + ALB 검증 (infra ADR-0008 원안)** — 사용자 진실원이 Cognito와 `enrollment.members`
  둘로 갈라지고, 동기화·클레임 주입에 Lambda가 늘며, 불일치를 발견해도 우리가 새 토큰을 서명해 줄 수
  없다. backend ADR-0007이 같은 근거로 기각했다.
- **auth-proxy를 항구 구성으로 유지** — 검증 로직과 발급 로직이 다른 레포·다른 언어로 갈라진 상태가
  고착된다. 검증이 발급자의 DB와 시크릿에 전적으로 의존하므로 같은 코드베이스(backend)에 두는 편이 맞다.

## Consequences/Tradeoffs

### Positive

- 통과·거부의 근거가 앱 계층 한 곳에 모이고, Cognito·Lambda·푸시 인프라 의존이 생기지 않는다.
- 네 레포가 참조할 인증 모델의 단일 결정 지점이 생긴다.

### Negative

- ALB가 무효 토큰을 걸러 주지 않으므로 인증 실패 트래픽까지 앱이 받는다.
- 이관 완료 전까지 검증자가 두 시대(auth-proxy 현행 / backend 예정)로 나뉘어 문서가 둘을 함께
  설명해야 한다.

## Follow-up

- backend ADR-0007(collector 이관 + 인증 계층)이 진행되면 검증 지점을 Spring Security로 이관하고,
  auth-proxy와 그 ECR 레포(`soma-376/auth-proxy`)·infra ADR-0022의 4·8·10번(태스크·리스너·로그 그룹)을
  함께 정리한다.
- `lib/prod/edge-stack.ts`의 Cognito 구축 코드(User Pool·`AuthenticateCognitoAction`)와 모드 A의
  `/v1/*` `authenticateJwt`는 **의도적 잔존**이다 — Spring Security 대체 시 함께 걷어낸다.

## References

- [`../contracts/telemetry-ingest.md`](../contracts/telemetry-ingest.md) §3·§4 — 전송·검증·신원 전파의 현재 합의
- backend ADR-0003(2단 토큰) · ADR-0007(인증 계층) / infra ADR-0008(대체됨) · ADR-0023(dev auth-proxy) /
  telemetryctl ADR-0001(인라인 프록시 토폴로지)
