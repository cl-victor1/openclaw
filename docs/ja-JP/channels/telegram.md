---
summary: "TelegramボットのサポートステータスとCapability、設定について"
read_when:
  - TelegramのフィーチャーやWebhookの作業をするとき
title: "Telegram"
---

# Telegram (Bot API)

ステータス: grammYを使用したボットDMおよびグループ向けにプロダクション対応済み。デフォルトモードはロングポーリング; Webhookモードはオプション。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    TelegramのデフォルトDMポリシーはペアリングです。
  </Card>
  <Card title="チャンネルトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    チャンネル横断の診断と修復プレイブック。
  </Card>
  <Card title="ゲートウェイ設定" icon="settings" href="/gateway/configuration">
    完全なチャンネル設定パターンと例。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="BotFatherでボットトークンを作成する">
    Telegramを開き、**@BotFather**（ハンドルが正確に`@BotFather`であることを確認）でチャットします。

    `/newbot`を実行し、プロンプトに従い、トークンを保存します。

  </Step>

  <Step title="トークンとDMポリシーを設定する">

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

    環境変数フォールバック: `TELEGRAM_BOT_TOKEN=...`（デフォルトアカウントのみ）。
    Telegramは`openclaw channels login telegram`を使用しません。トークンをconfig/envで設定してからゲートウェイを起動します。

  </Step>

  <Step title="ゲートウェイを起動して最初のDMを承認する">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    ペアリングコードは1時間後に失効します。

  </Step>

  <Step title="ボットをグループに追加する">
    ボットをグループに追加し、アクセスモデルに合わせて`channels.telegram.groups`と`groupPolicy`を設定します。
  </Step>
</Steps>

<Note>
トークン解決順序はアカウントを考慮します。実際には、configの値が環境変数フォールバックより優先され、`TELEGRAM_BOT_TOKEN`はデフォルトアカウントにのみ適用されます。
</Note>

## Telegram側の設定

<AccordionGroup>
  <Accordion title="プライバシーモードとグループの可視性">
    Telegramのボットはデフォルトで**プライバシーモード**が有効で、受信できるグループメッセージが制限されます。

    ボットがすべてのグループメッセージを受信する必要がある場合:

    - `/setprivacy`でプライバシーモードを無効化する、または
    - ボットをグループの管理者にする。

    プライバシーモードを切り替えた後、Telegramが変更を適用するよう、各グループでボットを削除して再追加してください。

  </Accordion>

  <Accordion title="グループの権限">
    管理者ステータスはTelegramのグループ設定で制御されます。

    管理者ボットはすべてのグループメッセージを受信します。これは常時オンのグループ動作に役立ちます。

  </Accordion>

  <Accordion title="便利なBotFatherのトグル">

    - `/setjoingroups` - グループへの追加を許可/拒否する
    - `/setprivacy` - グループの可視性動作

  </Accordion>
</AccordionGroup>

## アクセス制御とアクティベーション

<Tabs>
  <Tab title="DMポリシー">
    `channels.telegram.dmPolicy`はダイレクトメッセージへのアクセスを制御します:

    - `pairing`（デフォルト）
    - `allowlist`（`allowFrom`に少なくとも1つの送信者IDが必要）
    - `open`（`allowFrom`に`"*"`を含める必要がある）
    - `disabled`

    `channels.telegram.allowFrom`はTelegramの数値ユーザーIDを受け入れます。`telegram:`/`tg:`プレフィックスは受け入れられ、正規化されます。
    `dmPolicy: "allowlist"`で`allowFrom`が空の場合、すべてのDMがブロックされ、config検証で拒否されます。
    オンボーディングウィザードは`@username`入力を受け入れ、数値IDに解決します。
    configに`@username`のallowlistエントリが含まれている場合は、`openclaw doctor --fix`を実行して解決してください（ベストエフォート; Telegramボットトークンが必要）。
    以前にペアリングストアのallowlistファイルに依存していた場合、`openclaw doctor --fix`はallowlistフローで`channels.telegram.allowFrom`にエントリを回復できます（例: `dmPolicy: "allowlist"`に明示的なIDがまだない場合）。

    単一オーナーのボットには、設定でアクセスポリシーを永続化するため（以前のペアリング承認に依存せず）、明示的な数値`allowFrom` IDを持つ`dmPolicy: "allowlist"`を優先してください。

    ### TelegramユーザーIDを見つける

    より安全な方法（サードパーティボットなし）:

    1. ボットにDMを送信する。
    2. `openclaw logs --follow`を実行する。
    3. `from.id`を読む。

    公式Bot APIメソッド:

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    サードパーティメソッド（プライバシーが低い）: `@userinfobot`または`@getidsbot`。

  </Tab>

  <Tab title="グループポリシーとallowlist">
    2つの制御が一緒に適用されます:

    1. **どのグループが許可されているか** (`channels.telegram.groups`)
       - `groups`設定なし:
         - `groupPolicy: "open"`の場合: グループIDチェックを通過できる
         - `groupPolicy: "allowlist"`（デフォルト）の場合: `groups`エントリ（または`"*"`）を追加するまでグループはブロックされる
       - `groups`が設定されている場合: allowlistとして機能する（明示的なIDまたは`"*"`）

    2. **グループ内でどの送信者が許可されているか** (`channels.telegram.groupPolicy`)
       - `open`
       - `allowlist`（デフォルト）
       - `disabled`

    `groupAllowFrom`はグループ送信者フィルタリングに使用されます。設定されていない場合、Telegramは`allowFrom`にフォールバックします。
    `groupAllowFrom`エントリはTelegramの数値ユーザーID（`telegram:`/`tg:`プレフィックスは正規化される）である必要があります。
    数値以外のエントリは送信者認証で無視されます。
    セキュリティ境界（`2026.2.25+`）: グループ送信者認証はDMペアリングストアの承認を**継承しません**。
    ペアリングはDM専用です。グループには`groupAllowFrom`またはグループ/トピックごとの`allowFrom`を設定してください。
    ランタイムノート: `channels.telegram`が完全に欠落している場合、`channels.defaults.groupPolicy`が明示的に設定されていない限り、ランタイムはfail-closedの`groupPolicy="allowlist"`にデフォルトします。

    例: 特定の1つのグループで任意のメンバーを許可する:

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          groupPolicy: "open",
          requireMention: false,
        },
      },
    },
  },
}
```

  </Tab>

  <Tab title="メンション動作">
    グループの返信はデフォルトでメンションが必要です。

    メンションは以下から可能です:

    - ネイティブの`@botusername`メンション、または
    - 以下のメンションパターン:
      - `agents.list[].groupChat.mentionPatterns`
      - `messages.groupChat.mentionPatterns`

    セッションレベルのコマンドトグル:

    - `/activation always`
    - `/activation mention`

    これらはセッション状態のみを更新します。永続化にはconfigを使用してください。

    永続的なconfig例:

```json5
{
  channels: {
    telegram: {
      groups: {
        "*": { requireMention: false },
      },
    },
  },
}
```

    グループチャットIDを取得する方法:

    - `@userinfobot`/`@getidsbot`にグループメッセージを転送する
    - または`openclaw logs --follow`で`chat.id`を読む
    - またはBot API `getUpdates`を確認する

  </Tab>
</Tabs>

## ランタイム動作

- Telegramはゲートウェイプロセスが所有します。
- ルーティングは決定論的: Telegramへの受信はTelegramに返信します（モデルはチャンネルを選択しません）。
- 受信メッセージは返信メタデータとメディアプレースホルダーを含む共有チャンネルエンベロープに正規化されます。
- グループセッションはグループIDで分離されます。フォーラムトピックはトピックを分離するために`:topic:<threadId>`を追加します。
- DMメッセージは`message_thread_id`を持つことができます。OpenClawはスレッドを考慮したセッションキーでルーティングし、返信のためにスレッドIDを保持します。
- ロングポーリングはgrammYランナーとチャット/スレッドごとのシーケンシングを使用します。全体的なランナーシンクの並行性は`agents.defaults.maxConcurrent`を使用します。
- Telegram Bot APIには既読確認のサポートがありません（`sendReadReceipts`は適用されません）。

## フィーチャーリファレンス

<AccordionGroup>
  <Accordion title="ライブストリームプレビュー（メッセージ編集）">
    OpenClawはリアルタイムで部分的な返信をストリームできます:

    - ダイレクトチャット: プレビューメッセージ + `editMessageText`
    - グループ/トピック: プレビューメッセージ + `editMessageText`

    要件:

    - `channels.telegram.streaming`は`off | partial | block | progress`（デフォルト: `partial`）
    - `progress`はTelegramで`partial`にマッピングされます（チャンネル横断の名前との互換性）
    - レガシーの`channels.telegram.streamMode`とbooleanの`streaming`値は自動マッピングされます

    テキストのみの返信:

    - DM: OpenClawは同じプレビューメッセージを保持し、最終的な編集をインプレースで実行します（2番目のメッセージなし）
    - グループ/トピック: OpenClawは同じプレビューメッセージを保持し、最終的な編集をインプレースで実行します（2番目のメッセージなし）

    複雑な返信（例: メディアペイロード）の場合、OpenClawは通常の最終配信にフォールバックし、プレビューメッセージをクリーンアップします。

    プレビューストリーミングはブロックストリーミングとは別です。Telegramでブロックストリーミングが明示的に有効化されている場合、OpenClawはダブルストリーミングを避けるためにプレビューストリームをスキップします。

    ネイティブドラフトトランスポートが利用不可/拒否された場合、OpenClawは自動的に`sendMessage` + `editMessageText`にフォールバックします。

    Telegram専用の推論ストリーム:

    - `/reasoning stream`は生成中にライブプレビューに推論を送信します
    - 最終回答は推論テキストなしで送信されます

  </Accordion>

  <Accordion title="フォーマットとHTMLフォールバック">
    送信テキストはTelegramの`parse_mode: "HTML"`を使用します。

    - Markdown風のテキストはTelegramセーフなHTMLにレンダリングされます。
    - 生のモデルHTMLはTelegramのパース失敗を減らすためにエスケープされます。
    - TelegramがパースされたHTMLを拒否した場合、OpenClawはプレーンテキストとして再試行します。

    リンクプレビューはデフォルトで有効で、`channels.telegram.linkPreview: false`で無効化できます。

  </Accordion>

  <Accordion title="ネイティブコマンドとカスタムコマンド">
    Telegramコマンドメニューの登録は起動時に`setMyCommands`で処理されます。

    ネイティブコマンドのデフォルト:

    - `commands.native: "auto"`はTelegramのネイティブコマンドを有効化します

    カスタムコマンドメニューエントリを追加する:

```json5
{
  channels: {
    telegram: {
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
    },
  },
}
```

    ルール:

    - 名前は正規化されます（先頭の`/`を除去、小文字）
    - 有効なパターン: `a-z`、`0-9`、`_`、長さ`1..32`
    - カスタムコマンドはネイティブコマンドをオーバーライドできません
    - 競合/重複はスキップされ、ログに記録されます

    注:

    - カスタムコマンドはメニューエントリのみです。動作を自動実装しません
    - プラグイン/スキルコマンドはTelegramメニューに表示されなくても、入力することで機能することがあります

    ネイティブコマンドが無効化されると、組み込みコマンドが削除されます。カスタム/プラグインコマンドは設定されていれば引き続き登録できます。

    よくあるセットアップの失敗:

    - `setMyCommands failed`は通常、`api.telegram.org`へのアウトバウンドDNS/HTTPSがブロックされていることを意味します。

    ### デバイスペアリングコマンド（`device-pair`プラグイン）

    `device-pair`プラグインがインストールされている場合:

    1. `/pair`でセットアップコードを生成する
    2. iOSアプリでコードを貼り付ける
    3. `/pair approve`で最新の保留リクエストを承認する

    詳細: [ペアリング](/channels/pairing#pair-via-telegram-recommended-for-ios).

  </Accordion>

  <Accordion title="インラインボタン">
    インラインキーボードのスコープを設定する:

```json5
{
  channels: {
    telegram: {
      capabilities: {
        inlineButtons: "allowlist",
      },
    },
  },
}
```

    アカウントごとのオーバーライド:

```json5
{
  channels: {
    telegram: {
      accounts: {
        main: {
          capabilities: {
            inlineButtons: "allowlist",
          },
        },
      },
    },
  },
}
```

    スコープ:

    - `off`
    - `dm`
    - `group`
    - `all`
    - `allowlist`（デフォルト）

    レガシーの`capabilities: ["inlineButtons"]`は`inlineButtons: "all"`にマッピングされます。

    メッセージアクション例:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "オプションを選択してください:",
  buttons: [
    [
      { text: "はい", callback_data: "yes" },
      { text: "いいえ", callback_data: "no" },
    ],
    [{ text: "キャンセル", callback_data: "cancel" }],
  ],
}
```

    コールバッククリックはテキストとしてエージェントに渡されます:
    `callback_data: <value>`

  </Accordion>

  <Accordion title="エージェントと自動化のためのTelegramメッセージアクション">
    Telegramツールアクションには以下が含まれます:

    - `sendMessage`（`to`、`content`、オプション`mediaUrl`、`replyToMessageId`、`messageThreadId`）
    - `react`（`chatId`、`messageId`、`emoji`）
    - `deleteMessage`（`chatId`、`messageId`）
    - `editMessage`（`chatId`、`messageId`、`content`）
    - `createForumTopic`（`chatId`、`name`、オプション`iconColor`、`iconCustomEmojiId`）

    チャンネルメッセージアクションは人間工学的なエイリアスを公開します（`send`、`react`、`delete`、`edit`、`sticker`、`sticker-search`、`topic-create`）。

    ゲーティング制御:

    - `channels.telegram.actions.sendMessage`
    - `channels.telegram.actions.deleteMessage`
    - `channels.telegram.actions.reactions`
    - `channels.telegram.actions.sticker`（デフォルト: 無効）

    注: `edit`と`topic-create`は現在デフォルトで有効であり、別の`channels.telegram.actions.*`トグルはありません。

    リアクション削除のセマンティクス: [/tools/reactions](/tools/reactions)

  </Accordion>

  <Accordion title="返信スレッドタグ">
    Telegramは生成された出力で明示的な返信スレッドタグをサポートします:

    - `[[reply_to_current]]` - トリガーメッセージに返信する
    - `[[reply_to:<id>]]` - 特定のTelegramメッセージIDに返信する

    `channels.telegram.replyToMode`は処理を制御します:

    - `off`（デフォルト）
    - `first`
    - `all`

    注: `off`は暗黙の返信スレッディングを無効化します。明示的な`[[reply_to_*]]`タグは引き続き尊重されます。

  </Accordion>

  <Accordion title="フォーラムトピックとスレッド動作">
    フォーラムスーパーグループ:

    - トピックセッションキーに`:topic:<threadId>`を追加する
    - 返信とタイピングはトピックスレッドを対象とする
    - トピック設定パス:
      `channels.telegram.groups.<chatId>.topics.<threadId>`

    一般トピック（`threadId=1`）の特殊ケース:

    - メッセージ送信時に`message_thread_id`を省略します（Telegramは`sendMessage(...thread_id=1)`を拒否します）
    - タイピングアクションには引き続き`message_thread_id`が含まれます

    トピックの継承: トピックエントリはオーバーライドされない限りグループ設定を継承します（`requireMention`、`allowFrom`、`skills`、`systemPrompt`、`enabled`、`groupPolicy`）。
    `agentId`はトピック専用であり、グループのデフォルトから継承されません。

    **トピックごとのエージェントルーティング**: 各トピックはトピックconfigに`agentId`を設定することで異なるエージェントにルーティングできます。これにより各トピックが独自の分離されたワークスペース、メモリ、セッションを持ちます。例:

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // 一般トピック → メインエージェント
                "3": { agentId: "zu" },        // 開発トピック → zuエージェント
                "5": { agentId: "coder" }      // コードレビュー → coderエージェント
              }
            }
          }
        }
      }
    }
    ```

    各トピックには独自のセッションキーがあります: `agent:zu:telegram:group:-1001234567890:topic:3`

    **永続的ACPトピックバインディング**: フォーラムトピックはトップレベルの型付きACPバインディングを通じてACPハーネスセッションをピン留めできます:

    - `type: "acp"`と`match.channel: "telegram"`を持つ`bindings[]`

    例:

    ```json5
    {
      agents: {
        list: [
          {
            id: "codex",
            runtime: {
              type: "acp",
              acp: {
                agent: "codex",
                backend: "acpx",
                mode: "persistent",
                cwd: "/workspace/openclaw",
              },
            },
          },
        ],
      },
      bindings: [
        {
          type: "acp",
          agentId: "codex",
          match: {
            channel: "telegram",
            accountId: "default",
            peer: { kind: "group", id: "-1001234567890:topic:42" },
          },
        },
      ],
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "42": {
                  requireMention: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    これは現在、グループとスーパーグループのフォーラムトピックにスコープされています。

    **チャットからのスレッドバウンドACPスポーン**:

    - `/acp spawn <agent> --thread here|auto`は現在のTelegramトピックを新しいACPセッションにバインドできます。
    - 以降のトピックメッセージはバインドされたACPセッションに直接ルーティングされます（`/acp steer`は不要）。
    - OpenClawはバインドに成功した後、スポーン確認メッセージをトピック内にピン留めします。
    - `channels.telegram.threadBindings.spawnAcpSessions=true`が必要です。

    テンプレートコンテキストには以下が含まれます:

    - `MessageThreadId`
    - `IsForum`

    DMスレッド動作:

    - `message_thread_id`を持つプライベートチャットはDMルーティングを維持しますが、スレッドを考慮したセッションキー/返信ターゲットを使用します。

  </Accordion>

  <Accordion title="音声、動画、ステッカー">
    ### 音声メッセージ

    Telegramはボイスノートと音声ファイルを区別します。

    - デフォルト: 音声ファイルの動作
    - エージェント返信で`[[audio_as_voice]]`タグを使用してボイスノート送信を強制する

    メッセージアクション例:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/voice.ogg",
  asVoice: true,
}
```

    ### 動画メッセージ

    Telegramは動画ファイルと動画ノートを区別します。

    メッセージアクション例:

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    動画ノートはキャプションをサポートしません。提供されたメッセージテキストは別途送信されます。

    ### ステッカー

    受信ステッカーの処理:

    - 静的WEBP: ダウンロードして処理（プレースホルダー`<media:sticker>`）
    - アニメーションTGS: スキップ
    - 動画WEBM: スキップ

    ステッカーコンテキストフィールド:

    - `Sticker.emoji`
    - `Sticker.setName`
    - `Sticker.fileId`
    - `Sticker.fileUniqueId`
    - `Sticker.cachedDescription`

    ステッカーキャッシュファイル:

    - `~/.openclaw/telegram/sticker-cache.json`

    ステッカーは（可能な場合）一度説明され、繰り返しのビジョン呼び出しを減らすためにキャッシュされます。

    ステッカーアクションを有効化する:

```json5
{
  channels: {
    telegram: {
      actions: {
        sticker: true,
      },
    },
  },
}
```

    ステッカー送信アクション:

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    キャッシュされたステッカーを検索:

```json5
{
  action: "sticker-search",
  channel: "telegram",
  query: "cat waving",
  limit: 5,
}
```

  </Accordion>

  <Accordion title="リアクション通知">
    Telegramのリアクションは`message_reaction`更新として届きます（メッセージペイロードとは別）。

    有効化されると、OpenClawは次のようなシステムイベントをエンキューします:

    - `Telegram reaction added: 👍 by Alice (@alice) on msg 42`

    設定:

    - `channels.telegram.reactionNotifications`: `off | own | all`（デフォルト: `own`）
    - `channels.telegram.reactionLevel`: `off | ack | minimal | extensive`（デフォルト: `minimal`）

    注:

    - `own`はボットが送信したメッセージへのユーザーリアクションのみを意味します（送信済みメッセージキャッシュを介したベストエフォート）。
    - リアクションイベントはTelegramアクセス制御（`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`）を引き続き尊重します。未承認の送信者はドロップされます。
    - TelegramはリアクションアップデートでスレッドIDを提供しません。
      - 非フォーラムグループはグループチャットセッションにルーティングされます
      - フォーラムグループはグループ一般トピックセッション（`:topic:1`）にルーティングされます（正確な発生トピックではありません）

    ポーリング/WebhookのAupdateには`message_reaction`が自動的に含まれます。

  </Accordion>

  <Accordion title="確認リアクション">
    `ackReaction`はOpenClawが受信メッセージを処理中に確認の絵文字を送信します。

    解決順序:

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - エージェントのidentityの絵文字フォールバック（`agents.list[].identity.emoji`、それ以外は"👀"）

    注:

    - TelegramはUnicodeの絵文字を期待します（例: "👀"）。
    - チャンネルまたはアカウントのリアクションを無効化するには`""`を使用します。

  </Accordion>

  <Accordion title="Telegramイベントとコマンドからのconfig書き込み">
    チャンネルのconfig書き込みはデフォルトで有効です（`configWrites !== false`）。

    Telegramトリガーの書き込みには以下が含まれます:

    - グループマイグレーションイベント（`migrate_to_chat_id`）で`channels.telegram.groups`を更新する
    - `/config set`と`/config unset`（コマンドの有効化が必要）

    無効化する:

```json5
{
  channels: {
    telegram: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="ロングポーリングvsウェブフック">
    デフォルト: ロングポーリング。

    Webhookモード:

    - `channels.telegram.webhookUrl`を設定する
    - `channels.telegram.webhookSecret`を設定する（WebhookのURLが設定されている場合は必須）
    - オプション`channels.telegram.webhookPath`（デフォルト`/telegram-webhook`）
    - オプション`channels.telegram.webhookHost`（デフォルト`127.0.0.1`）
    - オプション`channels.telegram.webhookPort`（デフォルト`8787`）

    Webhookモードのデフォルトローカルリスナーは`127.0.0.1:8787`にバインドします。

    パブリックエンドポイントが異なる場合は、前にリバースプロキシを配置し、`webhookUrl`をパブリックURLに向けてください。
    外部からのイングレスが意図的に必要な場合は`webhookHost`（例: `0.0.0.0`）を設定します。

  </Accordion>

  <Accordion title="制限、リトライ、CLIターゲット">
    - `channels.telegram.textChunkLimit`のデフォルトは4000です。
    - `channels.telegram.chunkMode="newline"`は長さ分割の前に段落境界（空白行）を優先します。
    - `channels.telegram.mediaMaxMb`（デフォルト100）は受信/送信のTelegramメディアサイズを制限します。
    - `channels.telegram.timeoutSeconds`はTelegram APIクライアントのタイムアウトをオーバーライドします（未設定の場合、grammYのデフォルトが適用される）。
    - グループコンテキスト履歴は`channels.telegram.historyLimit`または`messages.groupChat.historyLimit`を使用します（デフォルト50）。`0`で無効化。
    - DMの履歴制御:
      - `channels.telegram.dmHistoryLimit`
      - `channels.telegram.dms["<user_id>"].historyLimit`
    - `channels.telegram.retry`設定は回復可能なアウトバウンドAPIエラーに対するTelegram送信ヘルパー（CLI/ツール/アクション）のリトライに適用されます。

    CLIの送信ターゲットは数値チャットIDまたはユーザー名を使用できます:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
```

    Telegramポールは`openclaw message poll`を使用し、フォーラムトピックをサポートします:

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    Telegram専用のポールフラグ:

    - `--poll-duration-seconds`（5-600）
    - `--poll-anonymous`
    - `--poll-public`
    - `--thread-id`（フォーラムトピック用。または`:topic:`ターゲットを使用）

    アクションゲーティング:

    - `channels.telegram.actions.sendMessage=false`はポールを含むアウトバウンドTelegramメッセージを無効化します
    - `channels.telegram.actions.poll=false`は通常の送信を有効にしたままTelegramポール作成を無効化します

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="ボットがメンションなしのグループメッセージに応答しない">

    - `requireMention=false`の場合、Telegramのプライバシーモードで完全な可視性が許可されている必要があります。
      - BotFather: `/setprivacy` -> 無効化
      - ボットをグループから削除して再追加
    - `openclaw channels status`はconfigがメンションなしのグループメッセージを期待している場合に警告します。
    - `openclaw channels status --probe`は明示的な数値グループIDを確認できます。ワイルドカード`"*"`はメンバーシップをプローブできません。
    - クイックセッションテスト: `/activation always`。

  </Accordion>

  <Accordion title="ボットがグループメッセージをまったく受信しない">

    - `channels.telegram.groups`が存在する場合、グループがリストされている必要があります（または`"*"`を含める）
    - グループ内のボットのメンバーシップを確認する
    - ログを確認する: スキップ理由を`openclaw logs --follow`で確認

  </Accordion>

  <Accordion title="コマンドが部分的にまたはまったく機能しない">

    - 送信者のidentityを承認する（ペアリングおよび/または数値`allowFrom`）
    - グループポリシーが`open`の場合でも、コマンド認証は引き続き適用されます
    - `setMyCommands failed`は通常、`api.telegram.org`へのDNS/HTTPS到達可能性の問題を示します

  </Accordion>

  <Accordion title="ポーリングまたはネットワークの不安定性">

    - Node 22+とカスタムfetch/proxyは、AbortSignalタイプが一致しない場合に即時アボート動作をトリガーする可能性があります。
    - 一部のホストは最初に`api.telegram.org`をIPv6に解決します。壊れたIPv6エグレスは断続的なTelegram API障害を引き起こす可能性があります。
    - ログに`TypeError: fetch failed`または`Network request for 'getUpdates' failed!`が含まれている場合、OpenClawはこれらを回復可能なネットワークエラーとして再試行するようになりました。
    - 不安定な直接エグレス/TLSを持つVPSホストでは、`channels.telegram.proxy`を通じてTelegram API呼び出しをルーティングします:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+はデフォルトで`autoSelectFamily=true`（WSL2を除く）と`dnsResultOrder=ipv4first`を使用します。
    - ホストがWSL2であるか、IPv4のみの動作の方が良い場合は、ファミリー選択を強制します:

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - 環境変数のオーバーライド（一時的）:
      - `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`
    - DNS回答を検証する:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

詳しいヘルプ: [チャンネルトラブルシューティング](/channels/troubleshooting).

## Telegram設定リファレンスポインター

主要なリファレンス:

- `channels.telegram.enabled`: チャンネル起動の有効化/無効化。
- `channels.telegram.botToken`: ボットトークン（BotFather）。
- `channels.telegram.tokenFile`: ファイルパスからトークンを読み込む。
- `channels.telegram.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: pairing）。
- `channels.telegram.allowFrom`: DMのallowlist（TelegramのユーザーIDの数値）。`allowlist`は少なくとも1つの送信者IDが必要。`open`は`"*"`が必要。`openclaw doctor --fix`はレガシーの`@username`エントリをIDに解決し、allowlistマイグレーションフローでペアリングストアファイルからallowlistエントリを回復できます。
- `channels.telegram.actions.poll`: Telegramポール作成の有効化または無効化（デフォルト: 有効; 引き続き`sendMessage`が必要）。
- `channels.telegram.defaultTo`: 明示的な`--reply-to`が指定されていない場合にCLIの`--deliver`で使用されるデフォルトのTelegramターゲット。
- `channels.telegram.groupPolicy`: `open | allowlist | disabled`（デフォルト: allowlist）。
- `channels.telegram.groupAllowFrom`: グループ送信者のallowlist（TelegramのユーザーIDの数値）。`openclaw doctor --fix`はレガシーの`@username`エントリをIDに解決できます。数値以外のエントリは認証時に無視されます。グループ認証はDMペアリングストアのフォールバックを使用しません（`2026.2.25+`）。
- マルチアカウントの優先順位:
  - 2つ以上のアカウントIDが設定されている場合、デフォルトのルーティングを明示的にするために`channels.telegram.defaultAccount`を設定（または`channels.telegram.accounts.default`を含める）します。
  - どちらも設定されていない場合、OpenClawは最初の正規化されたアカウントIDにフォールバックし、`openclaw doctor`が警告します。
  - `channels.telegram.accounts.default.allowFrom`と`channels.telegram.accounts.default.groupAllowFrom`は`default`アカウントにのみ適用されます。
  - 名前付きアカウントはアカウントレベルの値が未設定の場合、`channels.telegram.allowFrom`と`channels.telegram.groupAllowFrom`を継承します。
  - 名前付きアカウントは`channels.telegram.accounts.default.allowFrom`/`groupAllowFrom`を継承しません。
- `channels.telegram.groups`: グループごとのデフォルト + allowlist（グローバルデフォルトには`"*"`を使用）。
  - `channels.telegram.groups.<id>.groupPolicy`: groupPolicyのグループごとのオーバーライド（`open | allowlist | disabled`）。
  - `channels.telegram.groups.<id>.requireMention`: メンションゲーティングのデフォルト。
  - `channels.telegram.groups.<id>.skills`: スキルフィルター（省略 = 全スキル、空 = なし）。
  - `channels.telegram.groups.<id>.allowFrom`: グループごとの送信者allowlistのオーバーライド。
  - `channels.telegram.groups.<id>.systemPrompt`: グループの追加システムプロンプト。
  - `channels.telegram.groups.<id>.enabled`: `false`の場合にグループを無効化する。
  - `channels.telegram.groups.<id>.topics.<threadId>.*`: トピックごとのオーバーライド（グループフィールド + トピック専用の`agentId`）。
  - `channels.telegram.groups.<id>.topics.<threadId>.agentId`: このトピックを特定のエージェントにルーティングする（グループレベルおよびバインディングルーティングをオーバーライド）。
  - `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`: groupPolicyのトピックごとのオーバーライド（`open | allowlist | disabled`）。
  - `channels.telegram.groups.<id>.topics.<threadId>.requireMention`: メンションゲーティングのトピックごとのオーバーライド。
  - トップレベルの`bindings[]`（`type: "acp"`と`match.peer.id`に正規のトピックID `chatId:topic:topicId`）: 永続的なACPトピックバインディングフィールド（[ACPエージェント](/tools/acp-agents#channel-specific-settings)を参照）。
  - `channels.telegram.direct.<id>.topics.<threadId>.agentId`: DMトピックを特定のエージェントにルーティングする（フォーラムトピックと同じ動作）。
- `channels.telegram.capabilities.inlineButtons`: `off | dm | group | all | allowlist`（デフォルト: allowlist）。
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`: アカウントごとのオーバーライド。
- `channels.telegram.commands.nativeSkills`: Telegramのネイティブスキルコマンドの有効化/無効化。
- `channels.telegram.replyToMode`: `off | first | all`（デフォルト: `off`）。
- `channels.telegram.textChunkLimit`: 送信チャンクサイズ（文字数）。
- `channels.telegram.chunkMode`: `length`（デフォルト）または長さチャンク前に空白行（段落境界）で分割する`newline`。
- `channels.telegram.linkPreview`: 送信メッセージのリンクプレビューを切り替える（デフォルト: true）。
- `channels.telegram.streaming`: `off | partial | block | progress`（ライブストリームプレビュー; デフォルト: `partial`; `progress`は`partial`にマッピング; `block`はレガシープレビューモード互換）。Telegramプレビューストリーミングはインプレースで編集される単一のプレビューメッセージを使用します。
- `channels.telegram.mediaMaxMb`: 受信/送信のTelegramメディアキャップ（MB、デフォルト: 100）。
- `channels.telegram.retry`: 回復可能なアウトバウンドAPIエラーに対するTelegram送信ヘルパー（CLI/ツール/アクション）のリトライポリシー（attempts、minDelayMs、maxDelayMs、jitter）。
- `channels.telegram.network.autoSelectFamily`: NodeのautoSelectFamilyをオーバーライドする（true=有効、false=無効）。Node 22+ではデフォルトで有効。WSL2はデフォルトで無効。
- `channels.telegram.network.dnsResultOrder`: DNS結果の順序をオーバーライドする（`ipv4first`または`verbatim`）。Node 22+ではデフォルトで`ipv4first`。
- `channels.telegram.proxy`: Bot API呼び出しのプロキシURL（SOCKS/HTTP）。
- `channels.telegram.webhookUrl`: Webhookモードを有効化する（`channels.telegram.webhookSecret`が必要）。
- `channels.telegram.webhookSecret`: Webhookシークレット（webhookUrlが設定されている場合は必須）。
- `channels.telegram.webhookPath`: ローカルWebhookパス（デフォルト`/telegram-webhook`）。
- `channels.telegram.webhookHost`: ローカルWebhookのバインドホスト（デフォルト`127.0.0.1`）。
- `channels.telegram.webhookPort`: ローカルWebhookのバインドポート（デフォルト`8787`）。
- `channels.telegram.actions.reactions`: Telegramツールのリアクションをゲートする。
- `channels.telegram.actions.sendMessage`: Telegramツールのメッセージ送信をゲートする。
- `channels.telegram.actions.deleteMessage`: Telegramツールのメッセージ削除をゲートする。
- `channels.telegram.actions.sticker`: Telegramのステッカーアクション（送信と検索）をゲートする（デフォルト: false）。
- `channels.telegram.reactionNotifications`: `off | own | all` — システムイベントをトリガーするリアクションを制御する（デフォルト: `own`）。
- `channels.telegram.reactionLevel`: `off | ack | minimal | extensive` — エージェントのリアクション能力を制御する（デフォルト: `minimal`）。

- [設定リファレンス - Telegram](/gateway/configuration-reference#telegram)

Telegram固有の重要フィールド:

- 起動/認証: `enabled`、`botToken`、`tokenFile`、`accounts.*`
- アクセス制御: `dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`groups.*.topics.*`、トップレベル`bindings[]`（`type: "acp"`）
- コマンド/メニュー: `commands.native`、`commands.nativeSkills`、`customCommands`
- スレッディング/返信: `replyToMode`
- ストリーミング: `streaming`（プレビュー）、`blockStreaming`
- フォーマット/配信: `textChunkLimit`、`chunkMode`、`linkPreview`、`responsePrefix`
- メディア/ネットワーク: `mediaMaxMb`、`timeoutSeconds`、`retry`、`network.autoSelectFamily`、`proxy`
- Webhook: `webhookUrl`、`webhookSecret`、`webhookPath`、`webhookHost`
- アクション/capabilities: `capabilities.inlineButtons`、`actions.sendMessage|editMessage|deleteMessage|reactions|sticker`
- リアクション: `reactionNotifications`、`reactionLevel`
- 書き込み/履歴: `configWrites`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [マルチエージェントルーティング](/concepts/multi-agent)
- [トラブルシューティング](/channels/troubleshooting)
