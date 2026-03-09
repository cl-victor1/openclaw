---
summary: "Slackのセットアップとランタイム動作（ソケットモード + HTTP Events API）"
read_when:
  - Slackをセットアップするとき、またはSlackソケット/HTTPモードをデバッグするとき
title: "Slack"
---

# Slack

ステータス: SlackアプリインテグレーションによるDMおよびチャンネル向けにプロダクション対応済み。デフォルトモードはソケットモード; HTTP Events APIモードもサポートされています。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    SlackのDMはデフォルトでペアリングモードです。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/tools/slash-commands">
    ネイティブコマンドの動作とコマンドカタログ。
  </Card>
  <Card title="チャンネルトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    チャンネル横断の診断と修復プレイブック。
  </Card>
</CardGroup>

## クイックセットアップ

<Tabs>
  <Tab title="ソケットモード（デフォルト）">
    <Steps>
      <Step title="SlackアプリとトークンをZR作成する">
        Slackアプリ設定で:

        - **ソケットモード**を有効化する
        - `connections:write`を持つ**アプリトークン**（`xapp-...`）を作成する
        - アプリをインストールして**ボットトークン**（`xoxb-...`）をコピーする
      </Step>

      <Step title="OpenClawを設定する">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: "xapp-...",
      botToken: "xoxb-...",
    },
  },
}
```

        環境変数フォールバック（デフォルトアカウントのみ）:

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      </Step>

      <Step title="アプリのイベントを購読する">
        以下のボットイベントを購読する:

        - `app_mention`
        - `message.channels`、`message.groups`、`message.im`、`message.mpim`
        - `reaction_added`、`reaction_removed`
        - `member_joined_channel`、`member_left_channel`
        - `channel_rename`
        - `pin_added`、`pin_removed`

        DMのためにApp Home **Messagesタブ**も有効化する。
      </Step>

      <Step title="ゲートウェイを起動する">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP Events APIモード">
    <Steps>
      <Step title="HTTP用のSlackアプリを設定する">

        - モードをHTTPに設定する（`channels.slack.mode="http"`）
        - Slackの**署名シークレット**をコピーする
        - イベントサブスクリプション + インタラクティビティ + スラッシュコマンドのリクエストURLを同じWebhookパスに設定する（デフォルト`/slack/events`）

      </Step>

      <Step title="OpenClawのHTTPモードを設定する">

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: "xoxb-...",
      signingSecret: "your-signing-secret",
      webhookPath: "/slack/events",
    },
  },
}
```

      </Step>

      <Step title="マルチアカウントHTTPに一意のWebhookパスを使用する">
        アカウントごとのHTTPモードがサポートされています。

        登録が衝突しないよう、各アカウントに別々の`webhookPath`を設定してください。
      </Step>
    </Steps>

  </Tab>
</Tabs>

## トークンモデル

- `botToken` + `appToken`はソケットモードに必要です。
- HTTPモードには`botToken` + `signingSecret`が必要です。
- 設定トークンは環境変数フォールバックをオーバーライドします。
- `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN`の環境変数フォールバックはデフォルトアカウントにのみ適用されます。
- `userToken`（`xoxp-...`）は設定のみ（環境変数フォールバックなし）で、デフォルトで読み取り専用動作（`userTokenReadOnly: true`）になります。
- オプション: アクティブなエージェントアイデンティティ（カスタム`username`とアイコン）を使用して送信メッセージを表示したい場合は`chat:write.customize`を追加します。`icon_emoji`は`:emoji_name:`構文を使用します。

<Tip>
アクション/ディレクトリ読み取りの場合、設定されていればユーザートークンが優先されることがあります。書き込みの場合、ボットトークンが優先されます; ユーザートークンによる書き込みは`userTokenReadOnly: false`でボットトークンが利用できない場合のみ許可されます。
</Tip>

## アクセス制御とルーティング

<Tabs>
  <Tab title="DMポリシー">
    `channels.slack.dmPolicy`はDMアクセスを制御します（レガシー: `channels.slack.dm.policy`）:

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`channels.slack.allowFrom`に`"*"`が必要; レガシー: `channels.slack.dm.allowFrom`）
    - `disabled`

    DMフラグ:

    - `dm.enabled`（デフォルトtrue）
    - `channels.slack.allowFrom`（推奨）
    - `dm.allowFrom`（レガシー）
    - `dm.groupEnabled`（グループDMのデフォルトはfalse）
    - `dm.groupChannels`（オプションのMPIM allowlist）

    マルチアカウントの優先順位:

    - `channels.slack.accounts.default.allowFrom`は`default`アカウントにのみ適用されます。
    - 名前付きアカウントは自身の`allowFrom`が未設定の場合、`channels.slack.allowFrom`を継承します。
    - 名前付きアカウントは`channels.slack.accounts.default.allowFrom`を継承しません。

    DMのペアリングは`openclaw pairing approve slack <code>`を使用します。

  </Tab>

  <Tab title="チャンネルポリシー">
    `channels.slack.groupPolicy`はチャンネル処理を制御します:

    - `open`
    - `allowlist`
    - `disabled`

    チャンネルallowlistは`channels.slack.channels`以下にあります。

    ランタイムノート: `channels.slack`が完全に欠落している場合（環境変数のみのセットアップ）、`channels.defaults.groupPolicy`が設定されていても、ランタイムは`groupPolicy="allowlist"`にフォールバックして警告をログに記録します。

    名前/ID解決:

    - チャンネルallowlistエントリとDM allowlistエントリは、トークンアクセスが許可する場合に起動時に解決されます
    - 未解決のエントリは設定された状態のまま保持されます
    - インバウンド認証マッチングはデフォルトでIDファーストです; 直接のユーザー名/スラッグマッチングには`channels.slack.dangerouslyAllowNameMatching: true`が必要です

  </Tab>

  <Tab title="メンションとチャンネルユーザー">
    チャンネルメッセージはデフォルトでメンションゲーティングされています。

    メンションソース:

    - 明示的なアプリメンション（`<@botId>`）
    - メンション正規表現パターン（`agents.list[].groupChat.mentionPatterns`、フォールバック`messages.groupChat.mentionPatterns`）
    - 暗黙のボットへのスレッド返信動作

    チャンネルごとの制御（`channels.slack.channels.<id|name>`）:

    - `requireMention`
    - `users`（allowlist）
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`、`toolsBySender`
    - `toolsBySender`のキー形式: `id:`、`e164:`、`username:`、`name:`、または`"*"`ワイルドカード
      （レガシーのプレフィックスなしキーは引き続き`id:`のみにマッピング）

  </Tab>
</Tabs>

## コマンドとスラッシュの動作

- Slackでは、ネイティブコマンドの自動モードは**無効**です（`commands.native: "auto"`はSlackのネイティブコマンドを有効化しません）。
- `channels.slack.commands.native: true`（またはグローバル`commands.native: true`）でネイティブSlackコマンドハンドラーを有効化します。
- ネイティブコマンドが有効化されている場合、Slackに対応するスラッシュコマンド（`/<command>`名）を登録します。ただし1つ例外があります:
  - ステータスコマンドには`/agentstatus`を登録します（Slackが`/status`を予約しているため）
- ネイティブコマンドが有効化されていない場合、`channels.slack.slashCommand`で設定された単一のスラッシュコマンドを実行できます。

デフォルトのスラッシュコマンド設定:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

スラッシュセッションは分離されたキーを使用します:

- `agent:<agentId>:slack:slash:<userId>`

## スレッディング、セッション、返信タグ

- DMは`direct`としてルーティング; チャンネルは`channel`; MPIMは`group`。
- デフォルトの`session.dmScope=main`で、Slack DMはエージェントメインセッションに集約されます。
- チャンネルセッション: `agent:<agentId>:slack:channel:<channelId>`。
- スレッド返信は該当する場合にスレッドセッションサフィックス（`:thread:<threadTs>`）を作成できます。
- `channels.slack.thread.historyScope`のデフォルトは`thread`; `thread.inheritParent`のデフォルトは`false`。
- `channels.slack.thread.initialHistoryLimit`は新しいスレッドセッションが開始されたときに取得する既存のスレッドメッセージ数を制御します（デフォルト`20`; `0`で無効）。

返信スレッディングの制御:

- `channels.slack.replyToMode`: `off|first|all`（デフォルト`off`）
- `channels.slack.replyToModeByChatType`: `direct|group|channel`ごとに設定
- ダイレクトチャットのレガシーフォールバック: `channels.slack.dm.replyToMode`

手動返信タグがサポートされています:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

注: `replyToMode="off"`はSlackのすべての返信スレッディングを無効化します。これには明示的な`[[reply_to_*]]`タグも含まれます。これはTelegramとは異なり、Telegramでは`"off"`モードでも明示的なタグは引き続き尊重されます。この違いはプラットフォームのスレッディングモデルを反映しています: Slackのスレッドはメッセージをチャンネルから隠しますが、Telegramの返信はメインチャットフローで表示されたままです。

## メディア、チャンキング、デリバリー

<AccordionGroup>
  <Accordion title="インバウンド添付ファイル">
    SlackのファイルSend添付ファイルはSlackホストのプライベートURL（トークン認証リクエストフロー）からダウンロードされ、フェッチが成功してサイズ制限内の場合にメディアストアに書き込まれます。

    ランタイムのインバウンドサイズキャップのデフォルトは`20MB`で、`channels.slack.mediaMaxMb`でオーバーライドできます。

  </Accordion>

  <Accordion title="アウトバウンドテキストとファイル">
    - テキストチャンクは`channels.slack.textChunkLimit`（デフォルト4000）を使用します
    - `channels.slack.chunkMode="newline"`で段落優先の分割が有効になります
    - ファイル送信はSlackのアップロードAPIを使用し、スレッド返信（`thread_ts`）を含めることができます
    - アウトバウンドのメディアキャップは設定された場合に`channels.slack.mediaMaxMb`に従います; それ以外の場合、チャンネル送信はメディアパイプラインのMIMEタイプのデフォルトを使用します
  </Accordion>

  <Accordion title="デリバリーターゲット">
    推奨される明示的なターゲット:

    - DMには`user:<id>`
    - チャンネルには`channel:<id>`

    ユーザーターゲットに送信する際、Slack DMはSlackのConversation APIで開かれます。

  </Accordion>
</AccordionGroup>

## アクションとゲート

SlackのアクションはState`channels.slack.actions.*`で制御されます。

現在のSlackツールで利用可能なアクショングループ:

| グループ   | デフォルト |
| ---------- | ---------- |
| messages   | 有効       |
| reactions  | 有効       |
| pins       | 有効       |
| memberInfo | 有効       |
| emojiList  | 有効       |

## イベントと運用動作

- メッセージの編集/削除/スレッドブロードキャストはシステムイベントにマッピングされます。
- リアクションの追加/削除イベントはシステムイベントにマッピングされます。
- メンバーの参加/退出、チャンネルの作成/名称変更、ピンの追加/削除イベントはシステムイベントにマッピングされます。
- アシスタントスレッドのステータス更新（スレッド内の「入力中...」インジケーター）は`assistant.threads.setStatus`を使用し、ボットスコープ`assistant:write`が必要です。
- `channel_id_changed`は`configWrites`が有効の場合にチャンネル設定キーを移行できます。
- チャンネルのトピック/目的メタデータは信頼できないコンテキストとして扱われ、ルーティングコンテキストに注入できます。

## Ackリアクション

`ackReaction`はOpenClawが受信メッセージを処理中に確認の絵文字を送信します。

解決順序:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- エージェントのidentityの絵文字フォールバック（`agents.list[].identity.emoji`、それ以外は"👀"）

注:

- Slackはショートコードを期待します（例: `"eyes"`）。
- Slackアカウントまたはグローバルでリアクションを無効化するには`""`を使用します。

## タイピングリアクションフォールバック

`typingReaction`はOpenClawが返信を処理中にインバウンドのSlackメッセージに一時的なリアクションを追加し、実行完了後に削除します。これは特にDMでSlackネイティブのアシスタントタイピングが利用できない場合に便利なフォールバックです。

解決順序:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

注:

- Slackはショートコードを期待します（例: `"hourglass_flowing_sand"`）。
- リアクションはベストエフォートで、返信または失敗パス完了後に自動的にクリーンアップが試みられます。

## マニフェストとスコープチェックリスト

<AccordionGroup>
  <Accordion title="Slackアプリマニフェストの例">

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": {
      "display_name": "OpenClaw",
      "always_online": false
    },
    "app_home": {
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "Send a message to OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "chat:write",
        "channels:history",
        "channels:read",
        "groups:history",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "users:read",
        "app_mentions:read",
        "assistant:write",
        "reactions:read",
        "reactions:write",
        "pins:read",
        "pins:write",
        "emoji:read",
        "commands",
        "files:read",
        "files:write"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "reaction_added",
        "reaction_removed",
        "member_joined_channel",
        "member_left_channel",
        "channel_rename",
        "pin_added",
        "pin_removed"
      ]
    }
  }
}
```

  </Accordion>

  <Accordion title="オプションのユーザートークンスコープ（読み取り操作）">
    `channels.slack.userToken`を設定する場合、典型的な読み取りスコープは:

    - `channels:history`、`groups:history`、`im:history`、`mpim:history`
    - `channels:read`、`groups:read`、`im:read`、`mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read`（Slack検索読み取りに依存する場合）

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="チャンネルで返信なし">
    この順序で確認する:

    - `groupPolicy`
    - チャンネルallowlist（`channels.slack.channels`）
    - `requireMention`
    - チャンネルごとの`users` allowlist

    便利なコマンド:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DMメッセージが無視される">
    確認する:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy`（またはレガシー`channels.slack.dm.policy`）
    - ペアリング承認 / allowlistエントリ

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="ソケットモードが接続しない">
    Slackアプリ設定でボット + アプリトークンとソケットモードの有効化を確認してください。
  </Accordion>

  <Accordion title="HTTPモードがイベントを受信しない">
    確認する:

    - 署名シークレット
    - Webhookパス
    - SlackのリクエストURL（イベント + インタラクティビティ + スラッシュコマンド）
    - HTTPアカウントごとの一意の`webhookPath`

  </Accordion>

  <Accordion title="ネイティブ/スラッシュコマンドが起動しない">
    意図した設定を確認する:

    - ネイティブコマンドモード（`channels.slack.commands.native: true`）とSlackに登録された対応するスラッシュコマンド
    - または単一スラッシュコマンドモード（`channels.slack.slashCommand.enabled: true`）

    また`commands.useAccessGroups`とチャンネル/ユーザーallowlistを確認してください。

  </Accordion>
</AccordionGroup>

## テキストストリーミング

OpenClawはAgents and AI Apps API経由でSlackのネイティブテキストストリーミングをサポートします。

`channels.slack.streaming`はライブプレビュー動作を制御します:

- `off`: ライブプレビューストリーミングを無効化。
- `partial`（デフォルト）: プレビューテキストを最新の部分的な出力で置き換える。
- `block`: チャンク化されたプレビュー更新を追加する。
- `progress`: 生成中に進捗ステータステキストを表示し、最終テキストを送信する。

`channels.slack.nativeStreaming`は`streaming`が`partial`の場合にSlackのネイティブストリーミングAPI（`chat.startStream` / `chat.appendStream` / `chat.stopStream`）を制御します（デフォルト: `true`）。

ネイティブSlackストリーミングを無効化する（ドラフトプレビュー動作を維持）:

```yaml
channels:
  slack:
    streaming: partial
    nativeStreaming: false
```

レガシーキー:

- `channels.slack.streamMode`（`replace | status_final | append`）は`channels.slack.streaming`に自動移行されます。
- boolean `channels.slack.streaming`は`channels.slack.nativeStreaming`に自動移行されます。

### 要件

1. Slackアプリ設定で**Agents and AI Apps**を有効化する。
2. アプリが`assistant:write`スコープを持っていることを確認する。
3. そのメッセージに返信スレッドが利用可能である必要があります。スレッド選択は引き続き`replyToMode`に従います。

### 動作

- 最初のテキストチャンクがストリームを開始します（`chat.startStream`）。
- 後続のテキストチャンクが同じストリームに追加されます（`chat.appendStream`）。
- 返信の終了がストリームを確定します（`chat.stopStream`）。
- メディアと非テキストペイロードは通常のデリバリーにフォールバックします。
- ストリーミングが返信の途中で失敗した場合、OpenClawは残りのペイロードの通常のデリバリーにフォールバックします。

## 設定リファレンスポインター

主要リファレンス:

- [設定リファレンス - Slack](/gateway/configuration-reference#slack)

  重要なSlackフィールド:
  - モード/認証: `mode`、`botToken`、`appToken`、`signingSecret`、`webhookPath`、`accounts.*`
  - DMアクセス: `dm.enabled`、`dmPolicy`、`allowFrom`（レガシー: `dm.policy`、`dm.allowFrom`）、`dm.groupEnabled`、`dm.groupChannels`
  - 互換性トグル: `dangerouslyAllowNameMatching`（緊急手段; 必要でない限りオフに保つ）
  - チャンネルアクセス: `groupPolicy`、`channels.*`、`channels.*.users`、`channels.*.requireMention`
  - スレッディング/履歴: `replyToMode`、`replyToModeByChatType`、`thread.*`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
  - デリバリー: `textChunkLimit`、`chunkMode`、`mediaMaxMb`、`streaming`、`nativeStreaming`
  - 運用/機能: `configWrites`、`commands.native`、`slashCommand.*`、`actions.*`、`userToken`、`userTokenReadOnly`

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [トラブルシューティング](/channels/troubleshooting)
- [設定](/gateway/configuration)
- [スラッシュコマンド](/tools/slash-commands)
