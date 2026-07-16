# myiam-cli AIコーディングエージェントプラグイン

[한국어](../README.md) | [English](README.en.md)

[myiam-cli](https://github.com/myiam-io/myiam-cli)用AIコーディングエージェントコマンドリファレンス — MyIAMサービス設定（OAuth2クライアント、ログインタイプ、ポリシー、利用規約、ユーザーフィールド、UI/ブランディング、API設定）をターミナルから管理します。

## インストール

### Claude Code

```bash
# マーケットプレイスに登録（初回のみ）
claude plugin marketplace add myiam-io/myiam-cli-plugin

# プラグインをインストール
claude plugin install myiam@myiam
```

ローカル開発:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
claude --plugin-dir ./myiam-cli-plugin
```

### OpenAI Codex CLI

```bash
# マーケットプレイスに登録（初回のみ）
codex plugin marketplace add myiam-io/myiam-cli-plugin

# プラグインをインストール
codex plugin install myiam
```

ローカル開発:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
codex --plugin-dir ./myiam-cli-plugin
```

### Google Gemini CLI

```bash
gemini extensions install https://github.com/myiam-io/myiam-cli-plugin
```

ローカル開発:

```bash
git clone https://github.com/myiam-io/myiam-cli-plugin.git
gemini extensions link ./myiam-cli-plugin
```

## 前提条件

[myiam-cli](https://github.com/myiam-io/myiam-cli)をインストール:

```bash
# macOS / Linux (Homebrew)
brew tap myiam-io/tap
brew install myiam-cli
```

ログイン:

```bash
myiam-cli login
```

## 含まれる内容

| エージェント | ファイル | スコープ |
|---|---|---|
| Claude Code | `skills/myiam/SKILL.md` | `/myiam`スキル — MyIAMサービス/管理設定関連の質問で自動トリガー |
| Codex CLI | `skills/myiam/SKILL.md` | `@myiam`スキル — MyIAMサービス/管理設定関連の質問で自動トリガー |
| Gemini CLI | `skills/myiam/SKILL.md` | extensionスキル — MyIAMサービス/管理設定関連の質問で自動トリガー |

すべてのファイルにmyiam-cliの完全なコマンドリファレンスが含まれています:
- ログインと対象サービスの選択（login, service list, service use）
- OAuth2クライアント設定（redirect URI、scope、grant type、トークンTTLプリセット）
- サービス概要、情報、UI/ブランディング、ティア、API設定
- SNSログインを含むログインタイプ（Naver、Kakao、Google、Apple）
- ポリシーとポリシーバージョン、利用規約と利用規約バージョン
- ユーザーフィールド（システムフィールド及びカスタムフィールド）

## アップデート

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

## アンインストール

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

## ライセンス

Apache 2.0。詳細は[LICENSE](../LICENSE)をご覧ください。
