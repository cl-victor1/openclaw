---
summary: "ウェイクと分離エージェント実行のためのWebhookイングレス"
read_when:
  - webhookエンドポイントの追加や変更
  - 外部システムをOpenClawに接続する
title: "Webhooks"
---

# Webhooks

GatewayはWe外部トリガー向けの小さなHTTP webhookエンドポイントを公開できます。

## 有効化

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    // オプション: 明示的な `agentId` ルーティングをこの許可リストに制限する。
    // 省略または "*" を含める場合、任意のエージェントを許可。
    // [] に設定すると明示的な `agentId` ルーティングをすべて拒否。
    allowedAgentIds: ["hooks", "main"],
  },
}
```

注:

- `hooks.enabled=true` の場合、`hooks.token` は必須です。
- `hooks.path` のデフォルトは `/hooks` です。

## 認証

すべてのリクエストはhookトークンを含む必要があります。ヘッダーを推奨します:

- `Authorization: Bearer <token>`（推奨）
- `x-openclaw-token: <token>`
- クエリ文字列トークンは拒否されます（`?token=...` は `400` を返します）。

## エンドポイント

### `POST /hooks/wake`

ペイロード:

```json
{ "text": "System line", "mode": "now" }
```

- `text` **必須**（文字列）: イベントの説明（例: "New email received"）。
- `mode` オプション（`now` | `next-heartbeat`）: 即時ハートビートをトリガーするか（デフォルト `now`）、次の定期チェックを待機するか。

効果:

- **メイン**セッションのシステムイベントをエンキュー
- `mode=now` の場合、即時ハートビートをトリガー

### `POST /hooks/agent`

ペイロード:

```json
{
  "message": "Run this",
  "name": "Email",
  "agentId": "hooks",
  "sessionKey": "hook:email:msg-123",
  "wakeMode": "now",
  "deliver": true,
  "channel": "last",
  "to": "+15551234567",
  "model": "openai/gpt-5.2-mini",
  "thinking": "low",
  "timeoutSeconds": 120
}
```

- `message` **必須**（文字列）: エージェントが処理するプロンプトまたはメッセージ。
- `name` オプション（文字列）: hookの人が読みやすい名前（例: "GitHub"）、セッションサマリーのプレフィックスとして使用。
- `agentId` オプション（文字列）: このhookを特定のエージェントにルーティング。不明なIDはデフォルトエージェントにフォールバック。設定した場合、解決されたエージェントのワークスペースと設定を使用してhookが実行されます。
- `sessionKey` オプション（文字列）: エージェントのセッションを識別するために使用するキー。デフォルトでは `hooks.allowRequestSessionKey=true` でない限り、このフィールドは拒否されます。
- `wakeMode` オプション（`now` | `next-heartbeat`）: 即時ハートビートをトリガーするか（デフォルト `now`）、次の定期チェックを待機するか。
- `deliver` オプション（ブール値）: `true` の場合、エージェントの応答がメッセージングチャンネルに送信されます。デフォルトは `true`。ハートビート確認のみの応答は自動的にスキップされます。
- `channel` オプション（文字列）: デリバリー用のメッセージングチャンネル。`last`、`whatsapp`、`telegram`、`discord`、`slack`、`mattermost`（プラグイン）、`signal`、`imessage`、`msteams` のいずれか。デフォルトは `last`。
- `to` オプション（文字列）: チャンネルの受信者識別子（例: WhatsApp/Signalの電話番号、TelegramのチャットID、Discord/Slack/Mattermost（プラグイン）のチャンネルID、MS TeamsのコンバセーションID）。デフォルトはメインセッションの最後の受信者。
- `model` オプション（文字列）: モデルオーバーライド（例: `anthropic/claude-3-5-sonnet` またはエイリアス）。制限されている場合は許可されたモデルリストに含まれている必要があります。
- `thinking` オプション（文字列）: Thinkingレベルオーバーライド（例: `low`、`medium`、`high`）。
- `timeoutSeconds` オプション（数値）: エージェント実行の最大時間（秒）。

効果:

- **分離**エージェントターンを実行（独自のセッションキー）
- 常に**メイン**セッションにサマリーを投稿
- `wakeMode=now` の場合、即時ハートビートをトリガー

## セッションキーポリシー（破壊的変更）

`/hooks/agent` ペイロードの `sessionKey` オーバーライドはデフォルトで無効です。

- 推奨: 固定の `hooks.defaultSessionKey` を設定し、リクエストオーバーライドを無効のままにする。
- オプション: 必要な場合のみリクエストオーバーライドを許可し、プレフィックスを制限する。

推奨設定:

```json5
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: false,
    allowedSessionKeyPrefixes: ["hook:"],
  },
}
```

互換性設定（レガシー動作）:

```json5
{
  hooks: {
    enabled: true,
    token: "${OPENCLAW_HOOKS_TOKEN}",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:"], // 強く推奨
  },
}
```

### `POST /hooks/<name>`（マッピング）

カスタムhook名は `hooks.mappings` を通じて解決されます（設定参照）。マッピングは任意のペイロードを `wake` または `agent` アクションに変換でき、オプションのテンプレートまたはコード変換を使用できます。

マッピングオプション（サマリー）:

- `hooks.presets: ["gmail"]` は組み込みGmailマッピングを有効化します。
- `hooks.mappings` では設定で `match`、`action`、テンプレートを定義できます。
- `hooks.transformsDir` + `transform.module` はカスタムロジック用のJS/TSモジュールをロードします。
  - `hooks.transformsDir`（設定されている場合）はOpenClaw設定ディレクトリ下の変換ルート（通常 `~/.openclaw/hooks/transforms`）内に留まる必要があります。
  - `transform.module` は有効な変換ディレクトリ内で解決される必要があります（トラバーサル/エスケープパスは拒否されます）。
- `match.source` を使用してジェネリックなインジェストエンドポイントを維持します（ペイロード駆動ルーティング）。
- TSトランスフォームにはTSローダー（例: `bun` または `tsx`）またはランタイムでプリコンパイルされた `.js` が必要です。
- マッピングに `deliver: true` + `channel`/`to` を設定して、返信をチャットサーフェスにルーティングします（`channel` のデフォルトは `last`、WhatsAppにフォールバック）。
- `agentId` はhookを特定のエージェントにルーティングします。不明なIDはデフォルトエージェントにフォールバック。
- `hooks.allowedAgentIds` は明示的な `agentId` ルーティングを制限します。省略（または `*` を含める）すると任意のエージェントを許可。`[]` に設定すると明示的な `agentId` ルーティングを拒否。
- `hooks.defaultSessionKey` はhookエージェント実行の明示的なキーが提供されない場合のデフォルトセッションを設定します。
- `hooks.allowRequestSessionKey` は `/hooks/agent` ペイロードが `sessionKey` を設定できるかどうかを制御します（デフォルト: `false`）。
- `hooks.allowedSessionKeyPrefixes` はオプションでリクエストペイロードとマッピングからの明示的な `sessionKey` 値を制限します。
- `allowUnsafeExternalContent: true` はそのhookの外部コンテンツセーフティラッパーを無効化します（危険; 信頼できる内部ソースのみ）。
- hookペイロードはデフォルトで信頼できないものとして扱われ、セーフティ境界でラップされます。特定のhookに対してこれを無効にする必要がある場合は、そのhookのマッピングに `allowUnsafeExternalContent: true` を設定してください（危険）。
- `openclaw webhooks gmail setup` は `openclaw webhooks gmail run` 用の `hooks.gmail` 設定を書き込みます。完全なGmail watchフローについては [Gmail Pub/Sub](/automation/gmail-pubsub) を参照してください。

## レスポンス

- `200`: `/hooks/wake` 成功
- `200`: `/hooks/agent`（非同期実行が受け付けられた）
- `401`: 認証失敗
- `429`: 同じクライアントから繰り返し認証失敗（`Retry-After` を確認）
- `400`: 無効なペイロード
- `413`: ペイロードが大きすぎる

## 例

```bash
curl -X POST http://127.0.0.1:18789/hooks/wake \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"text":"New email received","mode":"now"}'
```

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'x-openclaw-token: SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","wakeMode":"next-heartbeat"}'
```

### 別のモデルを使用する

エージェントペイロード（またはマッピング）に `model` を追加して、その実行のモデルをオーバーライドします:

```bash
curl -X POST http://127.0.0.1:18789/hooks/agent \
  -H 'x-openclaw-token: SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"message":"Summarize inbox","name":"Email","model":"openai/gpt-5.2-mini"}'
```

`agents.defaults.models` を強制する場合は、オーバーライドモデルがそこに含まれていることを確認してください。

```bash
curl -X POST http://127.0.0.1:18789/hooks/gmail \
  -H 'Authorization: Bearer SECRET' \
  -H 'Content-Type: application/json' \
  -d '{"source":"gmail","messages":[{"from":"Ada","subject":"Hello","snippet":"Hi"}]}'
```

## セキュリティ

- hookエンドポイントはループバック、テールネット、または信頼できるリバースプロキシの後ろに置いてください。
- 専用のhookトークンを使用してください。Gatewayの認証トークンを再利用しないでください。
- 繰り返しの認証失敗はブルートフォース試行を遅らせるためにクライアントアドレスごとにレートリミットされます。
- マルチエージェントルーティングを使用する場合、`hooks.allowedAgentIds` を設定して明示的な `agentId` 選択を制限してください。
- 呼び出し元が選択したセッションが必要でない限り、`hooks.allowRequestSessionKey=false` を維持してください。
- リクエスト `sessionKey` を有効にする場合、`hooks.allowedSessionKeyPrefixes` を制限してください（例: `["hook:"]`）。
- webhookログに機密の生のペイロードを含めないようにしてください。
- hookペイロードはデフォルトで信頼できないものとして扱われ、セーフティ境界でラップされます。特定のhookに対してこれを無効にする必要がある場合は、そのhookのマッピングに `allowUnsafeExternalContent: true` を設定してください（危険）。
