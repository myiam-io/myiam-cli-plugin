# myiam-cli AI 코딩 에이전트 플러그인

[English](docs/README.en.md) | [日本語](docs/README.ja.md)

[myiam-cli](https://github.com/myiam-io/myiam-cli)용 AI 코딩 에이전트 명령어 레퍼런스 — MyIAM 서비스 설정(OAuth2 클라이언트, 로그인 타입, 정책, 약관, 사용자 필드, UI/브랜딩, API 설정)을 터미널에서 관리합니다.

## 설치

### Claude Code

```bash
# 마켓플레이스 등록 (최초 1회)
claude plugin marketplace add myiam-io/myiam-cli-plugin

# 플러그인 설치
claude plugin install myiam@myiam
```

로컬 개발:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
claude --plugin-dir ./myiam-cli-plugin
```

### OpenAI Codex CLI

```bash
# 마켓플레이스 등록 (최초 1회)
codex plugin marketplace add myiam-io/myiam-cli-plugin

# 플러그인 설치
codex plugin install myiam
```

로컬 개발:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
codex --plugin-dir ./myiam-cli-plugin
```

### Google Gemini CLI

```bash
gemini extensions install https://github.com/myiam-io/myiam-cli-plugin
```

로컬 개발:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
gemini extensions link ./myiam-cli-plugin
```

## 사전 요구사항

[myiam-cli](https://github.com/myiam-io/myiam-cli) 설치:

```bash
# macOS / Linux (Homebrew)
brew tap myiam-io/tap
brew install myiam-cli
```

로그인:

```bash
myiam-cli login
```

## 포함 내용

| 에이전트 | 파일 | 범위 |
|---|---|---|
| Claude Code | `skills/myiam/SKILL.md` | `/myiam` 스킬 — MyIAM 서비스/관리자 설정 관련 질문 시 자동 트리거 |
| Codex CLI | `skills/myiam/SKILL.md` | `@myiam` 스킬 — MyIAM 서비스/관리자 설정 관련 질문 시 자동 트리거 |
| Gemini CLI | `skills/myiam/SKILL.md` | extension 스킬 — MyIAM 서비스/관리자 설정 관련 질문 시 자동 트리거 |

모든 파일이 myiam-cli 전체 명령어 레퍼런스를 포함합니다:
- 로그인 및 대상 서비스 선택 (login, service list, service use)
- OAuth2 클라이언트 설정 (redirect URI, scope, grant type, 토큰 TTL 프리셋)
- 서비스 개요, 정보, UI/브랜딩, 티어, API 설정
- SNS 로그인 포함 로그인 타입 (Naver, Kakao, Google, Apple)
- 정책 및 정책 버전, 약관 및 약관 버전
- 사용자 필드 (시스템 필드 및 커스텀 필드)

## 업데이트

### Claude Code

```bash
claude plugin update myiam
```

### Codex CLI

```bash
codex plugin upgrade myiam
```

### Gemini CLI

```bash
gemini extensions install https://github.com/myiam-io/myiam-cli-plugin
```

## 제거

### Claude Code

```bash
claude plugin uninstall myiam
```

### Codex CLI

```bash
codex plugin uninstall myiam
```

### Gemini CLI

```bash
gemini extensions uninstall myiam
```

## 라이선스

Apache 2.0. 자세한 내용은 [LICENSE](LICENSE)를 참고하세요.
