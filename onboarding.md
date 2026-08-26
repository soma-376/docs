# 온보딩

## 1. 클론 배치 — 반드시 형제로

에이전트 스킬과 문서 참조가 이 배치를 1차 경로로 가정한다. **한 부모 디렉토리 아래 형제로 클론한다.**

```bash
mkdir -p ~/work/soma-376 && cd ~/work/soma-376

for r in docs agent-skills pulsemetry-backend telemetryctl \
         ai-telemetry-pipeline infra team-376-llm-wiki .github; do
  gh repo clone "soma-376/$r"
done
```

부모 폴더(`soma-376/`)는 git 레포가 아니라 **컨테이너 폴더**다. 여기서 `git init`을 하지 않는다.

`rdb-schema`는 현재 원격이 없다(local-only). `otel-collector`는 superseded이므로 클론하지 않아도 된다.
레포별 역할은 [`architecture/repos.md`](architecture/repos.md).

## 2. AI 에이전트 셋업

팀은 **Claude Code와 Codex CLI를 혼용**한다. 스킬은 SKILL.md 표준 한 벌로 양쪽을 커버한다.

### 2-1. Claude Code

각 레포에 `.claude/settings.json`이 커밋돼 있어서 **레포를 열고 신뢰 프롬프트를 승인하면**
`soma-376/agent-skills` 마켓플레이스가 등록되고 `pulsemetry` 플러그인이 활성화된다.

자동 설치가 안 되면 수동으로 한다.

```bash
claude plugin marketplace add soma-376/agent-skills
claude plugin install pulsemetry@soma-376
```

세션 안에서는 `/plugin marketplace add soma-376/agent-skills` → `/plugin install pulsemetry@soma-376`.

확인:

```bash
claude plugin list          # pulsemetry@soma-376 가 보여야 한다
```

> 신뢰 프롬프트를 거부하면 플러그인이 설치되지 않는다. 위 수동 명령이 그때의 경로다.

### 2-2. Codex CLI

**경로 1 (기본) — 레포 단위 스킬.** 각 레포에 `.agents/skills` 심링크가 커밋돼 있다.
형제 배치를 지켰다면 **클론 직후부터 아무 설정 없이 인식된다.** Codex는 cwd부터 레포 루트까지 `.agents/skills`를 스캔한다.

**경로 2 (폴백) — 홈 설치.** 형제 배치가 아니거나 단일 레포만 클론한 환경에서는 심링크가 끊긴다(무해하다).
그때는 홈에 설치한다.

```bash
cd ~/work/soma-376/agent-skills
./scripts/install-codex-skills.sh          # ~/.agents/skills 에 심링크
./scripts/install-codex-skills.sh --uninstall
```

확인:

```bash
ls -la ~/.agents/skills/    # spec adr adr-new conventions
```

### 2-3. 스킬 4종

| 스킬 | 언제 |
|---|---|
| `spec` | 기능 구현·요구사항·설계 논의를 **시작하기 전** |
| `adr` | 아키텍처·계약·구조 관련 코드를 **읽거나 바꾸기 전** |
| `adr-new` | 새 설계 결정을 기록하거나 기존 ADR을 개정할 때 |
| `conventions` | 브랜치 생성·커밋·PR 등 git 작업 전 |

스킬을 업데이트하려면 `agent-skills`를 `git pull` 한다.
Codex 심링크는 디렉토리 전체를 가리키므로 스킬이 추가돼도 레포를 다시 만질 필요가 없다.
Claude 플러그인은 `claude plugin update pulsemetry`.

> **심링크와 설치 스크립트는 macOS·Linux 전제다.** 팀 전원이 macOS라는 전제 위에 서 있다.

## 3. 문서를 읽는 순서

1. [`product/prd.md`](product/prd.md) — 무엇을 왜 만드는가
2. [`architecture/overview.md`](architecture/overview.md) — 시스템이 어떻게 생겼는가
3. [`architecture/repos.md`](architecture/repos.md) — **내가 만질 레포가 무엇을 소유하는가**
4. 그 레포가 당사자인 [`contracts/`](contracts/README.md) 문서
5. 그 레포의 `AGENTS.md`와 `docs/adr/`

## 4. 첫 PR

git 작업은 [`.github` 레포의 `CONVENTION.md`](https://github.com/soma-376/.github/blob/main/CONVENTION.md)를 따른다
(에이전트는 `conventions` 스킬을 쓴다). 요약:

| 단계 | 규칙 |
|---|---|
| 티켓 | 항상 Jira 티켓에서 시작 |
| 브랜치 | `develop`에서 분기. **`<타입>/<KEY-N>-<슬러그>`** — 타입이 먼저 |
| 커밋 | **`<KEY-N> <타입>: 내용`** — 티켓 키를 맨 앞에. **Jira 스마트 커밋(`#명령`) 금지** |
| PR | base는 `develop`. 제목·플레이스홀더·assignee는 **건드리지 않는다**(워크플로가 채운다). 본문의 작업 내용만 쓴다 |
| 머지 | 기타 → `develop`는 **squash and merge**, 리뷰 승인 1명 이상. `develop` → `main`은 **merge commit** |

**예외**: `docs`·`agent-skills`·`rdb-schema`·`team-376-llm-wiki`·`.github`는 `main` 단일 브랜치다.

## 5. Claude Code 텔레메트리 (선택)

이 프로젝트는 자기 제품으로 자기 팀을 계측한다(prd.md §7 가설 4).
컨테이너 루트의 `.claude/settings.json`에 OTel 환경변수가 있고, **루트 cwd에서 실행할 때만** 적용된다.
`printenv`로는 확인되지 않는다 — 확인하려면 로컬 수신기를 띄워 end-to-end로 본다.
