---
summary: "LINE Messaging API プラグインのセットアップ、設定、および使用方法"
read_when:
  - OpenClaw を LINE に接続したい場合
  - LINE のウェブフックと認証情報のセットアップが必要な場合
  - LINE 固有のメッセージオプションが必要な場合
title: LINE
---

# LINE (プラグイン)

LINE は LINE Messaging API を通じて OpenClaw に接続します。このプラグインはゲートウェイ上でウェブフック受信機として動作し、チャンネルアクセストークンとチャンネルシークレットを認証に使用します。

ステータス: プラグイン経由でサポートされています。ダイレクトメッセージ、グループチャット、メディア、位置情報、Flex メッセージ、テンプレートメッセージ、クイックリプライをサポートしています。リアクションとスレッドはサポートされていません。

## プラグインが必要です

LINE プラグインをインストールする場合:

```bash
openclaw plugins install @openclaw/line
```

ローカルチェックアウト（git リポジトリから実行する場合）:

```bash
openclaw plugins install ./extensions/line
```

## セットアップ

1. LINE Developers アカウントを作成し、コンソールを開きます:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. プロバイダーを作成（または選択）し、**Messaging API** チャンネルを追加します。
3. チャンネル設定から**チャンネルアクセストークン**と**チャンネルシークレット**をコピーします。
4. Messaging API 設定で**Webhook の利用**を有効化します。
5. ウェブフック URL をゲートウェイエンドポイントに設定します（HTTPS 必須）:

```
https://gateway-host/line/webhook
```

ゲートウェイは LINE のウェブフック検証（GET）と受信イベント（POST）に応答します。
カスタムパスが必要な場合は、`channels.line.webhookPath` または
`channels.line.accounts.<id>.webhookPath` を設定し、URL を更新してください。

セキュリティに関する注意:

- LINE の署名検証はボディ依存（生ボディの HMAC）であるため、OpenClaw は検証前に厳密な事前認証ボディ制限とタイムアウトを適用します。

## 設定

最小限の設定:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

環境変数（デフォルトアカウントのみ）:

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

トークン/シークレットファイル:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` と `secretFile` は通常ファイルを指定する必要があります。シンボリックリンクは拒否されます。

マルチアカウント:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## アクセス制御

ダイレクトメッセージはデフォルトでペアリングが必要です。未知の送信者にはペアリングコードが発行され、承認されるまでメッセージは無視されます。

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

許可リストとポリシー:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled`
- `channels.line.allowFrom`: DM 用の許可された LINE ユーザー ID
- `channels.line.groupPolicy`: `allowlist | open | disabled`
- `channels.line.groupAllowFrom`: グループ用の許可された LINE ユーザー ID
- グループごとのオーバーライド: `channels.line.groups.<groupId>.allowFrom`
- ランタイムの注意: `channels.line` が完全に欠落している場合、ランタイムはグループチェックに `groupPolicy="allowlist"` をフォールバックとして使用します（`channels.defaults.groupPolicy` が設定されていても）。

LINE ID は大文字と小文字を区別します。有効な ID の形式:

- ユーザー: `U` + 32 桁の 16 進数
- グループ: `C` + 32 桁の 16 進数
- ルーム: `R` + 32 桁の 16 進数

## メッセージの動作

- テキストは 5000 文字でチャンクされます。
- マークダウン書式は削除され、コードブロックとテーブルは可能な限り Flex カードに変換されます。
- ストリーミング応答はバッファリングされ、エージェントが処理中はローディングアニメーションとともに完全なチャンクが LINE に届きます。
- メディアのダウンロードは `channels.line.mediaMaxMb`（デフォルト 10）で制限されます。

## チャンネルデータ（リッチメッセージ）

`channelData.line` を使用して、クイックリプライ、位置情報、Flex カード、またはテンプレートメッセージを送信します。

```json5
{
  text: "Here you go",
  channelData: {
    line: {
      quickReplies: ["Status", "Help"],
      location: {
        title: "Office",
        address: "123 Main St",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "Status card",
        contents: {
          /* Flex payload */
        },
      },
      templateMessage: {
        type: "confirm",
        text: "Proceed?",
        confirmLabel: "Yes",
        confirmData: "yes",
        cancelLabel: "No",
        cancelData: "no",
      },
    },
  },
}
```

LINE プラグインには Flex メッセージプリセット用の `/card` コマンドも含まれています:

```
/card info "Welcome" "Thanks for joining!"
```

## トラブルシューティング

- **ウェブフック検証が失敗する:** ウェブフック URL が HTTPS であり、`channelSecret` が LINE コンソールと一致することを確認してください。
- **受信イベントがない:** ウェブフックパスが `channels.line.webhookPath` と一致し、ゲートウェイが LINE からアクセス可能であることを確認してください。
- **メディアダウンロードエラー:** メディアがデフォルト制限を超える場合は `channels.line.mediaMaxMb` を増やしてください。
