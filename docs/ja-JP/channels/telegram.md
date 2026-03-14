---
summary: "Telegram ボットのサポート状況、機能、および設定"
read_when:
  - Telegram の機能または webhook に関する作業を行う場合
title: "Telegram"
x-i18n:
  source_path: channels/telegram.md
---

# Telegram（Bot API）

ステータス：grammY を介したボット DM およびグループでの利用が本番環境に対応済みです。デフォルトはロングポーリングモードで、webhook モードはオプションです。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    Telegram のデフォルト DM ポリシーはペアリングです。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    チャンネル横断的な診断と修復プレイブック。
  </Card>
  <Card title="ゲートウェイ設定" icon="settings" href="/gateway/configuration">
    完全なチャンネル設定パターンと例。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="BotFather でボットトークンを作成する">
    Telegram を開き、**@BotFather**（ハンドルが正確に `@BotFather` であることを確認）とチャットします。

    `/newbot` を実行し、プロンプトに従い、トークンを保存します。

  </Step>

  <Step title="トークンと DM ポリシーを設定する">

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
    Telegram は `openclaw channels login telegram` を使用しません。config または env でトークンを設定してからゲートウェイを起動してください。

  </Step>

  <Step title="ゲートウェイを起動して最初の DM を承認する">

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

    ペアリングコードは 1 時間後に期限切れになります。

  </Step>

  <Step title="ボットをグループに追加する">
    ボットをグループに追加し、アクセスモデルに合わせて `channels.telegram.groups` と `groupPolicy` を設定します。
  </Step>
</Steps>

<Note>
トークンの解決順序はアカウントを考慮します。実際には、config の値が環境変数フォールバックよりも優先され、`TELEGRAM_BOT_TOKEN` はデフォルトアカウントにのみ適用されます。
</Note>

## Telegram 側の設定

<AccordionGroup>
  <Accordion title="プライバシーモードとグループの可視性">
    Telegram ボットはデフォルトで**プライバシーモード**が有効になっており、受信できるグループメッセージが制限されます。

    ボットがすべてのグループメッセージを受信する必要がある場合は、次のいずれかを行ってください。

    - `/setprivacy` でプライバシーモードを無効にする、または
    - ボットをグループ管理者にする。

    プライバシーモードを切り替えた後は、変更が適用されるよう各グループでボットを削除して再追加してください。

  </Accordion>

  <Accordion title="グループ権限">
    管理者ステータスは Telegram グループ設定で制御されます。

    管理者ボットはすべてのグループメッセージを受信するため、常時起動のグループ動作に役立ちます。

  </Accordion>

  <Accordion title="便利な BotFather トグル">

    - `/setjoingroups` — グループへの追加を許可/拒否
    - `/setprivacy` — グループの可視性動作

  </Accordion>
</AccordionGroup>

## アクセス制御と有効化

<Tabs>
  <Tab title="DM ポリシー">
    `channels.telegram.dmPolicy` はダイレクトメッセージのアクセスを制御します。

    - `pairing`（デフォルト）
    - `allowlist`（`allowFrom` に少なくとも 1 つの送信者 ID が必要）
    - `open`（`allowFrom` に `"*"` を含める必要あり）
    - `disabled`

    `channels.telegram.allowFrom` は Telegram の数値ユーザー ID を受け付けます。`telegram:` / `tg:` プレフィックスは受け付けられ、正規化されます。
    `dmPolicy: "allowlist"` で `allowFrom` が空の場合、すべての DM がブロックされ、設定検証で拒否されます。
    オンボーディングウィザードは `@username` 入力を受け付け、数値 ID に解決します。
    アップグレード後に config に `@username` の allowlist エントリが含まれている場合は、`openclaw doctor --fix` を実行して解決してください（ベストエフォート；Telegram ボットトークンが必要）。
    以前にペアリングストアの allowlist ファイルに依存していた場合、`openclaw doctor --fix` は allowlist フロー（例：`dmPolicy: "allowlist"` に明示的な ID がまだない場合）のために `channels.telegram.allowFrom` にエントリを回収できます。

    単一オーナーボットの場合、アクセスポリシーを config で永続的に保持するために（以前のペアリング承認に依存するのではなく）、明示的な数値 `allowFrom` ID を持つ `dmPolicy: "allowlist"` を推奨します。

    ### Telegram ユーザー ID の確認方法

    より安全な方法（サードパーティボット不要）：

    1. ボットに DM を送る。
    2. `openclaw logs --follow` を実行する。
    3. `from.id` を読み取る。

    公式 Bot API 方式：

```bash
curl "https://api.telegram.org/bot<bot_token>/getUpdates"
```

    サードパーティ方式（プライバシーが低い）：`@userinfobot` または `@getidsbot`。

  </Tab>

  <Tab title="グループポリシーと allowlist">
    2 つのコントロールが組み合わせて適用されます。

    1. **どのグループが許可されるか**（`channels.telegram.groups`）
       - `groups` 設定なし：
         - `groupPolicy: "open"` の場合：どのグループもグループ ID チェックを通過できる
         - `groupPolicy: "allowlist"`（デフォルト）の場合：`groups` エントリ（または `"*"`）を追加するまでグループはブロックされる
       - `groups` が設定されている場合：allowlist として機能（明示的な ID または `"*"`）

    2. **グループ内のどの送信者が許可されるか**（`channels.telegram.groupPolicy`）
       - `open`
       - `allowlist`（デフォルト）
       - `disabled`

    `groupAllowFrom` はグループ送信者フィルタリングに使用されます。設定されていない場合、Telegram は `allowFrom` にフォールバックします。
    `groupAllowFrom` エントリは Telegram の数値ユーザー ID である必要があります（`telegram:` / `tg:` プレフィックスは正規化されます）。
    Telegram グループまたはスーパーグループのチャット ID を `groupAllowFrom` に入れないでください。負のチャット ID は `channels.telegram.groups` の下に属します。
    数値以外のエントリは送信者承認時に無視されます。
    セキュリティ境界（`2026.2.25+`）：グループ送信者認証は DM ペアリングストアの承認を継承し**ません**。
    ペアリングは DM 専用です。グループの場合は `groupAllowFrom` またはグループ/トピックごとの `allowFrom` を設定してください。
    ランタイムノート：`channels.telegram` が完全に欠落している場合、`channels.defaults.groupPolicy` が明示的に設定されていない限り、ランタイムはデフォルトでフェイルクローズの `groupPolicy="allowlist"` になります。

    例：1 つの特定グループ内のすべてのメンバーを許可：

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

    例：1 つの特定グループ内の特定ユーザーのみを許可：

```json5
{
  channels: {
    telegram: {
      groups: {
        "-1001234567890": {
          requireMention: true,
          allowFrom: ["8734062810", "745123456"],
        },
      },
    },
  },
}
```

    <Warning>
      よくある間違い：`groupAllowFrom` は Telegram グループの allowlist ではありません。

      - `-1001234567890` のような負の Telegram グループまたはスーパーグループのチャット ID は `channels.telegram.groups` の下に入れてください。
      - `8734062810` のような Telegram ユーザー ID は、許可されたグループ内のどのユーザーがボットをトリガーできるかを制限したい場合に `groupAllowFrom` の下に入れてください。
      - 許可されたグループの任意のメンバーがボットと会話できるようにしたい場合にのみ `groupAllowFrom: ["*"]` を使用してください。
    </Warning>

  </Tab>

  <Tab title="メンション動作">
    グループの返信はデフォルトでメンションが必要です。

    メンションは以下から行えます。

    - ネイティブ `@botusername` メンション、または
    - 以下のメンションパターン：
      - `agents.list[].groupChat.mentionPatterns`
      - `messages.groupChat.mentionPatterns`

    セッションレベルのコマンドトグル：

    - `/activation always`
    - `/activation mention`

    これらはセッション状態のみを更新します。永続性のためには config を使用してください。

    永続的な config の例：

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

    グループチャット ID の取得方法：

    - `@userinfobot` / `@getidsbot` にグループメッセージを転送する
    - または `openclaw logs --follow` から `chat.id` を読み取る
    - または Bot API の `getUpdates` を調べる

  </Tab>
</Tabs>

## ランタイム動作

- Telegram はゲートウェイプロセスが所有します。
- ルーティングは決定論的です：Telegram の受信メッセージは Telegram に返信されます（モデルはチャンネルを選択しません）。
- 受信メッセージは返信メタデータとメディアプレースホルダーを含む共有チャンネルエンベロープに正規化されます。
- グループセッションはグループ ID で分離されます。フォーラムトピックは `:topic:<threadId>` を追加してトピックを分離します。
- DM メッセージは `message_thread_id` を持つことができます。OpenClaw はスレッドを考慮したセッションキーでルーティングし、返信のためにスレッド ID を保持します。
- ロングポーリングは grammY ランナーを使用し、チャット/スレッドごとのシーケンシングを行います。全体的なランナーシンクの並行性は `agents.defaults.maxConcurrent` を使用します。
- Telegram Bot API には既読確認サポートがありません（`sendReadReceipts` は適用されません）。

## 機能リファレンス

<AccordionGroup>
  <Accordion title="ライブストリームプレビュー（メッセージ編集）">
    OpenClaw はリアルタイムで部分的な返信をストリーミングできます。

    - ダイレクトチャット：プレビューメッセージ + `editMessageText`
    - グループ/トピック：プレビューメッセージ + `editMessageText`

    要件：

    - `channels.telegram.streaming` は `off | partial | block | progress`（デフォルト：`partial`）
    - `progress` は Telegram では `partial` にマッピングされます（クロスチャンネルの命名との互換性）
    - レガシーの `channels.telegram.streamMode` とブール値の `streaming` は自動マッピングされます

    テキストのみの返信の場合：

    - DM：OpenClaw は同じプレビューメッセージを保持し、最終的に同じ場所で編集します（2 つ目のメッセージなし）
    - グループ/トピック：OpenClaw は同じプレビューメッセージを保持し、最終的に同じ場所で編集します（2 つ目のメッセージなし）

    複雑な返信（例：メディアペイロード）の場合、OpenClaw は通常の最終デリバリーにフォールバックしてからプレビューメッセージをクリーンアップします。

    プレビューストリーミングはブロックストリーミングとは別です。Telegram でブロックストリーミングが明示的に有効になっている場合、OpenClaw はダブルストリーミングを避けるためにプレビューストリームをスキップします。

    ネイティブドラフトトランスポートが利用できない/拒否された場合、OpenClaw は自動的に `sendMessage` + `editMessageText` にフォールバックします。

    Telegram 専用の推論ストリーム：

    - `/reasoning stream` は生成中にプレビューに推論を送信します
    - 最終回答は推論テキストなしで送信されます

  </Accordion>

  <Accordion title="フォーマットと HTML フォールバック">
    送信テキストは Telegram の `parse_mode: "HTML"` を使用します。

    - Markdown 風のテキストは Telegram セーフな HTML にレンダリングされます。
    - 生のモデル HTML は Telegram のパースエラーを減らすためにエスケープされます。
    - Telegram がパース済み HTML を拒否した場合、OpenClaw はプレーンテキストで再試行します。

    リンクプレビューはデフォルトで有効になっており、`channels.telegram.linkPreview: false` で無効にできます。

  </Accordion>

  <Accordion title="ネイティブコマンドとカスタムコマンド">
    Telegram コマンドメニューの登録は `setMyCommands` で起動時に処理されます。

    ネイティブコマンドのデフォルト：

    - `commands.native: "auto"` は Telegram のネイティブコマンドを有効にします

    カスタムコマンドメニューエントリの追加：

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

    ルール：

    - 名前は正規化されます（先頭の `/` を削除、小文字化）
    - 有効なパターン：`a-z`、`0-9`、`_`、長さ `1..32`
    - カスタムコマンドはネイティブコマンドを上書きできません
    - 競合/重複はスキップされてログに記録されます

    注意事項：

    - カスタムコマンドはメニューエントリのみです；動作を自動実装しません
    - プラグイン/スキルコマンドは、Telegram メニューに表示されていなくても入力することで機能します

    ネイティブコマンドが無効の場合、組み込みコマンドは削除されます。カスタム/プラグインコマンドは設定されていれば引き続き登録される場合があります。

    よくあるセットアップ失敗：

    - `BOT_COMMANDS_TOO_MUCH` を伴う `setMyCommands failed` は、トリミング後も Telegram メニューがオーバーフローしていることを意味します；プラグイン/スキル/カスタムコマンドを減らすか `channels.telegram.commands.native` を無効にしてください。
    - ネットワーク/fetch エラーを伴う `setMyCommands failed` は通常、`api.telegram.org` へのアウトバウンド DNS/HTTPS がブロックされていることを意味します。

    ### デバイスペアリングコマンド（`device-pair` プラグイン）

    `device-pair` プラグインがインストールされている場合：

    1. `/pair` でセットアップコードを生成する
    2. iOS アプリでコードをペースト
    3. `/pair approve` で最新の保留中リクエストを承認する

    詳細：[ペアリング](/channels/pairing#pair-via-telegram-recommended-for-ios)。

  </Accordion>

  <Accordion title="インラインボタン">
    インラインキーボードのスコープを設定する：

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

    アカウントごとのオーバーライド：

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

    スコープ：

    - `off`
    - `dm`
    - `group`
    - `all`
    - `allowlist`（デフォルト）

    レガシーの `capabilities: ["inlineButtons"]` は `inlineButtons: "all"` にマッピングされます。

    メッセージアクションの例：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  message: "Choose an option:",
  buttons: [
    [
      { text: "Yes", callback_data: "yes" },
      { text: "No", callback_data: "no" },
    ],
    [{ text: "Cancel", callback_data: "cancel" }],
  ],
}
```

    コールバッククリックはテキストとしてエージェントに渡されます：
    `callback_data: <value>`

  </Accordion>

  <Accordion title="エージェントと自動化のための Telegram メッセージアクション">
    Telegram ツールアクションには以下が含まれます：

    - `sendMessage`（`to`、`content`、オプションで `mediaUrl`、`replyToMessageId`、`messageThreadId`）
    - `react`（`chatId`、`messageId`、`emoji`）
    - `deleteMessage`（`chatId`、`messageId`）
    - `editMessage`（`chatId`、`messageId`、`content`）
    - `createForumTopic`（`chatId`、`name`、オプションで `iconColor`、`iconCustomEmojiId`）

    チャンネルメッセージアクションは人間工学的なエイリアス（`send`、`react`、`delete`、`edit`、`sticker`、`sticker-search`、`topic-create`）を公開します。

    ゲーティングコントロール：

    - `channels.telegram.actions.sendMessage`
    - `channels.telegram.actions.deleteMessage`
    - `channels.telegram.actions.reactions`
    - `channels.telegram.actions.sticker`（デフォルト：無効）

    注意：`edit` と `topic-create` は現在デフォルトで有効になっており、個別の `channels.telegram.actions.*` トグルがありません。
    ランタイム送信はアクティブな config/シークレットスナップショット（起動/リロード）を使用するため、アクションパスは送信ごとにアドホックな SecretRef の再解決を行いません。

    リアクション削除のセマンティクス：[/tools/reactions](/tools/reactions)

  </Accordion>

  <Accordion title="返信スレッドタグ">
    Telegram は生成出力での明示的な返信スレッドタグをサポートしています：

    - `[[reply_to_current]]` — トリガーメッセージに返信する
    - `[[reply_to:<id>]]` — 特定の Telegram メッセージ ID に返信する

    `channels.telegram.replyToMode` は処理を制御します：

    - `off`（デフォルト）
    - `first`
    - `all`

    注意：`off` は暗黙の返信スレッドを無効にします。明示的な `[[reply_to_*]]` タグは引き続き処理されます。

  </Accordion>

  <Accordion title="フォーラムトピックとスレッド動作">
    フォーラムスーパーグループ：

    - トピックセッションキーに `:topic:<threadId>` が追加される
    - 返信とタイピングはトピックスレッドを対象とする
    - トピック設定パス：
      `channels.telegram.groups.<chatId>.topics.<threadId>`

    General トピック（`threadId=1`）の特殊ケース：

    - メッセージ送信時に `message_thread_id` が省略される（Telegram は `sendMessage(...thread_id=1)` を拒否）
    - タイピングアクションには引き続き `message_thread_id` が含まれる

    トピック継承：トピックエントリは上書きされない限りグループ設定を継承します（`requireMention`、`allowFrom`、`skills`、`systemPrompt`、`enabled`、`groupPolicy`）。
    `agentId` はトピック専用であり、グループのデフォルトから継承されません。

    **トピックごとのエージェントルーティング**：各トピックはトピック設定で `agentId` を設定することで、異なるエージェントにルーティングできます。これにより各トピックは独自の分離されたワークスペース、メモリ、セッションを持ちます。例：

    ```json5
    {
      channels: {
        telegram: {
          groups: {
            "-1001234567890": {
              topics: {
                "1": { agentId: "main" },      // General トピック → main エージェント
                "3": { agentId: "zu" },        // Dev トピック → zu エージェント
                "5": { agentId: "coder" }      // コードレビュー → coder エージェント
              }
            }
          }
        }
      }
    }
    ```

    各トピックは独自のセッションキーを持ちます：`agent:zu:telegram:group:-1001234567890:topic:3`

    **永続的な ACP トピックバインディング**：フォーラムトピックはトップレベルの型付き ACP バインディングを通じて ACP ハーネスセッションを固定できます：

    - `type: "acp"` と `match.channel: "telegram"` を持つ `bindings[]`

    例：

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

    **チャットからのスレッドバインド ACP スポーン**：

    - `/acp spawn <agent> --thread here|auto` は現在の Telegram トピックを新しい ACP セッションにバインドできます。
    - 後続のトピックメッセージは（`/acp steer` なしで）バインドされた ACP セッションに直接ルーティングされます。
    - OpenClaw は成功したバインド後にトピック内でスポーン確認メッセージをピン留めします。
    - `channels.telegram.threadBindings.spawnAcpSessions=true` が必要です。

    テンプレートコンテキストには以下が含まれます：

    - `MessageThreadId`
    - `IsForum`

    DM スレッド動作：

    - `message_thread_id` を持つプライベートチャットは DM ルーティングを維持しますが、スレッドを考慮したセッションキー/返信ターゲットを使用します。

  </Accordion>

  <Accordion title="音声、動画、ステッカー">
    ### 音声メッセージ

    Telegram は音声メモと音声ファイルを区別します。

    - デフォルト：音声ファイルの動作
    - エージェント返信で `[[audio_as_voice]]` タグを付けると音声メモ送信に強制できます

    メッセージアクションの例：

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

    Telegram は動画ファイルと動画メモを区別します。

    メッセージアクションの例：

```json5
{
  action: "send",
  channel: "telegram",
  to: "123456789",
  media: "https://example.com/video.mp4",
  asVideoNote: true,
}
```

    動画メモはキャプションをサポートしていません；提供されたメッセージテキストは別途送信されます。

    ### ステッカー

    受信ステッカーの処理：

    - 静的 WEBP：ダウンロードして処理（プレースホルダー `<media:sticker>`）
    - アニメーション TGS：スキップ
    - 動画 WEBM：スキップ

    ステッカーのコンテキストフィールド：

    - `Sticker.emoji`
    - `Sticker.setName`
    - `Sticker.fileId`
    - `Sticker.fileUniqueId`
    - `Sticker.cachedDescription`

    ステッカーキャッシュファイル：

    - `~/.openclaw/telegram/sticker-cache.json`

    ステッカーは可能な限り一度だけ説明され、繰り返しのビジョン呼び出しを減らすためにキャッシュされます。

    ステッカーアクションの有効化：

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

    ステッカー送信アクション：

```json5
{
  action: "sticker",
  channel: "telegram",
  to: "123456789",
  fileId: "CAACAgIAAxkBAAI...",
}
```

    キャッシュされたステッカーの検索：

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
    Telegram のリアクションは `message_reaction` 更新として届きます（メッセージペイロードとは別）。

    有効にすると、OpenClaw は次のようなシステムイベントをエンキューします：

    - `Telegram reaction added: 👍 by Alice (@alice) on msg 42`

    設定：

    - `channels.telegram.reactionNotifications`：`off | own | all`（デフォルト：`own`）
    - `channels.telegram.reactionLevel`：`off | ack | minimal | extensive`（デフォルト：`minimal`）

    注意事項：

    - `own` はボット送信メッセージへのユーザーリアクションのみを意味します（送信済みメッセージキャッシュ経由のベストエフォート）。
    - リアクションイベントは Telegram のアクセス制御（`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`）を引き続き尊重します；未承認の送信者はドロップされます。
    - Telegram はリアクション更新でスレッド ID を提供しません。
      - 非フォーラムグループはグループチャットセッションにルーティングされます
      - フォーラムグループはグループの General トピックセッション（`:topic:1`）にルーティングされ、正確な元トピックではありません

    ポーリング/webhook の `allowed_updates` には `message_reaction` が自動的に含まれます。

  </Accordion>

  <Accordion title="Ack リアクション">
    `ackReaction` は OpenClaw が受信メッセージを処理中に承認の絵文字を送信します。

    解決順序：

    - `channels.telegram.accounts.<accountId>.ackReaction`
    - `channels.telegram.ackReaction`
    - `messages.ackReaction`
    - エージェントアイデンティティ絵文字フォールバック（`agents.list[].identity.emoji`、なければ "👀"）

    注意事項：

    - Telegram は Unicode 絵文字を期待します（例："👀"）。
    - チャンネルまたはアカウントのリアクションを無効にするには `""` を使用します。

  </Accordion>

  <Accordion title="Telegram イベントとコマンドからの設定書き込み">
    チャンネル設定の書き込みはデフォルトで有効です（`configWrites !== false`）。

    Telegram トリガーによる書き込みには以下が含まれます：

    - グループ移行イベント（`migrate_to_chat_id`）による `channels.telegram.groups` の更新
    - `/config set` および `/config unset`（コマンドの有効化が必要）

    無効化：

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

  <Accordion title="ロングポーリング vs Webhook">
    デフォルト：ロングポーリング。

    Webhook モード：

    - `channels.telegram.webhookUrl` を設定する
    - `channels.telegram.webhookSecret` を設定する（webhook URL が設定されている場合に必須）
    - オプションで `channels.telegram.webhookPath`（デフォルト `/telegram-webhook`）
    - オプションで `channels.telegram.webhookHost`（デフォルト `127.0.0.1`）
    - オプションで `channels.telegram.webhookPort`（デフォルト `8787`）

    Webhook モードのデフォルトローカルリスナーは `127.0.0.1:8787` にバインドします。

    パブリックエンドポイントが異なる場合は、リバースプロキシを前に置き、`webhookUrl` をパブリック URL に向けてください。
    外部イングレスが意図的に必要な場合は `webhookHost`（例：`0.0.0.0`）を設定してください。

  </Accordion>

  <Accordion title="制限、リトライ、CLI ターゲット">
    - `channels.telegram.textChunkLimit` のデフォルトは 4000 です。
    - `channels.telegram.chunkMode="newline"` は長さ分割の前に段落境界（空行）を優先します。
    - `channels.telegram.mediaMaxMb`（デフォルト 100）は受信および送信 Telegram メディアサイズを制限します。
    - `channels.telegram.timeoutSeconds` は Telegram API クライアントタイムアウトを上書きします（未設定の場合、grammY のデフォルトが適用されます）。
    - グループコンテキスト履歴は `channels.telegram.historyLimit` または `messages.groupChat.historyLimit`（デフォルト 50）を使用します；`0` で無効。
    - DM 履歴コントロール：
      - `channels.telegram.dmHistoryLimit`
      - `channels.telegram.dms["<user_id>"].historyLimit`
    - `channels.telegram.retry` 設定は、回復可能なアウトバウンド API エラーに対する Telegram 送信ヘルパー（CLI/ツール/アクション）に適用されます。

    CLI 送信ターゲットは数値チャット ID またはユーザー名が使用できます：

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
```

    Telegram ポールは `openclaw message poll` を使用し、フォーラムトピックをサポートします：

```bash
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300 --poll-public
```

    Telegram 専用のポールフラグ：

    - `--poll-duration-seconds`（5〜600）
    - `--poll-anonymous`
    - `--poll-public`
    - `--thread-id`（フォーラムトピック用、または `:topic:` ターゲットを使用）

    アクションゲーティング：

    - `channels.telegram.actions.sendMessage=false` はポールを含む Telegram アウトバウンドメッセージを無効にします
    - `channels.telegram.actions.poll=false` は通常の送信を有効にしたまま Telegram ポールの作成を無効にします

  </Accordion>

  <Accordion title="Telegram での Exec 承認">
    Telegram は承認者 DM での exec 承認をサポートしており、オプションで元のチャットまたはトピックに承認プロンプトを投稿できます。

    設定パス：

    - `channels.telegram.execApprovals.enabled`
    - `channels.telegram.execApprovals.approvers`
    - `channels.telegram.execApprovals.target`（`dm` | `channel` | `both`、デフォルト：`dm`）
    - `agentFilter`、`sessionFilter`

    承認者は Telegram の数値ユーザー ID である必要があります。`enabled` が false または `approvers` が空の場合、Telegram は exec 承認クライアントとして機能しません。承認リクエストは他の設定された承認ルートまたは exec 承認フォールバックポリシーにフォールバックします。

    デリバリールール：

    - `target: "dm"` は設定された承認者 DM にのみ承認プロンプトを送信します
    - `target: "channel"` は元の Telegram チャット/トピックにプロンプトを返します
    - `target: "both"` は承認者 DM と元のチャット/トピックの両方に送信します

    設定された承認者のみが承認または拒否できます。非承認者は `/approve` を使用できず、Telegram 承認ボタンも使用できません。

    チャンネルデリバリーはチャットにコマンドテキストを表示するため、信頼されたグループ/トピックでのみ `channel` または `both` を有効にしてください。プロンプトがフォーラムトピックに届いた場合、OpenClaw は承認プロンプトと承認後のフォローアップの両方でトピックを保持します。

    インライン承認ボタンは、対象サーフェス（`dm`、`group`、または `all`）を許可する `channels.telegram.capabilities.inlineButtons` にも依存します。

    関連ドキュメント：[Exec 承認](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="ボットがメンションなしのグループメッセージに応答しない">

    - `requireMention=false` の場合、Telegram プライバシーモードが完全な可視性を許可する必要があります。
      - BotFather：`/setprivacy` -> Disable
      - その後、グループでボットを削除して再追加する
    - `openclaw channels status` は、設定がメンションなしのグループメッセージを期待している場合に警告します。
    - `openclaw channels status --probe` は明示的な数値グループ ID を確認できます；ワイルドカード `"*"` はメンバーシッププローブできません。
    - クイックセッションテスト：`/activation always`。

  </Accordion>

  <Accordion title="ボットがグループメッセージをまったく受信しない">

    - `channels.telegram.groups` が存在する場合、グループはリストに含まれている必要があります（または `"*"` を含める）
    - グループ内のボットメンバーシップを確認する
    - ログを確認する：スキップ理由については `openclaw logs --follow`

  </Accordion>

  <Accordion title="コマンドが部分的にしか機能しないか、まったく機能しない">

    - 送信者 ID を認証する（ペアリングおよび/または数値 `allowFrom`）
    - コマンド認証は、グループポリシーが `open` の場合でも適用されます
    - `BOT_COMMANDS_TOO_MUCH` を伴う `setMyCommands failed` は、ネイティブメニューのエントリが多すぎることを意味します；プラグイン/スキル/カスタムコマンドを減らすか、ネイティブメニューを無効にしてください
    - ネットワーク/fetch エラーを伴う `setMyCommands failed` は通常、`api.telegram.org` への DNS/HTTPS の到達可能性の問題を示します

  </Accordion>

  <Accordion title="ポーリングまたはネットワークの不安定性">

    - Node 22+ + カスタム fetch/プロキシは、AbortSignal タイプが一致しない場合に即時中断動作を引き起こす可能性があります。
    - 一部のホストは `api.telegram.org` を最初に IPv6 に解決します；壊れた IPv6 エグレスは断続的な Telegram API 障害を引き起こす可能性があります。
    - ログに `TypeError: fetch failed` または `Network request for 'getUpdates' failed!` が含まれている場合、OpenClaw は現在これらを回復可能なネットワークエラーとして再試行します。
    - 不安定な直接エグレス/TLS を持つ VPS ホストでは、`channels.telegram.proxy` を通じて Telegram API 呼び出しをルーティングします：

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - Node 22+ はデフォルトで `autoSelectFamily=true`（WSL2 を除く）と `dnsResultOrder=ipv4first` になります。
    - ホストが WSL2 であるか、IPv4 専用の動作で明示的に改善される場合は、ファミリー選択を強制します：

```yaml
channels:
  telegram:
    network:
      autoSelectFamily: false
```

    - 環境変数オーバーライド（一時的）：
      - `OPENCLAW_TELEGRAM_DISABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_ENABLE_AUTO_SELECT_FAMILY=1`
      - `OPENCLAW_TELEGRAM_DNS_RESULT_ORDER=ipv4first`
    - DNS 回答の検証：

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

詳細なヘルプ：[チャンネルのトラブルシューティング](/channels/troubleshooting)。

## Telegram 設定リファレンスポインター

主要リファレンス：

- `channels.telegram.enabled`：チャンネル起動の有効/無効。
- `channels.telegram.botToken`：ボットトークン（BotFather）。
- `channels.telegram.tokenFile`：通常のファイルパスからトークンを読み取る。シンボリックリンクは拒否されます。
- `channels.telegram.dmPolicy`：`pairing | allowlist | open | disabled`（デフォルト：pairing）。
- `channels.telegram.allowFrom`：DM allowlist（Telegram の数値ユーザー ID）。`allowlist` には少なくとも 1 つの送信者 ID が必要。`open` には `"*"` が必要。`openclaw doctor --fix` はレガシーの `@username` エントリを ID に解決し、allowlist 移行フローでペアリングストアファイルから allowlist エントリを回収できます。
- `channels.telegram.actions.poll`：Telegram ポールの作成を有効または無効にする（デフォルト：有効；引き続き `sendMessage` が必要）。
- `channels.telegram.defaultTo`：明示的な `--reply-to` が提供されていない場合に CLI `--deliver` で使用されるデフォルトの Telegram ターゲット。
- `channels.telegram.groupPolicy`：`open | allowlist | disabled`（デフォルト：allowlist）。
- `channels.telegram.groupAllowFrom`：グループ送信者 allowlist（Telegram の数値ユーザー ID）。`openclaw doctor --fix` はレガシーの `@username` エントリを ID に解決できます。数値以外のエントリは認証時に無視されます。グループ認証は DM ペアリングストアのフォールバックを使用しません（`2026.2.25+`）。
- マルチアカウントの優先順位：
  - 2 つ以上のアカウント ID が設定されている場合は、`channels.telegram.defaultAccount`（または `channels.telegram.accounts.default` を含める）を設定してデフォルトルーティングを明示的にしてください。
  - どちらも設定されていない場合、OpenClaw は最初の正規化されたアカウント ID にフォールバックし、`openclaw doctor` が警告します。
  - `channels.telegram.accounts.default.allowFrom` と `channels.telegram.accounts.default.groupAllowFrom` は `default` アカウントにのみ適用されます。
  - 名前付きアカウントは、アカウントレベルの値が設定されていない場合は `channels.telegram.allowFrom` と `channels.telegram.groupAllowFrom` を継承します。
  - 名前付きアカウントは `channels.telegram.accounts.default.allowFrom` / `groupAllowFrom` を継承しません。
- `channels.telegram.groups`：グループごとのデフォルト + allowlist（グローバルデフォルトには `"*"` を使用）。
  - `channels.telegram.groups.<id>.groupPolicy`：groupPolicy のグループごとのオーバーライド（`open | allowlist | disabled`）。
  - `channels.telegram.groups.<id>.requireMention`：メンションゲーティングのデフォルト。
  - `channels.telegram.groups.<id>.skills`：スキルフィルター（省略 = すべてのスキル、空 = なし）。
  - `channels.telegram.groups.<id>.allowFrom`：グループごとの送信者 allowlist オーバーライド。
  - `channels.telegram.groups.<id>.systemPrompt`：グループ用の追加システムプロンプト。
  - `channels.telegram.groups.<id>.enabled`：`false` の場合にグループを無効にする。
  - `channels.telegram.groups.<id>.topics.<threadId>.*`：トピックごとのオーバーライド（グループフィールド + トピック専用 `agentId`）。
  - `channels.telegram.groups.<id>.topics.<threadId>.agentId`：このトピックを特定のエージェントにルーティングする（グループレベルおよびバインディングルーティングを上書き）。
- `channels.telegram.groups.<id>.topics.<threadId>.groupPolicy`：groupPolicy のトピックごとのオーバーライド（`open | allowlist | disabled`）。
- `channels.telegram.groups.<id>.topics.<threadId>.requireMention`：メンションゲーティングのトピックごとのオーバーライド。
- トップレベルの `bindings[]` に `type: "acp"` と `match.peer.id` の正規トピック ID `chatId:topic:topicId`：永続的な ACP トピックバインディングフィールド（[ACP エージェント](/tools/acp-agents#channel-specific-settings)を参照）。
- `channels.telegram.direct.<id>.topics.<threadId>.agentId`：DM トピックを特定のエージェントにルーティングする（フォーラムトピックと同じ動作）。
- `channels.telegram.execApprovals.enabled`：このアカウントのチャットベースの exec 承認クライアントとして Telegram を有効にする。
- `channels.telegram.execApprovals.approvers`：exec リクエストを承認または拒否できる Telegram ユーザー ID。exec 承認が有効な場合に必須。
- `channels.telegram.execApprovals.target`：`dm | channel | both`（デフォルト：`dm`）。`channel` と `both` は存在する場合に元の Telegram トピックを保持します。
- `channels.telegram.execApprovals.agentFilter`：転送された承認プロンプトのオプションのエージェント ID フィルター。
- `channels.telegram.execApprovals.sessionFilter`：転送された承認プロンプトのオプションのセッションキーフィルター（部分文字列または正規表現）。
- `channels.telegram.accounts.<account>.execApprovals`：Telegram exec 承認ルーティングと承認者認証のアカウントごとのオーバーライド。
- `channels.telegram.capabilities.inlineButtons`：`off | dm | group | all | allowlist`（デフォルト：allowlist）。
- `channels.telegram.accounts.<account>.capabilities.inlineButtons`：アカウントごとのオーバーライド。
- `channels.telegram.commands.nativeSkills`：Telegram ネイティブスキルコマンドの有効/無効。
- `channels.telegram.replyToMode`：`off | first | all`（デフォルト：`off`）。
- `channels.telegram.textChunkLimit`：アウトバウンドチャンクサイズ（文字数）。
- `channels.telegram.chunkMode`：`length`（デフォルト）または `newline`（長さチャンキングの前に空行で段落境界で分割）。
- `channels.telegram.linkPreview`：アウトバウンドメッセージのリンクプレビューを切り替える（デフォルト：true）。
- `channels.telegram.streaming`：`off | partial | block | progress`（ライブストリームプレビュー；デフォルト：`partial`；`progress` は `partial` にマッピング；`block` はレガシープレビューモード互換）。Telegram プレビューストリーミングは同じ場所で編集される単一のプレビューメッセージを使用します。
- `channels.telegram.mediaMaxMb`：受信/送信 Telegram メディアの上限（MB、デフォルト：100）。
- `channels.telegram.retry`：回復可能なアウトバウンド API エラーに対する Telegram 送信ヘルパー（CLI/ツール/アクション）のリトライポリシー（attempts、minDelayMs、maxDelayMs、jitter）。
- `channels.telegram.network.autoSelectFamily`：Node autoSelectFamily のオーバーライド（true=有効、false=無効）。Node 22+ ではデフォルトで有効、WSL2 ではデフォルトで無効。
- `channels.telegram.network.dnsResultOrder`：DNS 結果順序のオーバーライド（`ipv4first` または `verbatim`）。Node 22+ ではデフォルトで `ipv4first`。
- `channels.telegram.proxy`：Bot API 呼び出しのプロキシ URL（SOCKS/HTTP）。
- `channels.telegram.webhookUrl`：Webhook モードを有効にする（`channels.telegram.webhookSecret` が必要）。
- `channels.telegram.webhookSecret`：Webhook シークレット（webhookUrl が設定されている場合に必須）。
- `channels.telegram.webhookPath`：ローカル Webhook パス（デフォルト `/telegram-webhook`）。
- `channels.telegram.webhookHost`：ローカル Webhook バインドホスト（デフォルト `127.0.0.1`）。
- `channels.telegram.webhookPort`：ローカル Webhook バインドポート（デフォルト `8787`）。
- `channels.telegram.actions.reactions`：Telegram ツールリアクションをゲートする。
- `channels.telegram.actions.sendMessage`：Telegram ツールメッセージ送信をゲートする。
- `channels.telegram.actions.deleteMessage`：Telegram ツールメッセージ削除をゲートする。
- `channels.telegram.actions.sticker`：Telegram ステッカーアクション（送信と検索）をゲートする（デフォルト：false）。
- `channels.telegram.reactionNotifications`：`off | own | all` — どのリアクションがシステムイベントをトリガーするかを制御（デフォルト：未設定の場合 `own`）。
- `channels.telegram.reactionLevel`：`off | ack | minimal | extensive` — エージェントのリアクション機能を制御（デフォルト：未設定の場合 `minimal`）。

- [設定リファレンス - Telegram](/gateway/configuration-reference#telegram)

Telegram 固有の重要なフィールド：

- 起動/認証：`enabled`、`botToken`、`tokenFile`、`accounts.*`（`tokenFile` は通常のファイルを指す必要があります；シンボリックリンクは拒否されます）
- アクセス制御：`dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`、`groups.*.topics.*`、トップレベル `bindings[]`（`type: "acp"`）
- exec 承認：`execApprovals`、`accounts.*.execApprovals`
- コマンド/メニュー：`commands.native`、`commands.nativeSkills`、`customCommands`
- スレッド/返信：`replyToMode`
- ストリーミング：`streaming`（プレビュー）、`blockStreaming`
- フォーマット/デリバリー：`textChunkLimit`、`chunkMode`、`linkPreview`、`responsePrefix`
- メディア/ネットワーク：`mediaMaxMb`、`timeoutSeconds`、`retry`、`network.autoSelectFamily`、`proxy`
- Webhook：`webhookUrl`、`webhookSecret`、`webhookPath`、`webhookHost`
- アクション/機能：`capabilities.inlineButtons`、`actions.sendMessage|editMessage|deleteMessage|reactions|sticker`
- リアクション：`reactionNotifications`、`reactionLevel`
- 書き込み/履歴：`configWrites`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [マルチエージェントルーティング](/concepts/multi-agent)
- [トラブルシューティング](/channels/troubleshooting)
