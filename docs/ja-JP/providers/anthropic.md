---
summary: "OpenClaw で API キーまたは setup-token を使用して Anthropic Claude を利用する"
read_when:
  - OpenClaw で Anthropic モデルを使用したい場合
  - API キーの代わりに setup-token を使用したい場合
title: "Anthropic"
---

# Anthropic（Claude）

Anthropic は **Claude** モデルファミリーを開発し、API 経由でアクセスを提供しています。
OpenClaw では、API キーまたは **setup-token** で認証できます。

## オプション A：Anthropic API キー

**最適な用途：** 標準的な API アクセスと従量課金制。
Anthropic Console で API キーを作成してください。

### CLI セットアップ

```bash
openclaw onboard
# 選択: Anthropic API key

# または非インタラクティブ
openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
```

### 設定スニペット

```json5
{
  env: { ANTHROPIC_API_KEY: "sk-ant-..." },
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Thinking のデフォルト設定（Claude 4.6）

- Anthropic Claude 4.6 モデルは、明示的な thinking レベルが設定されていない場合、OpenClaw では `adaptive` thinking がデフォルトになります。
- メッセージごとに上書きするには（`/think:<level>`）、またはモデルパラメータで設定できます：
  `agents.defaults.models["anthropic/<model>"].params.thinking`
- 関連する Anthropic ドキュメント：
  - [Adaptive thinking](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
  - [Extended thinking](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

## Fast モード（Anthropic API）

OpenClaw の共有 `/fast` トグルは、Anthropic API キーのトラフィックもサポートしています。

- `/fast on` は `service_tier: "auto"` にマッピングされます
- `/fast off` は `service_tier: "standard_only"` にマッピングされます
- 設定のデフォルト：

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-sonnet-4-5": {
          params: { fastMode: true },
        },
      },
    },
  },
}
```

重要な制限事項：

- これは **API キーのみ** です。Anthropic の setup-token / OAuth 認証では OpenClaw の fast モードのティア注入は機能しません。
- OpenClaw は `api.anthropic.com` への直接リクエストに対してのみ Anthropic のサービスティアを注入します。`anthropic/*` をプロキシやゲートウェイ経由でルーティングする場合、`/fast` は `service_tier` を変更しません。
- Anthropic はレスポンスの `usage.service_tier` に有効なティアを報告します。Priority Tier の容量がないアカウントでは、`service_tier: "auto"` でも `standard` に解決される場合があります。

## プロンプトキャッシング（Anthropic API）

OpenClaw は Anthropic のプロンプトキャッシング機能をサポートしています。これは **API のみ** の機能で、サブスクリプション認証ではキャッシュ設定は機能しません。

### 設定

モデル設定で `cacheRetention` パラメータを使用します：

| 値      | キャッシュ時間 | 説明                               |
| ------- | -------------- | ---------------------------------- |
| `none`  | キャッシュなし | プロンプトキャッシングを無効にする |
| `short` | 5 分           | API キー認証のデフォルト           |
| `long`  | 1 時間         | 拡張キャッシュ（beta フラグが必要）|

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

### デフォルト値

Anthropic API キー認証を使用する場合、OpenClaw はすべての Anthropic モデルに対して自動的に `cacheRetention: "short"`（5 分キャッシュ）を適用します。設定で `cacheRetention` を明示的に設定することで上書きできます。

### エージェントごとの cacheRetention オーバーライド

モデルレベルのパラメータをベースラインとして使用し、`agents.list[].params` で特定のエージェントを上書きします。

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" }, // ほとんどのエージェントのベースライン
        },
      },
    },
    list: [
      { id: "research", default: true },
      { id: "alerts", params: { cacheRetention: "none" } }, // このエージェントのみ上書き
    ],
  },
}
```

キャッシュ関連パラメータの設定マージ順序：

1. `agents.defaults.models["provider/model"].params`
2. `agents.list[].params`（`id` が一致する場合、キーで上書き）

これにより、あるエージェントは長期キャッシュを保持しながら、同じモデルを使用する別のエージェントがキャッシュを無効にして、バースト/低再利用トラフィックでの書き込みコストを避けることができます。

### Bedrock Claude に関する注意事項

- Bedrock 上の Anthropic Claude モデル（`amazon-bedrock/*anthropic.claude*`）は、設定されている場合 `cacheRetention` のパススルーを受け付けます。
- Anthropic 以外の Bedrock モデルは実行時に `cacheRetention: "none"` が強制されます。
- Anthropic API キーのスマートデフォルトも、明示的な値が設定されていない場合、Bedrock 上の Claude モデル参照に対して `cacheRetention: "short"` を設定します。

### レガシーパラメータ

古い `cacheControlTtl` パラメータは後方互換性のために引き続きサポートされています：

- `"5m"` は `short` にマッピングされます
- `"1h"` は `long` にマッピングされます

新しい `cacheRetention` パラメータへの移行をお勧めします。

OpenClaw は Anthropic API リクエストに `extended-cache-ttl-2025-04-11` beta フラグを含めています。プロバイダーヘッダーを上書きする場合はこれを保持してください（[/gateway/configuration](/gateway/configuration) を参照）。

## 1M コンテキストウィンドウ（Anthropic ベータ）

Anthropic の 1M コンテキストウィンドウはベータゲートが設けられています。OpenClaw では、サポートされている Opus/Sonnet モデルに対して `params.context1m: true` を設定することで、モデルごとに有効化できます。

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { context1m: true },
        },
      },
    },
  },
}
```

OpenClaw はこれを Anthropic リクエストの `anthropic-beta: context-1m-2025-08-07` にマッピングします。

これは、そのモデルに対して `params.context1m` が明示的に `true` に設定されている場合にのみ有効になります。

要件：Anthropic がそのクレデンシャルで長いコンテキストの使用を許可している必要があります
（通常は API キー課金、または Extra Usage が有効なサブスクリプションアカウント）。
それ以外の場合、Anthropic は次のエラーを返します：
`HTTP 429: rate_limit_error: Extra usage is required for long context requests`

注意：Anthropic は現在、OAuth/サブスクリプショントークン（`sk-ant-oat-*`）を使用した
`context-1m-*` ベータリクエストを拒否します。OpenClaw は OAuth 認証の場合に
context1m ベータヘッダーを自動的にスキップし、必要な OAuth ベータを保持します。

## オプション B：Claude setup-token

**最適な用途：** Claude サブスクリプションを使用する場合。

### setup-token の入手方法

setup-token は Anthropic Console ではなく、**Claude Code CLI** によって作成されます。**任意のマシン**で実行できます：

```bash
claude setup-token
```

トークンを OpenClaw に貼り付けます（ウィザード：**Anthropic token (paste setup-token)**）。またはゲートウェイホストで実行します：

```bash
openclaw models auth setup-token --provider anthropic
```

別のマシンでトークンを生成した場合は、貼り付けてください：

```bash
openclaw models auth paste-token --provider anthropic
```

### CLI セットアップ（setup-token）

```bash
# オンボーディング中に setup-token を貼り付ける
openclaw onboard --auth-choice setup-token
```

### 設定スニペット（setup-token）

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## 注意事項

- `claude setup-token` で setup-token を生成して貼り付けるか、ゲートウェイホストで `openclaw models auth setup-token` を実行してください。
- Claude サブスクリプションで「OAuth token refresh failed …」が表示される場合は、setup-token で再認証してください。[/gateway/troubleshooting#oauth-token-refresh-failed-anthropic-claude-subscription](/gateway/troubleshooting#oauth-token-refresh-failed-anthropic-claude-subscription) を参照してください。
- 認証の詳細と再利用ルールは [/concepts/oauth](/concepts/oauth) にあります。

## トラブルシューティング

**401 エラー / トークンが突然無効になる**

- Claude サブスクリプション認証は期限切れになったり、取り消されたりすることがあります。`claude setup-token` を再実行して、**ゲートウェイホスト**に貼り付けてください。
- Claude CLI ログインが別のマシンにある場合は、ゲートウェイホストで `openclaw models auth paste-token --provider anthropic` を使用してください。

**No API key found for provider "anthropic"**

- 認証は**エージェントごと**です。新しいエージェントはメインエージェントのキーを継承しません。
- そのエージェントに対してオンボーディングを再実行するか、ゲートウェイホストで setup-token / API キーを貼り付けて、`openclaw models status` で確認してください。

**No credentials found for profile `anthropic:default`**

- `openclaw models status` を実行して、どの認証プロファイルがアクティブかを確認してください。
- オンボーディングを再実行するか、そのプロファイルに setup-token / API キーを貼り付けてください。

**No available auth profile (all in cooldown/unavailable)**

- `openclaw models status --json` の `auth.unusableProfiles` を確認してください。
- 別の Anthropic プロファイルを追加するか、クールダウンが終わるまで待ってください。

詳細：[/gateway/troubleshooting](/gateway/troubleshooting) および [/help/faq](/help/faq)
