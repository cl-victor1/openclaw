---
summary: "Nextcloud Talk のサポート状況、機能、および設定"
read_when:
  - Nextcloud Talk チャンネルの機能に関する作業時
title: "Nextcloud Talk"
---

# Nextcloud Talk (プラグイン)

ステータス: プラグイン経由でサポート（Webhook ボット）。ダイレクトメッセージ、ルーム、リアクション、マークダウンメッセージに対応しています。

## プラグインが必要です

Nextcloud Talk はプラグインとして提供されており、コアインストールには含まれていません。

CLI 経由でインストール（npm レジストリ）:

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

ローカルチェックアウト（git リポジトリから実行する場合）:

```bash
openclaw plugins install ./extensions/nextcloud-talk
```

configure/オンボーディング時に Nextcloud Talk を選択し、git チェックアウトが検出された場合、
OpenClaw はローカルインストールパスを自動的に提案します。

詳細: [プラグイン](/tools/plugin)

## クイックセットアップ（初心者向け）

1. Nextcloud Talk プラグインをインストールします。
2. Nextcloud サーバーでボットを作成します:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature reaction
   ```

3. 対象ルームの設定でボットを有効にします。
4. OpenClaw を設定します:
   - 設定: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - または環境変数: `NEXTCLOUD_TALK_BOT_SECRET`（デフォルトアカウントのみ）
5. ゲートウェイを再起動します（またはオンボーディングを完了します）。

最小設定:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## 注意事項

- ボットは DM を開始できません。ユーザーが先にボットにメッセージを送る必要があります。
- Webhook URL はゲートウェイから到達可能でなければなりません。プロキシの背後にある場合は `webhookPublicUrl` を設定してください。
- メディアのアップロードはボット API ではサポートされていません。メディアは URL として送信されます。
- Webhook ペイロードは DM とルームを区別しません。ルームタイプの検索を有効にするには `apiUser` + `apiPassword` を設定してください（未設定の場合、DM はルームとして扱われます）。

## アクセス制御（DM）

- デフォルト: `channels.nextcloud-talk.dmPolicy = "pairing"`. 未知の送信者にはペアリングコードが送られます。
- 承認方法:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- パブリック DM: `channels.nextcloud-talk.dmPolicy="open"` および `channels.nextcloud-talk.allowFrom=["*"]`。
- `allowFrom` は Nextcloud ユーザー ID のみに対応します。表示名は無視されます。

## ルーム（グループ）

- デフォルト: `channels.nextcloud-talk.groupPolicy = "allowlist"`（メンション制限付き）。
- `channels.nextcloud-talk.rooms` でルームを許可リストに追加します:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- ルームを許可しない場合は、許可リストを空にするか `channels.nextcloud-talk.groupPolicy="disabled"` を設定してください。

## 機能

| 機能             | ステータス        |
| ---------------- | ----------------- |
| ダイレクトメッセージ | サポート中     |
| ルーム           | サポート中        |
| スレッド         | 未サポート        |
| メディア         | URL のみ          |
| リアクション     | サポート中        |
| ネイティブコマンド | 未サポート      |

## 設定リファレンス（Nextcloud Talk）

完全な設定: [設定](/gateway/configuration)

プロバイダーオプション:

- `channels.nextcloud-talk.enabled`: チャンネルの起動を有効/無効にします。
- `channels.nextcloud-talk.baseUrl`: Nextcloud インスタンスの URL。
- `channels.nextcloud-talk.botSecret`: ボットの共有シークレット。
- `channels.nextcloud-talk.botSecretFile`: 通常ファイルのシークレットパス。シンボリックリンクは拒否されます。
- `channels.nextcloud-talk.apiUser`: ルーム検索用の API ユーザー（DM 検出）。
- `channels.nextcloud-talk.apiPassword`: ルーム検索用の API/アプリパスワード。
- `channels.nextcloud-talk.apiPasswordFile`: API パスワードファイルのパス。
- `channels.nextcloud-talk.webhookPort`: Webhook リスナーポート（デフォルト: 8788）。
- `channels.nextcloud-talk.webhookHost`: Webhook ホスト（デフォルト: 0.0.0.0）。
- `channels.nextcloud-talk.webhookPath`: Webhook パス（デフォルト: /nextcloud-talk-webhook）。
- `channels.nextcloud-talk.webhookPublicUrl`: 外部からアクセス可能な Webhook URL。
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled`。
- `channels.nextcloud-talk.allowFrom`: DM 許可リスト（ユーザー ID）。`open` には `"*"` が必要です。
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled`。
- `channels.nextcloud-talk.groupAllowFrom`: グループ許可リスト（ユーザー ID）。
- `channels.nextcloud-talk.rooms`: ルームごとの設定と許可リスト。
- `channels.nextcloud-talk.historyLimit`: グループ履歴の上限（0 で無効）。
- `channels.nextcloud-talk.dmHistoryLimit`: DM 履歴の上限（0 で無効）。
- `channels.nextcloud-talk.dms`: DM ごとのオーバーライド（historyLimit）。
- `channels.nextcloud-talk.textChunkLimit`: 送信テキストのチャンクサイズ（文字数）。
- `channels.nextcloud-talk.chunkMode`: `length`（デフォルト）または空行（段落境界）で分割してから長さチャンクに分割する `newline`。
- `channels.nextcloud-talk.blockStreaming`: このチャンネルのブロックストリーミングを無効にします。
- `channels.nextcloud-talk.blockStreamingCoalesce`: ブロックストリーミングの結合チューニング。
- `channels.nextcloud-talk.mediaMaxMb`: 受信メディアの上限（MB）。
