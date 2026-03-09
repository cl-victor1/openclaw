---
summary: "Discord botのサポート状況、機能、および設定"
read_when:
  - Discord チャンネル機能の作業時
title: "Discord"
---

# Discord (Bot API)

ステータス: 公式 Discord ゲートウェイ経由の DM およびギルドチャンネルに対応済み。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    Discord DM はデフォルトでペアリングモードになります。
  </Card>
  <Card title="スラッシュコマンド" icon="terminal" href="/tools/slash-commands">
    ネイティブコマンドの動作とコマンドカタログ。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    チャンネル横断的な診断と修復フロー。
  </Card>
</CardGroup>

## クイックセットアップ

bot 付きの新しいアプリケーションを作成し、サーバーに bot を追加して、OpenClaw とペアリングする必要があります。自分のプライベートサーバーに bot を追加することをお勧めします。まだお持ちでない場合は、[サーバーを作成してください](https://support.discord.com/hc/en-us/articles/204849977-How-do-I-create-a-server)（**Create My Own > For me and my friends** を選択）。

<Steps>
  <Step title="Discord アプリケーションと bot を作成する">
    [Discord Developer Portal](https://discord.com/developers/applications) にアクセスして、**New Application** をクリックします。「OpenClaw」のような名前を付けてください。

    サイドバーの **Bot** をクリックします。**Username** に OpenClaw エージェントの名前を設定します。

  </Step>

  <Step title="特権インテントを有効にする">
    引き続き **Bot** ページで、**Privileged Gateway Intents** までスクロールして、以下を有効にします：

    - **Message Content Intent**（必須）
    - **Server Members Intent**（推奨。ロールのアローリストと名前から ID へのマッチングに必要）
    - **Presence Intent**（オプション。プレゼンス更新が必要な場合のみ）

  </Step>

  <Step title="bot トークンをコピーする">
    **Bot** ページ上部にスクロールして戻り、**Reset Token** をクリックします。

    <Note>
    名前に反して、これは初めてのトークンを生成するものです。「リセット」されるものはありません。
    </Note>

    トークンをコピーして保存しておいてください。これが **Bot Token** で、すぐに必要になります。

  </Step>

  <Step title="招待 URL を生成して bot をサーバーに追加する">
    サイドバーの **OAuth2** をクリックします。bot をサーバーに追加するための適切な権限を持つ招待 URL を生成します。

    **OAuth2 URL Generator** までスクロールして以下を有効にします：

    - `bot`
    - `applications.commands`

    下に **Bot Permissions** セクションが表示されます。以下を有効にします：

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions（オプション）

    下部の生成された URL をコピーし、ブラウザに貼り付け、サーバーを選択して、**Continue** をクリックして接続します。Discord サーバーに bot が表示されるようになります。

  </Step>

  <Step title="Developer Mode を有効にして ID を収集する">
    Discord アプリに戻り、内部 ID をコピーできるよう Developer Mode を有効にする必要があります。

    1. **User Settings**（アバターの横にある歯車アイコン）→ **Advanced** → **Developer Mode** をオンにする
    2. サイドバーの**サーバーアイコン**を右クリック → **Copy Server ID**
    3. **自分のアバター**を右クリック → **Copy User ID**

    **Server ID** と **User ID** を Bot Token と一緒に保存しておいてください。次のステップで 3 つすべてを OpenClaw に送信します。

  </Step>

  <Step title="サーバーメンバーからの DM を許可する">
    ペアリングが機能するためには、Discord が bot からの DM を許可する必要があります。**サーバーアイコン**を右クリック → **Privacy Settings** → **Direct Messages** をオンにします。

    これにより、サーバーメンバー（bot を含む）があなたに DM を送れるようになります。OpenClaw で Discord DM を使用したい場合は、これを有効にしたままにしてください。ギルドチャンネルのみを使用する予定の場合は、ペアリング後に DM を無効にできます。

  </Step>

  <Step title="ステップ 0: bot トークンをセキュアに設定する（チャットで送信しないこと）">
    Discord bot トークンはシークレット（パスワードのようなもの）です。エージェントにメッセージを送る前に、OpenClaw が動作しているマシンで設定してください。

```bash
openclaw config set channels.discord.token '"YOUR_BOT_TOKEN"' --json
openclaw config set channels.discord.enabled true --json
openclaw gateway
```

    OpenClaw がすでにバックグラウンドサービスとして動作している場合は、代わりに `openclaw gateway restart` を使用してください。

  </Step>

  <Step title="OpenClaw を設定してペアリングする">

    <Tabs>
      <Tab title="エージェントに依頼する">
        既存のチャンネル（例：Telegram）で OpenClaw エージェントと会話して伝えます。Discord が初めてのチャンネルの場合は、代わりに CLI / config タブを使用してください。

        > "Discord bot トークンを設定済みです。User ID `<user_id>` と Server ID `<server_id>` で Discord のセットアップを完了してください。"
      </Tab>
      <Tab title="CLI / config">
        ファイルベースの設定を希望する場合は、以下を設定します：

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "YOUR_BOT_TOKEN",
    },
  },
}
```

        デフォルトアカウントの環境変数フォールバック：

```bash
DISCORD_BOT_TOKEN=...
```

        `channels.discord.token` では SecretRef の値もサポートされています（env/file/exec プロバイダー）。[シークレット管理](/gateway/secrets)をご参照ください。

      </Tab>
    </Tabs>

  </Step>

  <Step title="最初の DM ペアリングを承認する">
    ゲートウェイが起動するまで待ち、Discord で bot に DM を送ります。ペアリングコードが返信されます。

    <Tabs>
      <Tab title="エージェントに依頼する">
        既存のチャンネルでエージェントにペアリングコードを送ります：

        > "この Discord ペアリングコードを承認してください: `<CODE>`"
      </Tab>
      <Tab title="CLI">

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

      </Tab>
    </Tabs>

    ペアリングコードは 1 時間後に有効期限が切れます。

    これで Discord の DM でエージェントとチャットできるようになります。

  </Step>
</Steps>

<Note>
トークンの解決はアカウントを認識します。設定のトークン値は環境変数のフォールバックより優先されます。`DISCORD_BOT_TOKEN` はデフォルトアカウントにのみ使用されます。
</Note>

## 推奨: ギルドワークスペースを設定する

DM が動作したら、Discord サーバーをフルワークスペースとして設定できます。各チャンネルが独自のコンテキストを持つエージェントセッションを取得します。これは自分と bot だけのプライベートサーバーに推奨されます。

<Steps>
  <Step title="サーバーをギルドアローリストに追加する">
    これにより、エージェントが DM だけでなくサーバーのすべてのチャンネルで応答できるようになります。

    <Tabs>
      <Tab title="エージェントに依頼する">
        > "Discord Server ID `<server_id>` をギルドアローリストに追加してください"
      </Tab>
      <Tab title="Config">

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="@mention なしでの応答を許可する">
    デフォルトでは、エージェントは @mention されたときのみギルドチャンネルで応答します。プライベートサーバーでは、すべてのメッセージに応答させたいと思うでしょう。

    <Tabs>
      <Tab title="エージェントに依頼する">
        > "@mention なしでもこのサーバーで応答できるようにしてください"
      </Tab>
      <Tab title="Config">
        ギルド設定で `requireMention: false` を設定します：

```json5
{
  channels: {
    discord: {
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: false,
        },
      },
    },
  },
}
```

      </Tab>
    </Tabs>

  </Step>

  <Step title="ギルドチャンネルでのメモリを計画する">
    デフォルトでは、長期メモリ（MEMORY.md）は DM セッションにのみ読み込まれます。ギルドチャンネルでは MEMORY.md が自動的に読み込まれません。

    <Tabs>
      <Tab title="エージェントに依頼する">
        > "Discord チャンネルで質問するときは、MEMORY.md からの長期コンテキストが必要な場合は memory_search または memory_get を使用してください。"
      </Tab>
      <Tab title="手動">
        すべてのチャンネルで共有コンテキストが必要な場合は、安定した指示を `AGENTS.md` または `USER.md` に記載してください（すべてのセッションに挿入されます）。長期的なメモは `MEMORY.md` に保持し、メモリツールでオンデマンドにアクセスします。
      </Tab>
    </Tabs>

  </Step>
</Steps>

Discord サーバーにいくつかのチャンネルを作成して会話を始めましょう。エージェントはチャンネル名を認識でき、各チャンネルは独自の分離セッションを取得します。`#coding`、`#home`、`#research` など、ワークフローに合ったものを設定できます。

## ランタイムモデル

- ゲートウェイが Discord 接続を所有します。
- 返信ルーティングは決定論的：Discord 受信は Discord に返信します。
- デフォルト（`session.dmScope=main`）では、ダイレクトチャットはエージェントのメインセッション（`agent:main:main`）を共有します。
- ギルドチャンネルは分離されたセッションキー（`agent:<agentId>:discord:channel:<channelId>`）を持ちます。
- グループ DM はデフォルトで無視されます（`channels.discord.dm.groupEnabled=false`）。
- ネイティブスラッシュコマンドは分離されたコマンドセッション（`agent:<agentId>:discord:slash:<userId>`）で実行されます。ルーティングされた会話セッションには引き続き `CommandTargetSessionKey` が渡されます。

## フォーラムチャンネル

Discord のフォーラムチャンネルとメディアチャンネルはスレッド投稿のみを受け付けます。OpenClaw はスレッドを作成する 2 つの方法をサポートしています：

- フォーラム親（`channel:<forumId>`）にメッセージを送信して自動的にスレッドを作成します。スレッドのタイトルはメッセージの最初の空でない行を使用します。
- `openclaw message thread create` を使用してスレッドを直接作成します。フォーラムチャンネルには `--message-id` を渡さないでください。

例：フォーラム親に送信してスレッドを作成する

```bash
openclaw message send --channel discord --target channel:<forumId> \
  --message "Topic title\nBody of the post"
```

例：フォーラムスレッドを明示的に作成する

```bash
openclaw message thread create --channel discord --target channel:<forumId> \
  --thread-name "Topic title" --message "Body of the post"
```

フォーラム親は Discord コンポーネントを受け付けません。コンポーネントが必要な場合は、スレッド自体（`channel:<threadId>`）に送信してください。

## インタラクティブコンポーネント

OpenClaw はエージェントメッセージに Discord コンポーネント v2 コンテナをサポートしています。`components` ペイロードを含むメッセージツールを使用します。インタラクション結果は通常の受信メッセージとしてエージェントにルーティングされ、既存の Discord `replyToMode` 設定に従います。

サポートされているブロック：

- `text`、`section`、`separator`、`actions`、`media-gallery`、`file`
- アクション行は最大 5 つのボタンまたは単一のセレクトメニューを許可
- セレクトタイプ：`string`、`user`、`role`、`mentionable`、`channel`

デフォルトではコンポーネントは 1 回限りの使用です。`components.reusable=true` を設定すると、有効期限が切れるまでボタン、セレクト、フォームを複数回使用できます。

ボタンをクリックできるユーザーを制限するには、そのボタンに `allowedUsers` を設定します（Discord ユーザー ID、タグ、または `*`）。設定されている場合、一致しないユーザーにはエフェメラルな拒否が返されます。

`/model` と `/models` スラッシュコマンドは、プロバイダーとモデルのドロップダウンに送信ステップを加えたインタラクティブなモデルピッカーを開きます。ピッカーの返信はエフェメラルで、呼び出したユーザーのみが使用できます。

ファイル添付：

- `file` ブロックは添付参照（`attachment://<filename>`）を指す必要があります
- `media`/`path`/`filePath`（単一ファイル）で添付を提供します。複数ファイルには `media-gallery` を使用
- 添付参照と一致させるべき場合は `filename` でアップロード名を上書きします

モーダルフォーム：

- 最大 5 つのフィールドで `components.modal` を追加
- フィールドタイプ：`text`、`checkbox`、`radio`、`select`、`role-select`、`user-select`
- OpenClaw がトリガーボタンを自動的に追加します

例：

```json5
{
  channel: "discord",
  action: "send",
  to: "channel:123456789012345678",
  message: "Optional fallback text",
  components: {
    reusable: true,
    text: "Choose a path",
    blocks: [
      {
        type: "actions",
        buttons: [
          {
            label: "Approve",
            style: "success",
            allowedUsers: ["123456789012345678"],
          },
          { label: "Decline", style: "danger" },
        ],
      },
      {
        type: "actions",
        select: {
          type: "string",
          placeholder: "Pick an option",
          options: [
            { label: "Option A", value: "a" },
            { label: "Option B", value: "b" },
          ],
        },
      },
    ],
    modal: {
      title: "Details",
      triggerLabel: "Open form",
      fields: [
        { type: "text", label: "Requester" },
        {
          type: "select",
          label: "Priority",
          options: [
            { label: "Low", value: "low" },
            { label: "High", value: "high" },
          ],
        },
      ],
    },
  },
}
```

## アクセス制御とルーティング

<Tabs>
  <Tab title="DM ポリシー">
    `channels.discord.dmPolicy` が DM アクセスを制御します（レガシー：`channels.discord.dm.policy`）：

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`channels.discord.allowFrom` に `"*"` が含まれている必要があります。レガシー：`channels.discord.dm.allowFrom`）
    - `disabled`

    DM ポリシーがオープンでない場合、未知のユーザーはブロックされます（または `pairing` モードではペアリングを求められます）。

    マルチアカウントの優先順位：

    - `channels.discord.accounts.default.allowFrom` は `default` アカウントにのみ適用されます。
    - 名前付きアカウントは、独自の `allowFrom` が設定されていない場合、`channels.discord.allowFrom` を継承します。
    - 名前付きアカウントは `channels.discord.accounts.default.allowFrom` を継承しません。

    配信用の DM ターゲットフォーマット：

    - `user:<id>`
    - `<@id>` メンション

    裸の数値 ID は曖昧で、明示的なユーザー/チャンネルのターゲット種別が提供されない限り拒否されます。

  </Tab>

  <Tab title="ギルドポリシー">
    ギルドの処理は `channels.discord.groupPolicy` で制御されます：

    - `open`
    - `allowlist`
    - `disabled`

    `channels.discord` が存在する場合のセキュアなベースラインは `allowlist` です。

    `allowlist` の動作：

    - ギルドは `channels.discord.guilds` に一致する必要があります（`id` が優先、スラグも受け付けます）
    - オプションの送信者アローリスト：`users`（安定した ID が推奨）と `roles`（ロール ID のみ）。どちらかが設定されている場合、送信者は `users` または `roles` に一致すると許可されます
    - 名前/タグの直接マッチングはデフォルトで無効です。互換モードとしてのみ `channels.discord.dangerouslyAllowNameMatching: true` を有効にしてください
    - 名前/タグは `users` でサポートされていますが、ID の方が安全です。`openclaw security audit` は名前/タグエントリが使用されている場合に警告します
    - ギルドに `channels` が設定されている場合、リストにないチャンネルは拒否されます
    - ギルドに `channels` ブロックがない場合、そのアローリストに登録されたギルドのすべてのチャンネルが許可されます

    例：

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        "123456789012345678": {
          requireMention: true,
          ignoreOtherMentions: true,
          users: ["987654321098765432"],
          roles: ["123456789012345678"],
          channels: {
            general: { allow: true },
            help: { allow: true, requireMention: true },
          },
        },
      },
    },
  },
}
```

    `DISCORD_BOT_TOKEN` のみを設定して `channels.discord` ブロックを作成しない場合、ランタイムのフォールバックは `groupPolicy="allowlist"` です（ログに警告あり）。`channels.defaults.groupPolicy` が `open` であっても同様です。

  </Tab>

  <Tab title="メンションとグループ DM">
    ギルドメッセージはデフォルトでメンションがゲートとして機能します。

    メンション検出には以下が含まれます：

    - 明示的な bot メンション
    - 設定されたメンションパターン（`agents.list[].groupChat.mentionPatterns`、フォールバック：`messages.groupChat.mentionPatterns`）
    - サポートされているケースでの bot への返信の暗黙的な動作

    `requireMention` はギルド/チャンネルごとに設定されます（`channels.discord.guilds...`）。
    `ignoreOtherMentions` はオプションで、bot 以外のユーザー/ロールにメンションするメッセージをドロップします（@everyone/@here を除く）。

    グループ DM：

    - デフォルト：無視（`dm.groupEnabled=false`）
    - `dm.groupChannels`（チャンネル ID またはスラグ）経由のオプションアローリスト

  </Tab>
</Tabs>

### ロールベースのエージェントルーティング

`bindings[].match.roles` を使用して、ロール ID によって Discord ギルドメンバーを異なるエージェントにルーティングします。ロールベースのバインディングはロール ID のみを受け付け、ピアまたは親ピアのバインディングの後、ギルドのみのバインディングの前に評価されます。バインディングに他のマッチフィールドも設定されている場合（例：`peer` + `guildId` + `roles`）、設定されたすべてのフィールドが一致する必要があります。

```json5
{
  bindings: [
    {
      agentId: "opus",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
        roles: ["111111111111111111"],
      },
    },
    {
      agentId: "sonnet",
      match: {
        channel: "discord",
        guildId: "123456789012345678",
      },
    },
  ],
}
```

## Developer Portal の設定

<AccordionGroup>
  <Accordion title="アプリと bot を作成する">

    1. Discord Developer Portal -> **Applications** -> **New Application**
    2. **Bot** -> **Add Bot**
    3. bot トークンをコピーする

  </Accordion>

  <Accordion title="特権インテント">
    **Bot -> Privileged Gateway Intents** で以下を有効にします：

    - Message Content Intent
    - Server Members Intent（推奨）

    Presence インテントはオプションで、メンバーのプレゼンス更新を受信したい場合のみ必要です。bot のプレゼンス設定（`setPresence`）にはメンバーのプレゼンス更新を有効にする必要はありません。

  </Accordion>

  <Accordion title="OAuth スコープとベースライン権限">
    OAuth URL ジェネレーター：

    - スコープ：`bot`、`applications.commands`

    典型的なベースライン権限：

    - View Channels
    - Send Messages
    - Read Message History
    - Embed Links
    - Attach Files
    - Add Reactions（オプション）

    明示的に必要な場合を除き、`Administrator` は避けてください。

  </Accordion>

  <Accordion title="ID をコピーする">
    Discord Developer Mode を有効にして、以下をコピーします：

    - サーバー ID
    - チャンネル ID
    - ユーザー ID

    信頼性の高い監査とプローブのために、OpenClaw の設定では数値 ID を優先してください。

  </Accordion>
</AccordionGroup>

## ネイティブコマンドとコマンド認証

- `commands.native` はデフォルトで `"auto"` であり、Discord では有効です。
- チャンネルごとのオーバーライド：`channels.discord.commands.native`。
- `commands.native=false` は以前に登録された Discord ネイティブコマンドを明示的にクリアします。
- ネイティブコマンド認証は、通常のメッセージ処理と同じ Discord アローリスト/ポリシーを使用します。
- 認証されていないユーザーには Discord UI でコマンドが表示される場合がありますが、実行時に OpenClaw 認証が適用され、「認証されていません」と返します。

コマンドカタログと動作については、[スラッシュコマンド](/tools/slash-commands)を参照してください。

デフォルトのスラッシュコマンド設定：

- `ephemeral: true`

## 機能の詳細

<AccordionGroup>
  <Accordion title="返信タグとネイティブ返信">
    Discord はエージェント出力で返信タグをサポートしています：

    - `[[reply_to_current]]`
    - `[[reply_to:<id>]]`

    `channels.discord.replyToMode` で制御します：

    - `off`（デフォルト）
    - `first`
    - `all`

    注：`off` は暗黙的な返信スレッドを無効にします。明示的な `[[reply_to_*]]` タグは引き続き適用されます。

    エージェントが特定のメッセージをターゲットにできるよう、メッセージ ID はコンテキスト/履歴に表示されます。

  </Accordion>

  <Accordion title="ライブストリームプレビュー">
    OpenClaw は一時的なメッセージを送信し、テキストが届くにつれて編集することで、下書き返信をストリーミングできます。

    - `channels.discord.streaming` がプレビューストリーミングを制御します（`off` | `partial` | `block` | `progress`、デフォルト：`off`）。
    - `progress` はクロスチャンネルの一貫性のために受け付けられ、Discord では `partial` にマップされます。
    - `channels.discord.streamMode` はレガシーエイリアスで自動移行されます。
    - `partial` はトークンが届くにつれて単一のプレビューメッセージを編集します。
    - `block` は下書きサイズのチャンクを出力します（サイズとブレークポイントを調整するには `draftChunk` を使用）。

    例：

```json5
{
  channels: {
    discord: {
      streaming: "partial",
    },
  },
}
```

    `block` モードのチャンク設定デフォルト（`channels.discord.textChunkLimit` にクランプ）：

```json5
{
  channels: {
    discord: {
      streaming: "block",
      draftChunk: {
        minChars: 200,
        maxChars: 800,
        breakPreference: "paragraph",
      },
    },
  },
}
```

    プレビューストリーミングはテキストのみです。メディア返信は通常の配信にフォールバックします。

    注：プレビューストリーミングはブロックストリーミングとは別です。Discord でブロックストリーミングが明示的に有効になっている場合、二重ストリーミングを避けるためにプレビューストリームをスキップします。

  </Accordion>

  <Accordion title="履歴、コンテキスト、スレッドの動作">
    ギルド履歴コンテキスト：

    - `channels.discord.historyLimit` デフォルト `20`
    - フォールバック：`messages.groupChat.historyLimit`
    - `0` で無効化

    DM 履歴の制御：

    - `channels.discord.dmHistoryLimit`
    - `channels.discord.dms["<user_id>"].historyLimit`

    スレッドの動作：

    - Discord スレッドはチャンネルセッションとしてルーティングされます
    - 親スレッドのメタデータは親セッションリンクに使用できます
    - スレッド固有のエントリが存在しない限り、スレッド設定は親チャンネルの設定を継承します

    チャンネルトピックは**信頼されていない**コンテキスト（システムプロンプトではなく）として挿入されます。

  </Accordion>

  <Accordion title="サブエージェント用のスレッドバウンドセッション">
    Discord はスレッドをセッションターゲットにバインドできるため、そのスレッドへのフォローアップメッセージは同じセッション（サブエージェントセッションを含む）にルーティングされ続けます。

    コマンド：

    - `/focus <target>` 現在/新しいスレッドをサブエージェント/セッションターゲットにバインド
    - `/unfocus` 現在のスレッドのバインドを削除
    - `/agents` アクティブな実行とバインド状態を表示
    - `/session idle <duration|off>` フォーカスされたバインドの非アクティブ時の自動アンフォーカスを確認/更新
    - `/session max-age <duration|off>` フォーカスされたバインドのハード最大期間を確認/更新

    設定：

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
    },
  },
  channels: {
    discord: {
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSubagentSessions: false, // opt-in
      },
    },
  },
}
```

    注：

    - `session.threadBindings.*` はグローバルデフォルトを設定します。
    - `channels.discord.threadBindings.*` は Discord の動作をオーバーライドします。
    - `spawnSubagentSessions` は `sessions_spawn({ thread: true })` でスレッドを自動作成/バインドするには true にする必要があります。
    - `spawnAcpSessions` は ACP（`/acp spawn ... --thread ...` または `sessions_spawn({ runtime: "acp", thread: true })`）でスレッドを自動作成/バインドするには true にする必要があります。
    - スレッドバインディングがアカウントで無効になっている場合、`/focus` および関連するスレッドバインディング操作は利用できません。

    [サブエージェント](/tools/subagents)、[ACP エージェント](/tools/acp-agents)、および[設定リファレンス](/gateway/configuration-reference)を参照してください。

  </Accordion>

  <Accordion title="永続 ACP チャンネルバインディング">
    安定した「常時オン」の ACP ワークスペースには、Discord の会話をターゲットとするトップレベルの型付き ACP バインディングを設定します。

    設定パス：

    - `bindings[]` に `type: "acp"` と `match.channel: "discord"` を設定

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
        channel: "discord",
        accountId: "default",
        peer: { kind: "channel", id: "222222222222222222" },
      },
      acp: { label: "codex-main" },
    },
  ],
  channels: {
    discord: {
      guilds: {
        "111111111111111111": {
          channels: {
            "222222222222222222": {
              requireMention: false,
            },
          },
        },
      },
    },
  },
}
```

    注：

    - スレッドメッセージは親チャンネルの ACP バインディングを継承できます。
    - バインドされたチャンネルまたはスレッドでは、`/new` と `/reset` は同じ ACP セッションをその場でリセットします。
    - 一時的なスレッドバインディングは引き続き機能し、アクティブな間はターゲット解決をオーバーライドできます。

    バインディングの動作については[ACP エージェント](/tools/acp-agents)を参照してください。

  </Accordion>

  <Accordion title="リアクション通知">
    ギルドごとのリアクション通知モード：

    - `off`
    - `own`（デフォルト）
    - `all`
    - `allowlist`（`guilds.<id>.users` を使用）

    リアクションイベントはシステムイベントに変換され、ルーティングされた Discord セッションに添付されます。

  </Accordion>

  <Accordion title="Ack リアクション">
    `ackReaction` は OpenClaw が受信メッセージを処理している間に確認の絵文字を送信します。

    解決順序：

    - `channels.discord.accounts.<accountId>.ackReaction`
    - `channels.discord.ackReaction`
    - `messages.ackReaction`
    - エージェントアイデンティティ絵文字フォールバック（`agents.list[].identity.emoji`、なければ「👀」）

    注：

    - Discord は Unicode 絵文字またはカスタム絵文字名を受け付けます。
    - チャンネルまたはアカウントのリアクションを無効にするには `""` を使用します。

  </Accordion>

  <Accordion title="設定の書き込み">
    チャンネルからの設定書き込みはデフォルトで有効です。

    これは `/config set|unset` フロー（コマンド機能が有効な場合）に影響します。

    無効にする方法：

```json5
{
  channels: {
    discord: {
      configWrites: false,
    },
  },
}
```

  </Accordion>

  <Accordion title="ゲートウェイプロキシ">
    `channels.discord.proxy` で Discord ゲートウェイの WebSocket トラフィックとスタートアップ REST ルックアップ（アプリケーション ID + アローリスト解決）を HTTP(S) プロキシ経由でルーティングします。

```json5
{
  channels: {
    discord: {
      proxy: "http://proxy.example:8080",
    },
  },
}
```

    アカウントごとのオーバーライド：

```json5
{
  channels: {
    discord: {
      accounts: {
        primary: {
          proxy: "http://proxy.example:8080",
        },
      },
    },
  },
}
```

  </Accordion>

  <Accordion title="PluralKit サポート">
    プロキシメッセージをシステムメンバーアイデンティティにマップするための PluralKit 解決を有効にします：

```json5
{
  channels: {
    discord: {
      pluralkit: {
        enabled: true,
        token: "pk_live_...", // オプション。プライベートシステムに必要
      },
    },
  },
}
```

    注：

    - アローリストでは `pk:<memberId>` を使用できます
    - メンバーの表示名は `channels.discord.dangerouslyAllowNameMatching: true` の場合のみ名前/スラグで一致します
    - ルックアップは元のメッセージ ID を使用し、タイムウィンドウ制限があります
    - ルックアップが失敗した場合、プロキシメッセージは bot メッセージとして扱われ、`allowBots=true` でない限りドロップされます

  </Accordion>

  <Accordion title="プレゼンス設定">
    ステータスまたはアクティビティフィールドを設定するか、自動プレゼンスを有効にすると、プレゼンス更新が適用されます。

    ステータスのみの例：

```json5
{
  channels: {
    discord: {
      status: "idle",
    },
  },
}
```

    アクティビティの例（カスタムステータスはデフォルトのアクティビティタイプ）：

```json5
{
  channels: {
    discord: {
      activity: "Focus time",
      activityType: 4,
    },
  },
}
```

    ストリーミングの例：

```json5
{
  channels: {
    discord: {
      activity: "Live coding",
      activityType: 1,
      activityUrl: "https://twitch.tv/openclaw",
    },
  },
}
```

    アクティビティタイプマップ：

    - 0: Playing
    - 1: Streaming（`activityUrl` が必要）
    - 2: Listening
    - 3: Watching
    - 4: Custom（アクティビティテキストをステータス状態として使用。絵文字はオプション）
    - 5: Competing

    自動プレゼンスの例（ランタイムヘルスシグナル）：

```json5
{
  channels: {
    discord: {
      autoPresence: {
        enabled: true,
        intervalMs: 30000,
        minUpdateIntervalMs: 15000,
        exhaustedText: "token exhausted",
      },
    },
  },
}
```

    自動プレゼンスはランタイムの可用性を Discord ステータスにマップします：正常 => online、劣化または不明 => idle、枯渇または利用不可 => dnd。オプションのテキストオーバーライド：

    - `autoPresence.healthyText`
    - `autoPresence.degradedText`
    - `autoPresence.exhaustedText`（`{reason}` プレースホルダーをサポート）

  </Accordion>

  <Accordion title="Discord での実行承認">
    Discord は DM でボタンベースの実行承認をサポートし、オプションで発信元チャンネルに承認プロンプトを投稿できます。

    設定パス：

    - `channels.discord.execApprovals.enabled`
    - `channels.discord.execApprovals.approvers`
    - `channels.discord.execApprovals.target`（`dm` | `channel` | `both`、デフォルト：`dm`）
    - `agentFilter`、`sessionFilter`、`cleanupAfterResolve`

    `target` が `channel` または `both` の場合、承認プロンプトがチャンネルに表示されます。設定された承認者のみがボタンを使用できます。他のユーザーにはエフェメラルな拒否が返されます。承認プロンプトにはコマンドテキストが含まれるため、信頼できるチャンネルでのみチャンネル配信を有効にしてください。セッションキーからチャンネル ID を取得できない場合、OpenClaw は DM 配信にフォールバックします。

    このハンドラーのゲートウェイ認証は、他のゲートウェイクライアントと同じ共有資格情報解決コントラクトを使用します：

    - 環境変数優先のローカル認証（`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` 次に `gateway.auth.*`）
    - ローカルモードでは、`gateway.auth.*` が未設定の場合、`gateway.remote.*` をフォールバックとして使用可能
    - 該当する場合は `gateway.remote.*` 経由のリモートモードサポート
    - URL オーバーライドはオーバーライドセーフ：CLI オーバーライドは暗黙的な資格情報を再利用せず、環境変数オーバーライドは環境変数の資格情報のみを使用

    不明な承認 ID で承認が失敗する場合は、承認者リストと機能の有効化を確認してください。

    関連ドキュメント：[実行承認](/tools/exec-approvals)

  </Accordion>
</AccordionGroup>

## ツールとアクションゲート

Discord メッセージアクションには、メッセージング、チャンネル管理、モデレーション、プレゼンス、メタデータアクションが含まれます。

主要な例：

- メッセージング：`sendMessage`、`readMessages`、`editMessage`、`deleteMessage`、`threadReply`
- リアクション：`react`、`reactions`、`emojiList`
- モデレーション：`timeout`、`kick`、`ban`
- プレゼンス：`setPresence`

アクションゲートは `channels.discord.actions.*` に設定されています。

デフォルトのゲート動作：

| アクショングループ                                                                                                                                                       | デフォルト |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| reactions, messages, threads, pins, polls, search, memberInfo, roleInfo, channelInfo, channels, voiceStatus, events, stickers, emojiUploads, stickerUploads, permissions | 有効       |
| roles                                                                                                                                                                    | 無効       |
| moderation                                                                                                                                                               | 無効       |
| presence                                                                                                                                                                 | 無効       |

## コンポーネント v2 UI

OpenClaw は実行承認とクロスコンテキストマーカーに Discord コンポーネント v2 を使用します。Discord メッセージアクションはカスタム UI に `components` を受け付けることもできます（高度な使い方。Carbon コンポーネントインスタンスが必要）。レガシーの `embeds` は引き続き利用可能ですが、推奨されません。

- `channels.discord.ui.components.accentColor` は Discord コンポーネントコンテナで使用されるアクセントカラーを設定します（16 進数）。
- `channels.discord.accounts.<id>.ui.components.accentColor` でアカウントごとに設定します。
- コンポーネント v2 が存在する場合、`embeds` は無視されます。

例：

```json5
{
  channels: {
    discord: {
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
    },
  },
}
```

## ボイスチャンネル

OpenClaw はリアルタイムの継続的な会話のために Discord ボイスチャンネルに参加できます。これはボイスメッセージの添付ファイルとは別のものです。

要件：

- ネイティブコマンドを有効にします（`commands.native` または `channels.discord.commands.native`）。
- `channels.discord.voice` を設定します。
- bot はターゲットのボイスチャンネルで Connect + Speak 権限が必要です。

Discord 専用のネイティブコマンド `/vc join|leave|status` を使用してセッションを制御します。コマンドはアカウントのデフォルトエージェントを使用し、他の Discord コマンドと同じアローリストおよびグループポリシーのルールに従います。

自動参加の例：

```json5
{
  channels: {
    discord: {
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
    },
  },
}
```

注：

- `voice.tts` はボイス再生のみ `messages.tts` をオーバーライドします。
- ボイストランスクリプトのターンは Discord `allowFrom`（または `dm.allowFrom`）からオーナーステータスを取得します。非オーナーの話者はオーナー専用ツール（例：`gateway` と `cron`）にアクセスできません。
- ボイスはデフォルトで有効です。無効にするには `channels.discord.voice.enabled=false` を設定します。
- `voice.daveEncryption` と `voice.decryptionFailureTolerance` は `@discordjs/voice` の参加オプションに渡されます。
- `@discordjs/voice` の未設定デフォルトは `daveEncryption=true` と `decryptionFailureTolerance=24` です。
- OpenClaw は受信の復号化失敗を監視し、短時間に繰り返し失敗した場合にボイスチャンネルを退出/再参加して自動回復します。
- 受信ログに `DecryptionFailed(UnencryptedWhenPassthroughDisabled)` が繰り返し表示される場合、これは [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) で追跡されているアップストリームの `@discordjs/voice` 受信バグである可能性があります。

## ボイスメッセージ

Discord のボイスメッセージは波形プレビューを表示し、OGG/Opus オーディオとメタデータが必要です。OpenClaw は波形を自動的に生成しますが、オーディオファイルを検査・変換するには、ゲートウェイホストで `ffmpeg` と `ffprobe` が利用可能である必要があります。

要件と制約：

- **ローカルファイルパス**を提供してください（URL は拒否されます）。
- テキストコンテンツを省略してください（Discord は同じペイロードでテキスト + ボイスメッセージを許可しません）。
- あらゆるオーディオフォーマットを受け付けます。必要に応じて OpenClaw が OGG/Opus に変換します。

例：

```bash
message(action="send", channel="discord", target="channel:123", path="/path/to/audio.mp3", asVoice=true)
```

## トラブルシューティング

<AccordionGroup>
  <Accordion title="許可されていないインテントが使用されたか、bot がギルドメッセージを受信できない">

    - Message Content Intent を有効にする
    - ユーザー/メンバーの解決に依存する場合は Server Members Intent を有効にする
    - インテントを変更した後はゲートウェイを再起動する

  </Accordion>

  <Accordion title="ギルドメッセージが予期せずブロックされる">

    - `groupPolicy` を確認する
    - `channels.discord.guilds` のギルドアローリストを確認する
    - ギルドの `channels` マップが存在する場合、リストされたチャンネルのみ許可されます
    - `requireMention` の動作とメンションパターンを確認する

    役立つチェック：

```bash
openclaw doctor
openclaw channels status --probe
openclaw logs --follow
```

  </Accordion>

  <Accordion title="requireMention が false でもブロックされる">
    一般的な原因：

    - ギルド/チャンネルのアローリストが一致しない `groupPolicy="allowlist"`
    - `requireMention` が間違った場所に設定されている（`channels.discord.guilds` またはチャンネルエントリの下である必要があります）
    - ギルド/チャンネルの `users` アローリストによって送信者がブロックされている

  </Accordion>

  <Accordion title="長時間実行ハンドラーがタイムアウトするか重複返信が発生する">

    典型的なログ：

    - `Listener DiscordMessageListener timed out after 30000ms for event MESSAGE_CREATE`
    - `Slow listener detected ...`
    - `discord inbound worker timed out after ...`

    リスナーバジェットの調整：

    - シングルアカウント：`channels.discord.eventQueue.listenerTimeout`
    - マルチアカウント：`channels.discord.accounts.<accountId>.eventQueue.listenerTimeout`

    ワーカー実行タイムアウトの調整：

    - シングルアカウント：`channels.discord.inboundWorker.runTimeoutMs`
    - マルチアカウント：`channels.discord.accounts.<accountId>.inboundWorker.runTimeoutMs`
    - デフォルト：`1800000`（30 分）。`0` で無効化

    推奨ベースライン：

```json5
{
  channels: {
    discord: {
      accounts: {
        default: {
          eventQueue: {
            listenerTimeout: 120000,
          },
          inboundWorker: {
            runTimeoutMs: 1800000,
          },
        },
      },
    },
  },
}
```

    リスナーの設定が遅い場合は `eventQueue.listenerTimeout` を使用し、`inboundWorker.runTimeoutMs` はキューに入ったエージェントターンに別の安全弁が必要な場合のみ使用してください。

  </Accordion>

  <Accordion title="権限監査の不一致">
    `channels status --probe` の権限チェックは数値チャンネル ID でのみ機能します。

    スラグキーを使用する場合、ランタイムマッチングは引き続き機能しますが、プローブは権限を完全に検証できません。

  </Accordion>

  <Accordion title="DM とペアリングの問題">

    - DM 無効：`channels.discord.dm.enabled=false`
    - DM ポリシー無効：`channels.discord.dmPolicy="disabled"`（レガシー：`channels.discord.dm.policy`）
    - `pairing` モードでのペアリング承認待ち

  </Accordion>

  <Accordion title="Bot 間のループ">
    デフォルトでは bot が作成したメッセージは無視されます。

    `channels.discord.allowBots=true` を設定する場合は、ループ動作を避けるために厳格なメンションとアローリストのルールを使用してください。
    bot をメンションした bot メッセージのみを受け付けるには `channels.discord.allowBots="mentions"` を優先してください。

  </Accordion>

  <Accordion title="ボイス STT が DecryptionFailed(...) でドロップする">

    - OpenClaw を最新の状態に保ちます（`openclaw update`）。Discord ボイス受信の回復ロジックが存在するようにします
    - `channels.discord.voice.daveEncryption=true`（デフォルト）を確認します
    - `channels.discord.voice.decryptionFailureTolerance=24`（アップストリームデフォルト）から始めて、必要な場合のみ調整します
    - 以下のログを監視します：
      - `discord voice: DAVE decrypt failures detected`
      - `discord voice: repeated decrypt failures; attempting rejoin`
    - 自動再参加後も失敗が続く場合は、ログを収集して [discord.js #11419](https://github.com/discordjs/discord.js/issues/11419) と比較してください

  </Accordion>
</AccordionGroup>

## 設定リファレンスポインター

主要リファレンス：

- [設定リファレンス - Discord](/gateway/configuration-reference#discord)

Discord の主要フィールド：

- スタートアップ/認証：`enabled`、`token`、`accounts.*`、`allowBots`
- ポリシー：`groupPolicy`、`dm.*`、`guilds.*`、`guilds.*.channels.*`
- コマンド：`commands.native`、`commands.useAccessGroups`、`configWrites`、`slashCommand.*`
- イベントキュー：`eventQueue.listenerTimeout`（リスナーバジェット）、`eventQueue.maxQueueSize`、`eventQueue.maxConcurrency`
- インバウンドワーカー：`inboundWorker.runTimeoutMs`
- 返信/履歴：`replyToMode`、`historyLimit`、`dmHistoryLimit`、`dms.*.historyLimit`
- 配信：`textChunkLimit`、`chunkMode`、`maxLinesPerMessage`
- ストリーミング：`streaming`（レガシーエイリアス：`streamMode`）、`draftChunk`、`blockStreaming`、`blockStreamingCoalesce`
- メディア/リトライ：`mediaMaxMb`、`retry`
  - `mediaMaxMb` は Discord のアウトバウンドアップロードに上限を設定します（デフォルト：`8MB`）
- アクション：`actions.*`
- プレゼンス：`activity`、`status`、`activityType`、`activityUrl`
- UI：`ui.components.accentColor`
- 機能：`threadBindings`、トップレベルの `bindings[]`（`type: "acp"`）、`pluralkit`、`execApprovals`、`intents`、`agentComponents`、`heartbeat`、`responsePrefix`

## 安全と運用

- bot トークンはシークレットとして扱います（管理された環境では `DISCORD_BOT_TOKEN` を推奨）。
- 最小権限の Discord 権限を付与します。
- コマンドのデプロイ/状態が古い場合は、ゲートウェイを再起動して `openclaw channels status --probe` で再確認します。

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [マルチエージェントルーティング](/concepts/multi-agent)
- [トラブルシューティング](/channels/troubleshooting)
- [スラッシュコマンド](/tools/slash-commands)
