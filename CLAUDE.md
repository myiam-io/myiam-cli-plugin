# CLAUDE.md

이 파일은 Claude Code (claude.ai/code)가 이 저장소에서 작업할 때 참고하는 가이드입니다.

## 프로젝트 개요

[myiam-cli](https://github.com/myiam-io/myiam-cli)용 AI 코딩 에이전트 플러그인. myiam-cli는 MyIAM 서비스(OAuth2 클라이언트, 로그인 타입, 정책, 약관, 사용자 필드, UI/브랜딩, API 설정 등)를 관리하는 관리자 도구이며, 이 플러그인은 Claude Code(`/myiam-cli`)와 Codex CLI(`@myiam-cli`) 스킬을 통해 전체 명령어 레퍼런스를 제공한다.

코드 프로젝트가 **아님** — 빌드, 테스트, 컴파일 대상 소스가 없다. 선언적 설정 파일과 스킬 마크다운 파일로만 구성된다.

## 저장소 구조

- `.claude-plugin/plugin.json` — Claude Code 플러그인 메타데이터
- `.claude-plugin/marketplace.json` — Claude Code 마켓플레이스 등록 설정
- `.codex-plugin/plugin.json` — Codex CLI 플러그인 메타데이터
- `gemini-extension.json` — Gemini CLI extension 메타데이터
- `skills/myiam-cli/SKILL.md` — `/myiam-cli`(Claude Code) · `@myiam-cli`(Codex CLI) · Gemini CLI extension이 공통으로 쓰는 스킬 정의. 이 저장소의 핵심 산출물로, myiam-cli를 조작할 때 사용하는 전체 명령어 레퍼런스. frontmatter의 `name`과 `description`이 스킬 자동 트리거 조건을 결정한다. **원본은 `../myiam-cli/skills/myiam-cli/SKILL.md`** — 이 파일은 그 사본이다 (아래 "스킬 편집" 참고).
- `README.md` / `docs/README.en.md` / `docs/README.ja.md` — 설치·업데이트·제거 안내 (한국어/영어/일본어). 셋 다 같은 내용을 유지해야 한다.
- `scripts/bump-version.sh` — `plugin.json`(Claude) · `marketplace.json` · `plugin.json`(Codex) · `gemini-extension.json` 4개 파일의 `version` 필드를 한 번에 변경. 사용법: `scripts/bump-version.sh 1.1.0`

## 개발

로컬 실행:
```bash
claude --plugin-dir ./myiam-cli-plugin
```

`myiam-io/myiam-cli-plugin` GitHub 저장소에 push하면 배포된다. 사용자 설치:
```bash
claude plugin marketplace add myiam-io/myiam-cli-plugin
claude plugin install myiam-cli@myiam-cli
```

버전을 올릴 때는 4개 메타데이터 파일을 수동으로 각각 고치지 말고 `scripts/bump-version.sh <version>`을 사용한다.

## 관련 저장소

- `../myiam-cli/` — myiam-cli 원본 소스 (Go). **`skills/myiam-cli/SKILL.md`의 원본이 이 저장소에 있다** (`../myiam-cli/skills/myiam-cli/SKILL.md`). 그 외 `cmd/schema.go`(`myiam-cli schema` 출력의 원본)와 `docs/commands.md`가 명령어 레퍼런스의 최신 근거
- `../../myiam-io/homebrew-tap/` — Homebrew Formula. 배포 버전/설치 경로 변경 시 참조

## 스킬 편집

`skills/myiam-cli/SKILL.md`는 `../myiam-cli/skills/myiam-cli/SKILL.md`의 사본이다 — **원본을 그쪽에서 먼저 수정한 뒤 이 저장소로 복사**한다 (`cp ../myiam-cli/skills/myiam-cli/SKILL.md skills/myiam-cli/SKILL.md`). 이 저장소에서 직접 내용을 고치지 않는다 — 두 파일이 갈라지면 다음 동기화 때 덮어써진다. myiam-cli의 실제 명령어와 항상 동기화 상태를 유지해야 한다. 주요 규칙:

- 복잡한 중첩 데이터(패널 전체 수정)를 다루는 create/update 예시는 `--from-stdin` 패턴 사용 (`echo '{"data":{...}}' | myiam-cli ... --from-stdin`)
- 모든 명령어의 기본 출력은 JSON — 사람이 볼 때만 `--output table` 사용
- SKILL.md frontmatter의 `description` 필드가 스킬 자동 트리거를 결정하므로 의도 매칭에 정확하게 유지
- 명령어 추가·삭제·변경 시 반드시 `../myiam-cli/cmd/schema.go`와 대조해 동기화
- 하단의 워크플로우 가이드가 사용자 의도 → 명령어 순서를 매핑
