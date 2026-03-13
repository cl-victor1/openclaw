---
summary: "Tlon/Urbitのサポート状況、機能、および設定"
read_when:
  - Tlon/Urbitチャンネル機能の開発時
title: "Tlon"
---

# Tlon（プラグイン）

TlonはUrbit上に構築された分散型メッセンジャーです。OpenClawはUrbitシップに接続し、DMおよびグループチャットメッセージに応答できます。グループへの返信はデフォルトで@メンションが必要であり、アローリストによってさらに制限することができます。

ステータス：プラグインでサポートされています。DM、グループメンション、スレッド返信、リッチテキストフォーマット、画像アップロードをサポートしています。リアクションとポーリングはまだサポートされていません。

## プラグインが必要です

TlonはプラグインとしてShipされており、コアインストールにはバンドルされていません。

CLI経由でインストール（npmレジストリ）：

```bash
openclaw plugins install @openclaw/tlon
```

ローカルチェックアウト（gitリポジトリから実行する場合）：

```bash
openclaw plugins install ./extensions/tlon
```

詳細：[Plugins](/tools/plugin)

## セットアップ

1. Tlonプラグインをインストールします。
2. シップURLとログインコードを取得します。
3. `channels.tlon`を設定します。
4. ゲートウェイを再起動します。
5. ボットにDMを送るか、グループチャンネルでメンションします。

最小限の設定（シングルアカウント）：

```json5
{
  channels: {
    tlon: {
      enabled: true,
      ship: "~sampel-palnet",
      url: "https://your-ship-host",
      code: "lidlut-tabwed-pillex-ridrup",
      ownerShip: "~your-main-ship", // 推奨：自分のシップ、常に許可される
    },
  },
}
```

## プライベート/LANシップ

デフォルトでは、OpenClawはSSRF保護のためにプライベート/内部ホスト名とIPレンジをブロックします。
シップがプライベートネットワーク（localhost、LAN IP、または内部ホスト名）で動作している場合は、明示的にオプトインする必要があります：

```json5
{
  channels: {
    tlon: {
      url: "http://localhost:8080",
      allowPrivateNetwork: true,
    },
  },
}
```

以下のようなURLに適用されます：

- `http://localhost:8080`
- `http://192.168.x.x:8080`
- `http://my-ship.local:8080`

⚠️ ローカルネットワークを信頼できる場合のみ有効にしてください。この設定は、シップURLへのリクエストのSSRF保護を無効にします。

## グループチャンネル

自動検出はデフォルトで有効です。チャンネルを手動でピン留めすることもできます：

```json5
{
  channels: {
    tlon: {
      groupChannels: ["chat/~host-ship/general", "chat/~host-ship/support"],
    },
  },
}
```

自動検出を無効にする：

```json5
{
  channels: {
    tlon: {
      autoDiscoverChannels: false,
    },
  },
}
```

## アクセス制御

DMアローリスト（空の場合 = DMは許可されない。`ownerShip`を承認フローに使用）：

```json5
{
  channels: {
    tlon: {
      dmAllowlist: ["~zod", "~nec"],
    },
  },
}
```

グループ認証（デフォルトで制限あり）：

```json5
{
  channels: {
    tlon: {
      defaultAuthorizedShips: ["~zod"],
      authorization: {
        channelRules: {
          "chat/~host-ship/general": {
            mode: "restricted",
            allowedShips: ["~zod", "~nec"],
          },
          "chat/~host-ship/announcements": {
            mode: "open",
          },
        },
      },
    },
  },
}
```

## オーナーと承認システム

未承認ユーザーが操作しようとしたときに承認リクエストを受け取るオーナーシップを設定します：

```json5
{
  channels: {
    tlon: {
      ownerShip: "~your-main-ship",
    },
  },
}
```

オーナーシップは**すべての場所で自動的に承認**されます。DM招待は自動で承認され、チャンネルメッセージは常に許可されます。オーナーを`dmAllowlist`や`defaultAuthorizedShips`に追加する必要はありません。

設定すると、オーナーは以下のDM通知を受け取ります：

- アローリストにないシップからのDMリクエスト
- 認証なしのチャンネルでのメンション
- グループ招待リクエスト

## 自動承認設定

DM招待の自動承認（dmAllowlistのシップ向け）：

```json5
{
  channels: {
    tlon: {
      autoAcceptDmInvites: true,
    },
  },
}
```

グループ招待の自動承認：

```json5
{
  channels: {
    tlon: {
      autoAcceptGroupInvites: true,
    },
  },
}
```

## 配信ターゲット（CLI/cron）

`openclaw message send`またはcron配信で使用します：

- DM：`~sampel-palnet`または`dm/~sampel-palnet`
- グループ：`chat/~host-ship/channel`または`group:~host-ship/channel`

## バンドルスキル

Tlonプラグインにはバンドルスキル（[`@tloncorp/tlon-skill`](https://github.com/tloncorp/tlon-skill)）が含まれており、Tlon操作へのCLIアクセスを提供します：

- **コンタクト**：プロフィールの取得/更新、コンタクトリスト
- **チャンネル**：リスト、作成、メッセージ投稿、履歴取得
- **グループ**：リスト、作成、メンバー管理
- **DM**：メッセージ送信、メッセージへのリアクション
- **リアクション**：投稿とDMへの絵文字リアクションの追加/削除
- **設定**：スラッシュコマンドによるプラグイン権限の管理

プラグインをインストールすると、スキルは自動的に利用可能になります。

## 機能一覧

| 機能             | ステータス                                    |
| --------------- | -------------------------------------------- |
| ダイレクトメッセージ | ✅ サポート済み                              |
| グループ/チャンネル  | ✅ サポート済み（デフォルトでメンション必須）  |
| スレッド          | ✅ サポート済み（スレッド内で自動返信）       |
| リッチテキスト     | ✅ MarkdownをTlonフォーマットに変換          |
| 画像             | ✅ Tlonストレージにアップロード              |
| リアクション       | ✅ [バンドルスキル](#バンドルスキル)経由     |
| ポーリング        | ❌ 未サポート                                |
| ネイティブコマンド  | ✅ サポート済み（デフォルトでオーナーのみ）  |

## トラブルシューティング

まずこのコマンドを順番に実行してください：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
```

よくある問題：

- **DMが無視される**：送信者が`dmAllowlist`にいないか、承認フローのための`ownerShip`が設定されていない。
- **グループメッセージが無視される**：チャンネルが検出されていないか、送信者が承認されていない。
- **接続エラー**：シップURLにアクセスできるか確認してください。ローカルシップの場合は`allowPrivateNetwork`を有効にしてください。
- **認証エラー**：ログインコードが最新かどうか確認してください（コードはローテーションされます）。

## 設定リファレンス

完全な設定：[Configuration](/gateway/configuration)

プロバイダーオプション：

- `channels.tlon.enabled`：チャンネルの起動を有効/無効にします。
- `channels.tlon.ship`：ボットのUrbitシップ名（例：`~sampel-palnet`）。
- `channels.tlon.url`：シップURL（例：`https://sampel-palnet.tlon.network`）。
- `channels.tlon.code`：シップのログインコード。
- `channels.tlon.allowPrivateNetwork`：localhost/LAN URLを許可（SSRFバイパス）。
- `channels.tlon.ownerShip`：承認システムのオーナーシップ（常に承認済み）。
- `channels.tlon.dmAllowlist`：DMを許可するシップ（空 = なし）。
- `channels.tlon.autoAcceptDmInvites`：アローリストのシップからのDMを自動承認。
- `channels.tlon.autoAcceptGroupInvites`：すべてのグループ招待を自動承認。
- `channels.tlon.autoDiscoverChannels`：グループチャンネルを自動検出（デフォルト：true）。
- `channels.tlon.groupChannels`：手動でピン留めしたチャンネルネスト。
- `channels.tlon.defaultAuthorizedShips`：すべてのチャンネルで承認されたシップ。
- `channels.tlon.authorization.channelRules`：チャンネルごとの認証ルール。
- `channels.tlon.showModelSignature`：メッセージにモデル名を追記。

## 注意事項

- グループ返信には（例：`~your-bot-ship`のような）メンションが必要です。
- スレッド返信：受信メッセージがスレッド内の場合、OpenClawはスレッド内で返信します。
- リッチテキスト：Markdownフォーマット（太字、斜体、コード、ヘッダー、リスト）はTlonのネイティブフォーマットに変換されます。
- 画像：URLはTlonストレージにアップロードされ、画像ブロックとして埋め込まれます。
