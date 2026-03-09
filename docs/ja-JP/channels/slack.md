---
summary: "Slackのセットアップとランタイム動作（ソケットモード + HTTP Events API）"
read_when:
  - Slackのセットアップまたはソケット/HTTPモードのデバッグ
title: "Slack"
---

# Slack

ステータス：SlackアプリインテグレーションによるDM + チャンネルへの対応が本番環境で利用可能。デフォルトモードはソケットモード；HTTP Events APIモードもサポートされています。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    Slack DMはデフォルトでペアリングモードです。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/tools/slash-commands">
    ネイティブコマンドの動作とコマンドカタログ。
  </Card>
  <Card title="チャンネルトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    クロスチャンネルの診断と修復プレイブック。
  </Card>
</CardGroup>

## クイックセットアップ

<Tabs>
  <Tab title="ソケットモード（デフォルト）">
    <Steps>
      <Step title="SlackアプリとトークンN作成">
        Slackアプリ設定で：

        - **ソケットモード**を有効化
        - `connections:write` スコープで**アプリトークン**（`xapp-...`）を作成
        - アプリをインストールして**ボットトークン**（`xoxb-...`）をコピー
      </Step>

      <Step title="OpenClawの設定">

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

        環境変数フォールバック（デフォルトアカウントのみ）：

```bash
SLACK_APP_TOKEN=xapp-...
SLACK_BOT_TOKEN=xoxb-...
```

      </Step>

      <Step title="アプリイベントのサブスクライブ">
        以下のボットイベントをサブスクライブ：

        - `app_mention`
        - `message.channels`、`message.groups`、`message.im`、`message.mpim`
        - `reaction_added`、`reaction_removed`
        - `member_joined_channel`、`member_left_channel`
        - `channel_rename`
        - `pin_added`、`pin_removed`

        DMのためにApp Homeの**メッセージタブ**も有効化してください。
      </Step>

      <Step title="Gatewayの起動">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="HTTP Events APIモード">
    <Steps>
      <Step title="HTTP用のSlackアプリの設定">

        - モードをHTTPに設定（`channels.slack.mode="http"`）
        - Slackの**署名シークレット**をコピー
        - イベントサブスクリプション + インタラクティビティ + スラッシュコマンドのリクエストURLを同じwebhookパスに設定（デフォルト `/slack/events`）

      </Step>

      <Step title="OpenClawのHTTPモード設定">

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

      <Step title="マルチアカウントHTTPに一意のwebhookパスを使用">
        アカウントごとのHTTPモードがサポートされています。

        登録が衝突しないよう、各アカウントに異なる `webhookPath` を設定してください。
      </Step>
    </Steps>

  </Tab>
</Tabs>

## トークンモデル

- ソケットモードには `botToken` + `appToken` が必要です。
- HTTPモードには `botToken` + `signingSecret` が必要です。
- 設定ファイルのトークンは環境変数フォールバックを上書きします。
- `SLACK_BOT_TOKEN` / `SLACK_APP_TOKEN` の環境変数フォールバックはデフォルトアカウントのみに適用されます。
- `userToken`（`xoxp-...`）は設定ファイルのみ（環境変数フォールバックなし）で、デフォルトで読み取り専用動作（`userTokenReadOnly: true`）。
- オプション：アクティブなエージェントIDを使用した発信メッセージ（カスタム `username` とアイコン）が必要な場合は `chat:write.customize` を追加。`icon_emoji` は `:emoji_name:` 構文を使用。

<Tip>
アクション/ディレクトリ読み取りでは、設定されている場合にユーザートークンが優先される場合があります。書き込みではボットトークンが優先；ユーザートークンによる書き込みは `userTokenReadOnly: false` でボットトークンが利用できない場合のみ許可されます。
</Tip>

## アクセス制御とルーティング

<Tabs>
  <Tab title="DMポリシー">
    `channels.slack.dmPolicy` はDMアクセスを制御します（レガシー：`channels.slack.dm.policy`）：

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`channels.slack.allowFrom` に `"*"` が必要；レガシー：`channels.slack.dm.allowFrom`）
    - `disabled`

    DMフラグ：

    - `dm.enabled`（デフォルトtrue）
    - `channels.slack.allowFrom`（推奨）
    - `dm.allowFrom`（レガシー）
    - `dm.groupEnabled`（グループDMのデフォルトはfalse）
    - `dm.groupChannels`（オプションのMPIM許可リスト）

    マルチアカウント優先順位：

    - `channels.slack.accounts.default.allowFrom` は `default` アカウントのみに適用。
    - 名前付きアカウントは自身の `allowFrom` が未設定の場合 `channels.slack.allowFrom` を継承。
    - 名前付きアカウントは `channels.slack.accounts.default.allowFrom` を継承しません。

    DMのペアリングは `openclaw pairing approve slack <code>` を使用します。

  </Tab>

  <Tab title="チャンネルポリシー">
    `channels.slack.groupPolicy` はチャンネル処理を制御します：

    - `open`
    - `allowlist`
    - `disabled`

    チャンネル許可リストは `channels.slack.channels` にあります。

    ランタイム注記：`channels.slack` が完全に欠落している場合（環境変数のみのセットアップ）、ランタイムは `groupPolicy="allowlist"` にフォールバックして警告をログ出力します（`channels.defaults.groupPolicy` が設定されていても）。

    名前/ID解決：

    - チャンネル許可リストエントリとDM許可リストエントリはトークンアクセスが許可される場合、起動時に解決されます
    - 未解決のエントリは設定通りに保持されます
    - 受信認可マッチングはデフォルトでID優先；ユーザー名/スラッグの直接マッチングには `channels.slack.dangerouslyAllowNameMatching: true` が必要

  </Tab>

  <Tab title="メンションとチャンネルユーザー">
    チャンネルメッセージはデフォルトでメンションゲートされます。

    メンションソース：

    - 明示的なアプリメンション（`<@botId>`）
    - メンション正規表現パターン（`agents.list[].groupChat.mentionPatterns`、フォールバック `messages.groupChat.mentionPatterns`）
    - 暗黙的なボットスレッドへの返信動作

    チャンネルごとの制御（`channels.slack.channels.<id|name>`）：

    - `requireMention`
    - `users`（許可リスト）
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`、`toolsBySender`
    - `toolsBySender` のキー形式：`id:`、`e164:`、`username:`、`name:`、または `"*"` ワイルドカード
      （レガシーのプレフィックスなしキーは `id:` のみにマッピングされます）

  </Tab>
</Tabs>

## コマンドとスラッシュ動作

- ネイティブコマンドの自動モードはSlackでは**オフ**です（`commands.native: "auto"` ではSlackのネイティブコマンドは有効化されません）。
- `channels.slack.commands.native: true`（またはグローバル `commands.native: true`）でネイティブSlackコマンドハンドラーを有効化。
- ネイティブコマンドが有効な場合、Slackで対応するスラッシュコマンド（`/<command>` 名）を登録してください。ただし1つ例外があります：
  - ステータスコマンドには `/agentstatus` を登録（Slackは `/status` を予約済み）
- ネイティブコマンドが有効でない場合、`channels.slack.slashCommand` で設定された単一のスラッシュコマンドを使用できます。
- ネイティブ引数メニューはレンダリング戦略を適応させます：
  - 5つ以下のオプション：ボタンブロック
  - 6〜100のオプション：静的セレクトメニュー
  - 100を超えるオプション：インタラクティビティオプションハンドラーが利用可能な場合は非同期オプションフィルタリングを持つ外部セレクト
  - エンコードされたオプション値がSlackの制限を超える場合、フローはボタンにフォールバック
- 長いオプションペイロードの場合、スラッシュコマンド引数メニューは選択値をディスパッチする前に確認ダイアログを使用します。

デフォルトのスラッシュコマンド設定：

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

スラッシュセッションは分離されたキーを使用します：

- `agent:<agentId>:slack:slash:<userId>`

ターゲット会話セッション（`CommandTargetSessionKey`）に対してコマンド実行をルーティングします。

## スレッド、セッション、返信タグ

- DMは `direct` としてルーティング；チャンネルは `channel`；MPIMは `group`。
- デフォルトの `session.dmScope=main` では、Slack DMはエージェントのメインセッションに集約されます。
- チャンネルセッション：`agent:<agentId>:slack:channel:<channelId>`。
- スレッド返信は該当する場合にスレッドセッションサフィックス（`:thread:<threadTs>`）を作成できます。
- `channels.slack.thread.historyScope` のデフォルトは `thread`；`thread.inheritParent` のデフォルトは `false`。
- `channels.slack.thread.initialHistoryLimit` は新しいスレッドセッション開始時に取得する既存スレッドメッセージの数を制御します（デフォルト `20`；`0` で無効化）。

返信スレッド制御：

- `channels.slack.replyToMode`: `off|first|all`（デフォルト `off`）
- `channels.slack.replyToModeByChatType`: `direct|group|channel` ごと
- ダイレクトチャットのレガシーフォールバック：`channels.slack.dm.replyToMode`

手動返信タグがサポートされています：

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

注記：`replyToMode="off"` は明示的な `[[reply_to_*]]` タグを含むSlackの**すべての**返信スレッドを無効化します。これはTelegramとは異なり、Telegramでは `"off"` モードでも明示的なタグは有効です。この違いはプラットフォームのスレッドモデルを反映しています：Slackのスレッドはチャンネルからメッセージを隠しますが、Telegramの返信はメインチャットフローに表示されます。

## メディア、チャンク、配信

<AccordionGroup>
  <Accordion title="受信添付ファイル">
    Slackファイル添付ファイルは、SlackホストのプライベートURL（トークン認証リクエストフロー）からダウンロードされ、フェッチが成功してサイズ制限内であればメディアストアに書き込まれます。

    ランタイムの受信サイズ上限は、`channels.slack.mediaMaxMb` で上書きされない限りデフォルトで `20MB` です。

  </Accordion>

  <Accordion title="送信テキストとファイル">
    - テキストチャンクは `channels.slack.textChunkLimit` を使用（デフォルト4000）
    - `channels.slack.chunkMode="newline"` で段落優先分割を有効化
    - ファイル送信はSlackアップロードAPIを使用し、スレッド返信（`thread_ts`）を含むことができます
    - 送信メディア上限は `channels.slack.mediaMaxMb` が設定されている場合はそれに従い、そうでない場合はメディアパイプラインのMIME種別デフォルトを使用
  </Accordion>

  <Accordion title="配信ターゲット">
    推奨される明示的なターゲット：

    - DMには `user:<id>`
    - チャンネルには `channel:<id>`

    Slack DMは、ユーザーターゲットへの送信時にSlack会話APIで開かれます。

  </Accordion>
</AccordionGroup>

## アクションとゲート

Slackアクションは `channels.slack.actions.*` で制御されます。

現在のSlackツールで利用可能なアクショングループ：

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
- メンバーの参加/退出、チャンネルの作成/名前変更、ピンの追加/削除イベントはシステムイベントにマッピングされます。
- アシスタントスレッドステータスの更新（スレッドの「入力中...」インジケーター用）は `assistant.threads.setStatus` を使用し、ボットスコープ `assistant:write` が必要です。
- `channel_id_changed` は `configWrites` が有効な場合にチャンネル設定キーを移行できます。
- チャンネルのトピック/目的メタデータは信頼されていないコンテキストとして扱われ、ルーティングコンテキストに注入できます。
- ブロックアクションとモーダルインタラクションは、リッチなペイロードフィールドを持つ構造化された `Slack interaction: ...` システムイベントを発行します：
  - ブロックアクション：選択値、ラベル、ピッカー値、`workflow_*` メタデータ
  - `view_submission` と `view_closed` イベント（ルーティングされたチャンネルメタデータとフォーム入力付き）

## Ackリアクション

`ackReaction` はOpenClawが受信メッセージを処理中に確認の絵文字を送信します。

解決順序：

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- エージェントIDの絵文字フォールバック（`agents.list[].identity.emoji`、なければ "👀"）

注記：

- Slackはショートコードを期待します（例：`"eyes"`）。
- Slackアカウントまたはグローバルでリアクションを無効化するには `""` を使用。

## タイピングリアクションフォールバック

`typingReaction` はOpenClawが返信を処理中に受信Slackメッセージに一時的なリアクションを追加し、実行終了時に削除します。特にDMでSlackのネイティブアシスタントタイピングが利用できない場合に便利なフォールバックです。

解決順序：

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

注記：

- Slackはショートコードを期待します（例：`"hourglass_flowing_sand"`）。
- リアクションはベストエフォートで、返信またはエラーパスの完了後に自動的にクリーンアップが試みられます。

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
    `channels.slack.userToken` を設定する場合、一般的な読み取りスコープ：

    - `channels:history`、`groups:history`、`im:history`、`mpim:history`
    - `channels:read`、`groups:read`、`im:read`、`mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read`（Slack検索読み取りが必要な場合）

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="チャンネルで返答がない">
    以下の順序で確認：

    - `groupPolicy`
    - チャンネル許可リスト（`channels.slack.channels`）
    - `requireMention`
    - チャンネルごとの `users` 許可リスト

    便利なコマンド：

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="DMメッセージが無視される">
    確認：

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy`（またはレガシー `channels.slack.dm.policy`）
    - ペアリング承認 / 許可リストエントリ

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="ソケットモードが接続できない">
    Slackアプリ設定でボットトークン + アプリトークンとソケットモードの有効化を検証。
  </Accordion>

  <Accordion title="HTTPモードでイベントを受信できない">
    以下を検証：

    - 署名シークレット
    - webhookパス
    - SlackリクエストURL（イベント + インタラクティビティ + スラッシュコマンド）
    - HTTPアカウントごとの一意な `webhookPath`

  </Accordion>

  <Accordion title="ネイティブ/スラッシュコマンドが動作しない">
    意図した設定を確認：

    - ネイティブコマンドモード（`channels.slack.commands.native: true`）とSlackで登録されたスラッシュコマンド
    - または単一スラッシュコマンドモード（`channels.slack.slashCommand.enabled: true`）

    `commands.useAccessGroups` とチャンネル/ユーザー許可リストも確認。

  </Accordion>
</AccordionGroup>

## テキストストリーミング

OpenClawはAgents and AI Apps API経由のSlackネイティブテキストストリーミングをサポートします。

`channels.slack.streaming` はライブプレビュー動作を制御します：

- `off`：ライブプレビューストリーミングを無効化。
- `partial`（デフォルト）：プレビューテキストを最新の部分出力に置換。
- `block`：チャンク化されたプレビュー更新を追加。
- `progress`：生成中に進捗ステータステキストを表示し、最終テキストを送信。

`channels.slack.nativeStreaming` は `streaming` が `partial` の場合のSlackネイティブストリーミングAPI（`chat.startStream` / `chat.appendStream` / `chat.stopStream`）を制御します（デフォルト：`true`）。

ネイティブSlackストリーミングを無効化（ドラフトプレビュー動作を維持）：

```yaml
channels:
  slack:
    streaming: partial
    nativeStreaming: false
```

レガシーキー：

- `channels.slack.streamMode`（`replace | status_final | append`）は `channels.slack.streaming` に自動移行されます。
- ブール値の `channels.slack.streaming` は `channels.slack.nativeStreaming` に自動移行されます。

### 要件

1. Slackアプリ設定で**Agents and AI Apps**を有効化。
2. アプリが `assistant:write` スコープを持つことを確認。
3. そのメッセージに対して返信スレッドが利用可能であること。スレッド選択は引き続き `replyToMode` に従います。

### 動作

- 最初のテキストチャンクがストリームを開始（`chat.startStream`）。
- その後のテキストチャンクが同じストリームに追加（`chat.appendStream`）。
- 返信の終了でストリームを確定（`chat.stopStream`）。
- メディアと非テキストペイロードは通常の配信にフォールバック。
- ストリーミングが返信途中で失敗した場合、OpenClawは残りのペイロードを通常の配信にフォールバック。

## 設定リファレンスポインター

主要リファレンス：

- [設定リファレンス - Slack](/gateway/configuration-reference#slack)

  Slackの重要なフィールド：
  - モード/認証：`mode`、`botToken`、`appToken`、`signingSecret`、`webhookPath`、`accounts.*`
  - DMアクセス：`dm.enabled`、`dmPolicy`、`allowFrom`（レガシー：`dm.policy`、`dm.allowFrom`）、`dm.groupEnabled`、`dm.groupChannels`
  - 互換性トグル：`dangerouslyAllowNameMatching`（緊急用；必要でない限りオフ）
  - チャンネルアクセス：`groupPolicy`、`channels.*`、`channels.*.users`、`channels.*.requireMention`
  - スレッド/履歴：`replyToMode`、`replyToModeByChatType`、`thread.*`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
  - 配信：`textChunkLimit`、`chunkMode`、`mediaMaxMb`、`streaming`、`nativeStreaming`
  - 運用/機能：`configWrites`、`commands.native`、`slashCommand.*`、`actions.*`、`userToken`、`userTokenReadOnly`

## 関連

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [トラブルシューティング](/channels/troubleshooting)
- [設定](/gateway/configuration)
- [スラッシュコマンド](/tools/slash-commands)
