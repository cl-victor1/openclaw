---
summary: "`openclaw models` の CLI リファレンス（status/list/set/scan、エイリアス、フォールバック、認証）"
read_when:
  - デフォルトモデルを変更したり、プロバイダーの認証ステータスを確認したいとき
  - 利用可能なモデル/プロバイダーをスキャンしたり、認証プロファイルをデバッグしたいとき
title: "models"
---

# `openclaw models`

モデルの検出、スキャン、設定（デフォルトモデル、フォールバック、認証プロファイル）。

関連ドキュメント:

- プロバイダー + モデル: [モデル](/providers/models)
- プロバイダー認証設定: [はじめに](/start/getting-started)

## 主要コマンド

```bash
openclaw models status
openclaw models list
openclaw models set <model-or-alias>
openclaw models scan
```

`openclaw models status` は解決されたデフォルト/フォールバックと認証の概要を表示します。
プロバイダーの使用状況スナップショットが利用可能な場合、OAuth/トークンステータスセクションに
プロバイダーの使用状況ヘッダーが含まれます。
`--probe` を追加すると、設定済みの各プロバイダープロファイルに対してライブ認証プローブを実行します。
プローブは実際のリクエストです（トークンを消費し、レート制限をトリガーする可能性があります）。
`--agent <id>` を使用して、設定済みエージェントのモデル/認証状態を検査します。省略した場合、
`OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR` が設定されていればそれを使用し、そうでなければ
設定済みのデフォルトエージェントを使用します。

注意事項:

- `models set <model-or-alias>` は `provider/model` またはエイリアスを受け入れます。
- モデル参照は**最初の** `/` で分割して解析されます。モデル ID に `/` が含まれる場合（OpenRouter スタイル）、プロバイダープレフィックスを含めてください（例: `openrouter/moonshotai/kimi-k2`）。
- プロバイダーを省略した場合、OpenClaw は入力をエイリアスまたは**デフォルトプロバイダー**のモデルとして扱います（モデル ID に `/` がない場合のみ機能します）。

### `models status`

オプション:

- `--json`
- `--plain`
- `--check`（終了コード 1=期限切れ/欠落、2=期限切れ間近）
- `--probe`（設定済み認証プロファイルのライブプローブ）
- `--probe-provider <name>`（1つのプロバイダーをプローブ）
- `--probe-profile <id>`（繰り返しまたはカンマ区切りのプロファイル ID）
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`
- `--agent <id>`（設定済みエージェント ID。`OPENCLAW_AGENT_DIR`/`PI_CODING_AGENT_DIR` を上書き）

## エイリアス + フォールバック

```bash
openclaw models aliases list
openclaw models fallbacks list
```

## 認証プロファイル

```bash
openclaw models auth add
openclaw models auth login --provider <id>
openclaw models auth setup-token
openclaw models auth paste-token
```

`models auth login` はプロバイダープラグインの認証フロー（OAuth/API キー）を実行します。
インストール済みのプロバイダーを確認するには `openclaw plugins list` を使用してください。

注意事項:

- `setup-token` はセットアップトークン値の入力を求めます（任意のマシンで `claude setup-token` を使用して生成します）。
- `paste-token` は他の場所または自動化から生成されたトークン文字列を受け入れます。
- Anthropic ポリシーに関する注意: setup-token サポートは技術的な互換性です。Anthropic は過去に Claude Code 外でのサブスクリプション使用をブロックしたことがあるため、広く使用する前に現在の利用規約を確認してください。
