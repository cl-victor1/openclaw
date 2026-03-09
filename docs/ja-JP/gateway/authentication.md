---
summary: "モデル認証: OAuth、APIキー、およびsetup-token"
read_when:
  - モデル認証またはOAuthの有効期限のデバッグ時
  - 認証または資格情報ストレージの文書化時
title: "認証"
---

# 認証

OpenClawはモデルプロバイダーに対してOAuthとAPIキーをサポートしています。常時稼働の
Gatewayホストでは、APIキーが通常最も予測可能なオプションです。サブスクリプション/
OAuthフローも、プロバイダーアカウントモデルに一致する場合にサポートされます。

完全なOAuthフローとストレージレイアウトについては[/concepts/oauth](/concepts/oauth)を参照してください。
SecretRefベースの認証（`env`/`file`/`exec`プロバイダー）については[シークレット管理](/gateway/secrets)を参照してください。
`models status --probe`で使用される資格情報の適格性/理由コードルールについては[認証資格情報セマンティクス](/auth-credential-semantics)を参照してください。

## 推奨セットアップ（APIキー、任意のプロバイダー）

長期間稼働するGatewayを運用している場合は、選択したプロバイダーのAPIキーから始めてください。
Anthropicの場合、APIキー認証が安全なパスであり、サブスクリプションの
setup-token認証よりも推奨されます。

1. プロバイダーコンソールでAPIキーを作成します。
2. **Gatewayホスト**（`openclaw gateway`を実行するマシン）に配置します。

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Gatewayがsystemd/launchdで実行されている場合、デーモンが読み取れるように
   `~/.openclaw/.env`にキーを配置することを推奨します:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

その後、デーモンを再起動（またはGatewayプロセスを再起動）して再確認します:

```bash
openclaw models status
openclaw doctor
```

環境変数を自分で管理したくない場合、オンボーディングウィザードでデーモン用の
APIキーを保存できます: `openclaw onboard`。

環境変数の継承（`env.shellEnv`、`~/.openclaw/.env`、systemd/launchd）の詳細については
[ヘルプ](/help)を参照してください。

## Anthropic: setup-token（サブスクリプション認証）

Claudeサブスクリプションを使用している場合、setup-tokenフローがサポートされています。
**Gatewayホスト**で実行します:

```bash
claude setup-token
```

その後、OpenClawに貼り付けます:

```bash
openclaw models auth setup-token --provider anthropic
```

トークンが別のマシンで作成された場合、手動で貼り付けます:

```bash
openclaw models auth paste-token --provider anthropic
```

以下のようなAnthropicエラーが表示された場合:

```
This credential is only authorized for use with Claude Code and cannot be used for other API requests.
```

…代わりにAnthropic APIキーを使用してください。

<Warning>
Anthropicのsetup-tokenサポートは技術的な互換性のみです。Anthropicは過去に
Claude Code以外でのサブスクリプション使用をブロックしたことがあります。ポリシーリスクが
許容可能であると判断した場合にのみ使用し、Anthropicの現在の利用規約を自分で確認してください。
</Warning>

手動トークン入力（任意のプロバイダー、`auth-profiles.json`に書き込み + 設定を更新）:

```bash
openclaw models auth paste-token --provider anthropic
openclaw models auth paste-token --provider openrouter
```

認証プロファイル参照は静的資格情報にもサポートされています:

- `api_key`資格情報は`keyRef: { source, provider, id }`を使用可能
- `token`資格情報は`tokenRef: { source, provider, id }`を使用可能

自動化向けチェック（期限切れ/不足時はexit `1`、期限切れ間近は`2`）:

```bash
openclaw models status --check
```

オプションの運用スクリプト（systemd/Termux）はこちらに記載:
[/automation/auth-monitoring](/automation/auth-monitoring)

> `claude setup-token`には対話的なTTYが必要です。

## モデル認証ステータスの確認

```bash
openclaw models status
openclaw doctor
```

## APIキーローテーション動作（Gateway）

一部のプロバイダーは、API呼び出しがプロバイダーのレート制限に達した場合に
代替キーでリクエストを再試行することをサポートしています。

- 優先順位:
  - `OPENCLAW_LIVE_<PROVIDER>_KEY`（単一オーバーライド）
  - `<PROVIDER>_API_KEYS`
  - `<PROVIDER>_API_KEY`
  - `<PROVIDER>_API_KEY_*`
- Googleプロバイダーは追加のフォールバックとして`GOOGLE_API_KEY`も含みます。
- 同じキーリストは使用前に重複排除されます。
- OpenClawはレート制限エラー（例:
  `429`、`rate_limit`、`quota`、`resource exhausted`）に対してのみ次のキーで再試行します。
- レート制限以外のエラーは代替キーで再試行されません。
- すべてのキーが失敗した場合、最後の試行からの最終エラーが返されます。

## 使用する資格情報の制御

### セッションごと（チャットコマンド）

`/model <alias-or-id>@<profileId>`を使用して、現在のセッションに特定のプロバイダー資格情報を固定します（プロファイルIDの例: `anthropic:default`、`anthropic:work`）。

`/model`（または`/model list`）でコンパクトなピッカーを表示、`/model status`で完全なビュー（候補 + 次の認証プロファイル、設定時のプロバイダーエンドポイント詳細も含む）を表示します。

### エージェントごと（CLIオーバーライド）

エージェントに明示的な認証プロファイル順序オーバーライドを設定します（そのエージェントの`auth-profiles.json`に保存）:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

`--agent <id>`で特定のエージェントをターゲットにします。省略すると設定されたデフォルトエージェントが使用されます。

## トラブルシューティング

### 「No credentials found」

Anthropicトークンプロファイルが見つからない場合、**Gatewayホスト**で`claude setup-token`を実行し、
再確認します:

```bash
openclaw models status
```

### トークンの期限切れ/期限切れ間近

`openclaw models status`を実行して、どのプロファイルが期限切れ間近か確認します。プロファイルが
見つからない場合、`claude setup-token`を再実行してトークンを再度貼り付けます。

## 要件

- Anthropicサブスクリプションアカウント（`claude setup-token`用）
- Claude Code CLIがインストール済み（`claude`コマンドが利用可能）
