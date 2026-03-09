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

  <Accordion title="ロングポーリングvsウェブフック">
    デフォルト: ロングポーリング。

    Webhookモード:

    - `channels.telegram.webhookUrl`を設定する
    - `channels.telegram.webhookSecret`を設定する（WebhookのURLが設定されている場合は必須）
    - オプション`channels.telegram.webhookPath`（デフォルト`/telegram-webhook`）
    - オプション`channels.telegram.webhookHost`（デフォルト`127.0.0.1`）
    - オプション`channels.telegram.webhookPort`（デフォルト`8787`）

    デフォルトのローカルリスナーは`127.0.0.1:8787`にバインドします。

    パブリックエンドポイントが異なる場合は、前にリバースプロキシを配置し、`webhookUrl`をパブリックURLに向けてください。

  </Accordion>

  <Accordion title="制限、リトライ、CLIターゲット">
    - `channels.telegram.textChunkLimit`のデフォルトは4000です。
    - `channels.telegram.chunkMode="newline"`は長さ分割の前に段落境界（空白行）を優先します。
    - `channels.telegram.mediaMaxMb`（デフォルト100）は受信/送信のTelegramメディアサイズを制限します。

    CLIの送信ターゲットは数値チャットIDまたはユーザー名を使用できます:

```bash
openclaw message send --channel telegram --target 123456789 --message "hi"
openclaw message send --channel telegram --target @name --message "hi"
```

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="ボットがメンションなしのグループメッセージに応答しない">

    - `requireMention=false`の場合、Telegramのプライバシーモードで完全な可視性が許可されている必要があります。
      - BotFather: `/setprivacy` -> 無効化
      - ボットをグループから削除して再追加
    - `openclaw channels status`はconfigがメンションなしのグループメッセージを期待している場合に警告します。
    - クイックセッションテスト: `/activation always`。

  </Accordion>

  <Accordion title="ボットがグループメッセージをまったく受信しない">

    - `channels.telegram.groups`が存在する場合、グループがリストされている必要があります（または`"*"`を含める）
    - グループ内のボットのメンバーシップを確認する
    - ログを確認する: `openclaw logs --follow`

  </Accordion>

  <Accordion title="ポーリングまたはネットワークの不安定性">

    - 不安定なエグレス/TLSを持つVPSホストでは、`channels.telegram.proxy`でプロキシを設定します:

```yaml
channels:
  telegram:
    proxy: socks5://<user>:<password>@proxy-host:1080
```

    - DNS回答を検証する:

```bash
dig +short api.telegram.org A
dig +short api.telegram.org AAAA
```

  </Accordion>
</AccordionGroup>

詳しいヘルプ: [チャンネルトラブルシューティング](/channels/troubleshooting).

## Telegram設定リファレンスポインター

- `channels.telegram.enabled`: チャンネル起動の有効化/無効化。
- `channels.telegram.botToken`: ボットトークン（BotFather）。
- `channels.telegram.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: pairing）。
- `channels.telegram.allowFrom`: DMのallowlist（TelegramのユーザーIDの数値）。
- `channels.telegram.groupPolicy`: `open | allowlist | disabled`（デフォルト: allowlist）。
- `channels.telegram.groupAllowFrom`: グループ送信者のallowlist（TelegramのユーザーIDの数値）。
- `channels.telegram.groups`: グループごとのデフォルト + allowlist。
- `channels.telegram.capabilities.inlineButtons`: `off | dm | group | all | allowlist`（デフォルト: allowlist）。
- `channels.telegram.replyToMode`: `off | first | all`（デフォルト: `off`）。
- `channels.telegram.textChunkLimit`: 送信チャンクサイズ（文字数）。
- `channels.telegram.streaming`: `off | partial | block | progress`（デフォルト: `partial`）。
- `channels.telegram.mediaMaxMb`: 受信/送信のメディアキャップ（MB、デフォルト: 100）。
- `channels.telegram.proxy`: Bot API呼び出しのプロキシURL（SOCKS/HTTP）。
- `channels.telegram.webhookUrl`: Webhookモードを有効化する。
- `channels.telegram.webhookSecret`: Webhookシークレット（必須）。

- [設定リファレンス - Telegram](/gateway/configuration-reference#telegram)

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [マルチエージェントルーティング](/concepts/multi-agent)
- [トラブルシューティング](/channels/troubleshooting)
