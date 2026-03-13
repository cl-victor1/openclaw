---
summary: "ペアリングの概要: DM 送信者の承認とノードの参加許可"
read_when:
  - DM アクセス制御を設定する場合
  - 新しい iOS/Android ノードをペアリングする場合
  - OpenClaw のセキュリティ体制を確認する場合
title: "ペアリング"
---

# ペアリング

「ペアリング」は OpenClaw の明示的な**オーナー承認**ステップです。
以下の 2 つの場面で使用されます:

1. **DM ペアリング**（ボットとの会話を許可するユーザー）
2. **ノードペアリング**（ゲートウェイネットワークへの参加を許可するデバイス/ノード）

セキュリティのコンテキスト: [セキュリティ](/gateway/security)

## 1) DM ペアリング（インバウンドチャットアクセス）

チャンネルが DM ポリシー `pairing` で設定されている場合、未知の送信者には短いコードが発行され、承認されるまでメッセージは**処理されません**。

デフォルトの DM ポリシーについては: [セキュリティ](/gateway/security)

ペアリングコード:

- 8 文字、大文字、紛らわしい文字なし（`0O1I`）。
- **1 時間後に有効期限切れ**。ボットは新しいリクエストが作成された場合にのみペアリングメッセージを送信します（送信者ごとにおよそ 1 時間に 1 回）。
- 保留中の DM ペアリングリクエストはデフォルトでチャンネルごとに **3 件**が上限です。上限を超えたリクエストは、いずれかが有効期限切れか承認されるまで無視されます。

### 送信者を承認する

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

対応チャンネル: `telegram`、`whatsapp`、`signal`、`imessage`、`discord`、`slack`、`feishu`。

### 状態の保存場所

`~/.openclaw/credentials/` 以下に保存されます:

- 保留中のリクエスト: `<channel>-pairing.json`
- 承認済み許可リストストア:
  - デフォルトアカウント: `<channel>-allowFrom.json`
  - デフォルト以外のアカウント: `<channel>-<accountId>-allowFrom.json`

アカウントスコープの動作:

- デフォルト以外のアカウントはスコープ付きの許可リストファイルのみ読み書きします。
- デフォルトアカウントはチャンネルスコープのスコープなし許可リストファイルを使用します。

これらはアシスタントへのアクセスを制御するため、機密として扱ってください。

## 2) ノードデバイスペアリング（iOS/Android/macOS/ヘッドレスノード）

ノードは `role: node` を持つ**デバイス**としてゲートウェイに接続します。ゲートウェイはデバイスペアリングリクエストを作成し、承認が必要です。

### Telegram 経由でペアリングする（iOS の場合推奨）

`device-pair` プラグインを使用すると、Telegram から初めてのデバイスペアリングをすべて行えます:

1. Telegram でボットに `/pair` とメッセージを送信します。
2. ボットが 2 つのメッセージを返信します: 説明メッセージと別の**セットアップコード**メッセージ（Telegram でコピー/ペーストしやすい形式）。
3. スマートフォンで OpenClaw iOS アプリを開き、設定 → ゲートウェイに進みます。
4. セットアップコードを貼り付けて接続します。
5. Telegram に戻り、`/pair approve` を実行します。

セットアップコードは以下を含む base64 エンコードされた JSON ペイロードです:

- `url`: ゲートウェイの WebSocket URL（`ws://...` または `wss://...`）
- `bootstrapToken`: 初回ペアリングハンドシェイクに使用される短命の単一デバイスブートストラップトークン

セットアップコードは有効な間はパスワードと同様に扱ってください。

### ノードデバイスを承認する

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
```

### ノードペアリング状態の保存

`~/.openclaw/devices/` 以下に保存されます:

- `pending.json`（短命; 保留中のリクエストは有効期限切れになります）
- `paired.json`（ペアリング済みデバイス + トークン）

### 注意事項

- レガシーの `node.pair.*` API（CLI: `openclaw nodes pending/approve`）は別のゲートウェイ所有のペアリングストアです。WS ノードは引き続きデバイスペアリングが必要です。

## 関連ドキュメント

- セキュリティモデル + プロンプトインジェクション: [セキュリティ](/gateway/security)
- 安全なアップデート（doctor を実行）: [アップデート](/install/updating)
- チャンネル設定:
  - Telegram: [Telegram](/channels/telegram)
  - WhatsApp: [WhatsApp](/channels/whatsapp)
  - Signal: [Signal](/channels/signal)
  - BlueBubbles (iMessage): [BlueBubbles](/channels/bluebubbles)
  - iMessage（レガシー）: [iMessage](/channels/imessage)
  - Discord: [Discord](/channels/discord)
  - Slack: [Slack](/channels/slack)
