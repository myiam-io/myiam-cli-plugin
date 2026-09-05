# myiam-cli Plugin for AI Coding Agents

[한국어](../README.md) | [日本語](README.ja.md)

[myiam-cli](https://github.com/myiam-io/myiam-cli) command reference for AI coding agents — manage MyIAM service settings (OAuth2 client, login types, policies, terms, user fields, UI/branding, API config) from the terminal.

## Install

### Claude Code

```bash
# Add this repo as a marketplace (one-time)
claude plugin marketplace add myiam-io/myiam-cli-plugin

# Install the plugin
claude plugin install myiam@myiam
```

Local development:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
claude --plugin-dir ./myiam-cli-plugin
```

### OpenAI Codex CLI

```bash
# Add this repo as a marketplace (one-time)
codex plugin marketplace add myiam-io/myiam-cli-plugin

# Install the plugin
codex plugin install myiam
```

Local development:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
codex --plugin-dir ./myiam-cli-plugin
```

### Google Antigravity CLI

```bash
agy plugin install https://github.com/myiam-io/myiam-cli-plugin
```

Local development:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
agy plugin install ./myiam-cli-plugin
```

During the transition, the legacy Gemini CLI (`gemini extensions install ...`) still installs it.

## Prerequisites

Install [myiam-cli](https://github.com/myiam-io/myiam-cli):

```bash
# macOS / Linux (Homebrew)
brew tap myiam-io/tap
brew install myiam-cli
```

Then login:

```bash
myiam-cli login
```

## What's included

| Agent | File | Scope |
|---|---|---|
| Claude Code | `skills/myiam/SKILL.md` | `/myiam` skill — auto-triggers on MyIAM service/admin queries |
| Codex CLI | `skills/myiam/SKILL.md` | `@myiam` skill — auto-triggers on MyIAM service/admin queries |
| Antigravity CLI | `skills/myiam/SKILL.md` | Plugin skill — auto-triggers on MyIAM service/admin queries |

All files cover the full myiam-cli command reference:
- Login & target service selection (`login`, `service list`, `service use`)
- OAuth2 client configuration (redirect URIs, scopes, grant types, token TTL presets)
- Service overview, info, UI/branding, tier, API config
- Login types incl. SNS providers (Naver, Kakao, Google, Apple)
- Policies & policy versions, terms & term versions
- User fields (system & custom)

## Update

### Claude Code

```bash
claude plugin update myiam
```

### Codex CLI

```bash
codex plugin upgrade myiam
```

### Antigravity CLI

```bash
agy plugin install https://github.com/myiam-io/myiam-cli-plugin
```

## Uninstall

### Claude Code

```bash
claude plugin uninstall myiam
```

### Codex CLI

```bash
codex plugin uninstall myiam
```

### Antigravity CLI

```bash
agy plugin uninstall myiam
```

## License

Apache 2.0. See [LICENSE](../LICENSE) for details.
