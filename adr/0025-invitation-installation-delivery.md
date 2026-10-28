# 0025. 초대 설치 링크와 사전 검증은 최종 enroll 소비와 분리한다

## Status

Accepted

## Context

`pulsemetry-backend`는 초대 메일과 부트스트랩을, `pulsemetry-frontend`는 공개 설치 안내를,
`telemetryctl`은 릴리스 CLI를, `infra`는 SES와 공개 진입점의 요청 제한을 소유한다.
로그인 주소와 원시 설치 명령만 있는 메일에서는 공개 설치 페이지로 이어지는 경로가 없었다.
형식만 검사하던 부트스트랩은 사용할 수 없는 코드에도 CLI를 내려받게 했다.

## Decision

1. 설치 가능한 초대 메일에는 별도 `installation-page-url`에 정규화한 `code` query를 붙인다.
   dev 값은 `https://dev.pulsemetry.co.kr/install`이다. SSO `invitation-accept-url`에는 코드를
   붙이지 않는다. 일반 초대·설치 전용·재발급 모두 같은 규칙을 따르며 설치 권한을 소진하면
   설치 링크와 명령을 생략한다. 암호화 outbox·멱등 적재·미발송 취소·재시도는 보존한다.
2. `/install`은 세션 없이 OS별 명령을 제공한다. 페이지 열기·링크 미리보기는 코드를 소비하지
   않는다. query 전달은 이 설치 동선에 한정한다. `no-store`와 `no-referrer`로 캐시·외부
   referrer 전달을 막고 일반 로그·제목·오류에는 코드를 기록하지 않는다.
3. `/unix`·`/windows`는 정규화·형식 검사 후 SHA-256 해시로 DB 상태를 읽는다. 유효한 코드에만
   다운로드 스크립트를 준다. 누락·형식 오류·미존재·만료·사용·취소는 HTTP 200 `text/plain`으로
   고정된 이유를 stderr에 출력하고 실패 종료하는 스크립트를 반환한다. 모든 스크립트는
   `Cache-Control: no-store`를 사용한다. DB 장애는 서버 오류이며 정상 스크립트로 우회하지 않는다.
4. 사전 조회는 설치·구성원·초대 상태를 변경하지 않는다. `POST /api/v1/enroll`이 트랜잭션 안에서
   조건부 UPDATE로 다시 검사하고 소비한다. 조회 후 취소·만료·다른 요청의 소비를 최종 단계에서
   거부하며 동일 코드의 동시 성공은 하나뿐이다. 응답은 기존 정확한 네 키를 유지한다.
5. infra는 부트스트랩 URI를 한 번 URL decode한 뒤 `/unix`·`/windows`와 정확히 일치시킨다.
   기존 등록 API는 raw URI의 정확 일치를 유지한다. 이 경로들은 출발지 IP별 합산
   300회/300초 예산을 공유한다. 차단은 429와 `Retry-After: 60`이다. 전달 헤더를 집계 키로
   쓰거나 앱 limiter를 추가하지 않는다. 이 정책의 dev 배포·실제 검증이 사전 조회 기능
   공개보다 선행한다.
6. CLI는 GitHub Releases의 게시된 `v0.1.0-rc.2` CLI 6종과 `SHA256SUMS`를 소비한다.
   태그를 고정하고 필수 파일·목록·해시를 검증한 뒤 enrollment 이미지의 versioned 디렉터리에
   포함한다. 조회 실패·파일 누락·해시 불일치는 빌드를 실패시킨다. 로컬 재빌드·latest 자동 선택·
   GUI 대체는 허용하지 않는다. 공개 `/bin/pulsemetry_*` 매핑은 유지한다.

## Alternatives Considered

- 로그인 링크에 코드를 붙이면 SSO와 설치 권한의 경계가 섞인다. 별도 설치 URL을 사용한다.
- 프론트 상태 조회 API는 설치 클릭 전의 상태만 알려 주며 별도 공개 표면을 늘린다. 기존
  부트스트랩에서 확인하고 enroll에서 재검증한다.
- 사전 조회에서 코드를 소비하면 메일 미리보기·명령 재시도로 설치가 막힌다. 소비는 enroll에 둔다.

## Consequences/Tradeoffs

### Positive

- 실제 수신 메일부터 공개 페이지·게시된 CLI·enroll까지 같은 초대의 동선을 검증할 수 있다.
- 잘못된 코드로 불필요한 다운로드가 발생하지 않으며 기존 enroll 원자성이 유지된다.

### Negative

- 공개 부트스트랩이 초대 상태를 구분하므로 DB 읽기와 코드 탐색 표면이 늘어난다. WAF 선행
  적용, 고정 오류 문구, 로그 마스킹을 함께 요구한다.
- 설치 URL은 메일함·브라우저 기록에 남을 수 있다. 짧은 수명·일회성·취소와 query 보호가 필요하다.
- HTTP 429·서버 오류에서 `curl -fsSL … | sh` 전체 파이프의 종료 코드는 이 변경이 보장하지 않는다.

## Acceptance Criteria

메일 분기·재발급·멱등성, 조회 비소비·최종 재검증·동시 단일 성공, 실제 sh/PowerShell 오류 실행,
CLI 6종 해시, WAF 혼합 경로 차단·다른 IP·복구를 검증한다. 실제 수신함의 메일에서 시작하는
dev enroll 성공 전에는 SES MessageId나 outbox `sent`만으로 통합 완료라고 판단하지 않는다.

### 수용 결과와 범위

- dev의 정상 관리 API·outbox·SES를 거친 실제 수신 메일에서 설치 페이지와 명령을 확인하고,
  게시된 rc.2 CLI로 Linux ARM64 설치를 마쳤다. CLI 성공 종료·로컬 설치 식별자·DB의 초대
  소비·설치 한 건 증가·구성원 활성화를 대조했다. CLI는 성공 HTTP 상태를 직접 노출하지
  않으므로 이 설치에서 HTTP 201을 직접 관측했다고 기록하지 않는다.
- 여섯 공개 바이너리의 해시·실행 형식과 업데이트 응답을 검증했다. 여섯 OS/CPU의 네이티브
  버전 실행은 CI 증거이며, 여섯 환경의 실제 enroll 성공을 뜻하지 않는다.
- 누락·형식 오류·미존재·만료·사용·취소의 고정 오류 스크립트를 실제 sh·PowerShell에서
  실행해 exit 1·고정 stderr·설치 경로 미생성을 확인했다. 다운로드·enroll 미호출은 받은
  스크립트와 고정 소스의 바이트 일치에 근거한다. 네트워크 패킷 캡처는 수행하지 않았다.
- 세 초대를 각각 네 번 조회해 비소비를 확인하고, 조회 후 취소·만료·사용을 최종 enroll이
  거부하는 것을 확인했다. 재발급 메일의 실제 수신과 새 링크를 확인한 뒤 동시 요청 여덟 건에서
  HTTP 201 한 건·409 일곱 건과 설치 한 건 증가를 관측했다. 원래 멱등 키의 재요청은 초대와
  outbox를 중복 생성하지 않았으며 재발급 전 코드는 거부됐다.
- WAF의 혼합 경로 차단·다른 IP·복구를 검증한 뒤 사전 조회를 공개했다. 현재 runtime 설정을
  보존한 이전 이미지 digest 복구와 새 이미지 재배포를 수행하고, 각 단계의 실제 이미지·health와
  공개 다운로드 여섯 건·업데이트 판정 열두 건을 다시 확인했다.

상세 수용 기록은 아래 티켓에서 관리한다. 개인 수신주소·초대 코드·토큰·메일 원문은 이 문서나
PR에 복사하지 않는다. CLI 설치의 원문 출력도 저장하지 않고 종료 코드와 출력 바이트 수만 남긴다.

## References

- [enrollment API 계약](../contracts/enrollment-api.md)
- [PROJ-266](https://team376.atlassian.net/browse/PROJ-266), [PROJ-267](https://team376.atlassian.net/browse/PROJ-267)
- [PROJ-268](https://team376.atlassian.net/browse/PROJ-268), [PROJ-269](https://team376.atlassian.net/browse/PROJ-269)
- [PROJ-146](https://team376.atlassian.net/browse/PROJ-146)
- [메일 링크 구현](https://github.com/soma-376/pulsemetry-backend/pull/46), [게시 CLI 공급](https://github.com/soma-376/pulsemetry-backend/pull/47), [부트스트랩 사전 검증](https://github.com/soma-376/pulsemetry-backend/pull/48)
- [설치 경로 WAF 정책](https://github.com/soma-376/infra/pull/62), [CLI 공급·복구 운영 절차](https://github.com/soma-376/pulsemetry-backend/blob/32c9cb279242492024dccaca97f2b1be2cee24d5/docs/cli-release-operations.md)
