---
summary: "Synology Chat のウェブフック設定と OpenClaw の構成"
read_when:
  - Synology Chat を OpenClaw と連携させる場合
  - Synology Chat のウェブフックルーティングをデバッグする場合
title: "Synology Chat"
---

# Synology Chat (プラグイン)

ステータス: Synology Chat ウェブフックを使用したダイレクトメッセージチャンネルとして、プラグイン経由でサポートされています。
このプラグインは Synology Chat の送信ウェブフックからの受信メッセージを受け付け、
Synology Chat の受信ウェブフックを通じて返信を送信します。

## プラグインが必要です

Synology Chat はプラグインベースであり、デフォルトのコアチャンネルインストールには含まれていません。

ローカルチェックアウトからインストールする場合:

```bash
openclaw plugins install ./extensions/synology-chat
```

詳細: [プラグイン](/tools/plugin)

## クイックセットアップ

1. Synology Chat プラグインをインストールして有効化します。
2. Synology Chat のインテグレーション設定で以下を行います:
   - 受信ウェブフックを作成し、その URL をコピーします。
   - シークレットトークンを使用した送信ウェブフックを作成します。
3. 送信ウェブフックの URL を OpenClaw ゲートウェイに設定します:
   - デフォルトは `https://gateway-host/webhook/synology` です。
   - またはカスタムの `channels.synology-chat.webhookPath` を使用してください。
4. OpenClaw で `channels.synology-chat` を設定します。
5. ゲートウェイを再起動し、Synology Chat ボットに DM を送信します。

最小限の設定:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## 環境変数

デフォルトアカウントでは、以下の環境変数を使用できます:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (カンマ区切り)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

設定値は環境変数より優先されます。

## DM ポリシーとアクセス制御

- `dmPolicy: "allowlist"` が推奨デフォルトです。
- `allowedUserIds` には Synology ユーザー ID のリスト（またはカンマ区切り文字列）を指定します。
- `allowlist` モードで `allowedUserIds` が空の場合、設定ミスとみなされ、ウェブフックルートは起動しません（すべてを許可する場合は `dmPolicy: "open"` を使用してください）。
- `dmPolicy: "open"` は送信者を問わず許可します。
- `dmPolicy: "disabled"` は DM をブロックします。
- ペアリング承認には以下を使用します:
  - `openclaw pairing list synology-chat`
  - `openclaw pairing approve synology-chat <CODE>`

## 送信配信

送信先として Synology Chat のユーザー ID（数値）を使用します。

例:

```bash
openclaw message send --channel synology-chat --target 123456 --text "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --text "Hello again"
```

メディアの送信は URL ベースのファイル配信でサポートされています。

## マルチアカウント

複数の Synology Chat アカウントは `channels.synology-chat.accounts` でサポートされています。
各アカウントは、トークン、受信 URL、ウェブフックパス、DM ポリシー、制限をオーバーライドできます。

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## セキュリティに関する注意事項

- `token` は秘密に保ち、漏洩した場合はローテーションしてください。
- ローカル NAS の自己署名証明書を明示的に信頼する場合を除き、`allowInsecureSsl: false` を維持してください。
- 受信ウェブフックのリクエストはトークン検証され、送信者ごとにレート制限されます。
- 本番環境では `dmPolicy: "allowlist"` を推奨します。
