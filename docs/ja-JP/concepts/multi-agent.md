---
summary: "マルチエージェントルーティング：分離されたエージェント、チャネルアカウント、バインディング"
title: マルチエージェントルーティング
read_when: "You want multiple isolated agents (workspaces + auth) in one gateway process."
status: active
---

# マルチエージェントルーティング

目標：1つの実行中のゲートウェイで、複数の_分離された_エージェント（個別のワークスペース＋`agentDir`＋セッション）と複数のチャネルアカウント（例：2つのWhatsApp）を実現する。インバウンドはバインディングを介してエージェントにルーティングされます。

## 「1つのエージェント」とは？

**エージェント**は、独自の以下を持つ完全にスコープされたブレインです：

- **ワークスペース**（ファイル、AGENTS.md/SOUL.md/USER.md、ローカルメモ、ペルソナルール）。
- **ステートディレクトリ**（`agentDir`）：認証プロファイル、モデルレジストリ、エージェントごとの設定。
- **セッションストア**（チャット履歴＋ルーティング状態）：`~/.openclaw/agents/<agentId>/sessions`配下。

認証プロファイルは**エージェントごと**です。各エージェントは独自の以下から読み取ります：

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

メインエージェントの認証情報は**自動的に共有されません**。エージェント間で`agentDir`を再利用しないでください（認証/セッションの衝突を引き起こします）。認証情報を共有したい場合は、`auth-profiles.json`を他のエージェントの`agentDir`にコピーしてください。

スキルは各ワークスペースの`skills/`フォルダを通じてエージェントごとに設定され、共有スキルは`~/.openclaw/skills`から利用可能です。[スキル：エージェントごと vs 共有](/tools/skills#per-agent-vs-shared-skills)を参照してください。

ゲートウェイは**1つのエージェント**（デフォルト）または**多数のエージェント**を並べてホストできます。

**ワークスペースの注意：** 各エージェントのワークスペースは**デフォルトのcwd**であり、ハードサンドボックスではありません。相対パスはワークスペース内で解決されますが、サンドボックスが有効でない限り、絶対パスは他のホスト上の場所に到達できます。[サンドボックス](/gateway/sandboxing)を参照してください。

## パス（クイックマップ）

- 設定：`~/.openclaw/openclaw.json`（または`OPENCLAW_CONFIG_PATH`）
- ステートディレクトリ：`~/.openclaw`（または`OPENCLAW_STATE_DIR`）
- ワークスペース：`~/.openclaw/workspace`（または`~/.openclaw/workspace-<agentId>`）
- エージェントディレクトリ：`~/.openclaw/agents/<agentId>/agent`（または`agents.list[].agentDir`）
- セッション：`~/.openclaw/agents/<agentId>/sessions`

### シングルエージェントモード（デフォルト）

何もしなければ、OpenClawは単一のエージェントを実行します：

- `agentId`のデフォルトは**`main`**。
- セッションは`agent:main:<mainKey>`としてキー付けされます。
- ワークスペースのデフォルトは`~/.openclaw/workspace`（`OPENCLAW_PROFILE`が設定されている場合は`~/.openclaw/workspace-<profile>`）。
- ステートのデフォルトは`~/.openclaw/agents/main/agent`。

## エージェントヘルパー

エージェントウィザードを使用して新しい分離されたエージェントを追加します：

```bash
openclaw agents add work
```

次に`bindings`を追加（またはウィザードに任せる）してインバウンドメッセージをルーティングします。

以下で確認します：

```bash
openclaw agents list --bindings
```

## クイックスタート

<Steps>
  <Step title="各エージェントワークスペースの作成">

ウィザードを使用するか、手動でワークスペースを作成します：

```bash
openclaw agents add coding
openclaw agents add social
```

各エージェントは`SOUL.md`、`AGENTS.md`、オプションの`USER.md`を含む独自のワークスペース、および`~/.openclaw/agents/<agentId>`配下の専用`agentDir`とセッションストアを取得します。

  </Step>

  <Step title="チャネルアカウントの作成">

希望するチャネルでエージェントごとに1つのアカウントを作成します：

- Discord：エージェントごとに1つのボット、Message Content Intentを有効にし、各トークンをコピー。
- Telegram：BotFatherを通じてエージェントごとに1つのボット、各トークンをコピー。
- WhatsApp：アカウントごとに各電話番号をリンク。

```bash
openclaw channels login --channel whatsapp --account work
```

チャネルガイドを参照：[Discord](/channels/discord)、[Telegram](/channels/telegram)、[WhatsApp](/channels/whatsapp)。

  </Step>

  <Step title="エージェント、アカウント、バインディングの追加">

`agents.list`にエージェントを、`channels.<channel>.accounts`にチャネルアカウントを追加し、`bindings`で接続します（以下の例）。

  </Step>

  <Step title="再起動と確認">

```bash
openclaw gateway restart
openclaw agents list --bindings
openclaw channels status --probe
```

  </Step>
</Steps>

## 複数エージェント = 複数の人、複数のパーソナリティ

**複数のエージェント**では、各`agentId`が**完全に分離されたペルソナ**になります：

- **異なる電話番号/アカウント**（チャネルごとの`accountId`）。
- **異なるパーソナリティ**（`AGENTS.md`や`SOUL.md`などのエージェントごとのワークスペースファイル）。
- **個別の認証＋セッション**（明示的に有効にしない限りクロストークなし）。

これにより**複数の人**が1つのゲートウェイサーバーを共有しながら、AIの「ブレイン」とデータを分離できます。

## 1つのWhatsApp番号、複数の人（DMスプリット）

**1つのWhatsAppアカウント**のまま、**異なるWhatsApp DM**を異なるエージェントにルーティングできます。送信者E.164（`+15551234567`など）を`peer.kind: "direct"`でマッチングします。返信は同じWhatsApp番号から送信されます（エージェントごとの送信者IDはありません）。

重要な詳細：ダイレクトチャットはエージェントの**メインセッションキー**に集約されるため、真の分離には**人ごとに1つのエージェント**が必要です。

例：

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

注意事項：

- DMアクセス制御はWhatsAppアカウントごとに**グローバル**（ペアリング/許可リスト）であり、エージェントごとではありません。
- 共有グループについては、グループを1つのエージェントにバインドするか、[ブロードキャストグループ](/channels/broadcast-groups)を使用してください。

## ルーティングルール（メッセージがエージェントを選択する方法）

バインディングは**決定的**で**最も具体的なものが優先**されます：

1. `peer`マッチ（正確なDM/グループ/チャネルID）
2. `parentPeer`マッチ（スレッド継承）
3. `guildId + roles`（Discordロールルーティング）
4. `guildId`（Discord）
5. `teamId`（Slack）
6. チャネルの`accountId`マッチ
7. チャネルレベルマッチ（`accountId: "*"`）
8. デフォルトエージェントにフォールバック（`agents.list[].default`、それ以外はリストの最初のエントリ、デフォルト：`main`）

同じティアで複数のバインディングがマッチする場合、設定順で最初のものが優先されます。
バインディングが複数のマッチフィールドを設定する場合（例：`peer`＋`guildId`）、指定されたすべてのフィールドが必要です（`AND`セマンティクス）。

重要なアカウントスコープの詳細：

- `accountId`を省略するバインディングはデフォルトアカウントのみにマッチします。
- すべてのアカウントにまたがるチャネル全体のフォールバックには`accountId: "*"`を使用してください。
- 後で同じエージェントに対して明示的なアカウントIDで同じバインディングを追加すると、OpenClawは重複する代わりに既存のチャネルのみのバインディングをアカウントスコープにアップグレードします。

## 複数アカウント / 電話番号

**複数アカウント**をサポートするチャネル（例：WhatsApp）は、各ログインを識別するために`accountId`を使用します。各`accountId`を異なるエージェントにルーティングでき、セッションを混在させることなく1つのサーバーで複数の電話番号をホストできます。

`accountId`が省略された場合のチャネル全体のデフォルトアカウントが必要な場合は、`channels.<channel>.defaultAccount`を設定してください（オプション）。未設定の場合、OpenClawは`default`が存在すればそれにフォールバックし、そうでなければ最初の設定済みアカウントID（ソート順）にフォールバックします。

このパターンをサポートする一般的なチャネル：

- `whatsapp`、`telegram`、`discord`、`slack`、`signal`、`imessage`
- `irc`、`line`、`googlechat`、`mattermost`、`matrix`、`nextcloud-talk`
- `bluebubbles`、`zalo`、`zalouser`、`nostr`、`feishu`

## コンセプト

- `agentId`：1つの「ブレイン」（ワークスペース、エージェントごとの認証、エージェントごとのセッションストア）。
- `accountId`：1つのチャネルアカウントインスタンス（例：WhatsAppアカウント`"personal"`対`"biz"`）。
- `binding`：`(channel, accountId, peer)`およびオプションのguild/team IDによってインバウンドメッセージを`agentId`にルーティング。
- ダイレクトチャットは`agent:<agentId>:<mainKey>`に集約（エージェントごとの「main」、`session.mainKey`）。

## プラットフォーム例

### エージェントごとのDiscordボット

各Discordボットアカウントは一意の`accountId`にマッピングされます。各アカウントをエージェントにバインドし、ボットごとに許可リストを保持します。

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "coding", workspace: "~/.openclaw/workspace-coding" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "discord", accountId: "default" } },
    { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
  ],
  channels: {
    discord: {
      groupPolicy: "allowlist",
      accounts: {
        default: {
          token: "DISCORD_BOT_TOKEN_MAIN",
          guilds: {
            "123456789012345678": {
              channels: {
                "222222222222222222": { allow: true, requireMention: false },
              },
            },
          },
        },
        coding: {
          token: "DISCORD_BOT_TOKEN_CODING",
          guilds: {
            "123456789012345678": {
              channels: {
                "333333333333333333": { allow: true, requireMention: false },
              },
            },
          },
        },
      },
    },
  },
}
```

注意事項：

- 各ボットをギルドに招待し、Message Content Intentを有効にしてください。
- トークンは`channels.discord.accounts.<id>.token`にあります（デフォルトアカウントは`DISCORD_BOT_TOKEN`を使用可能）。

### エージェントごとのTelegramボット

```json5
{
  agents: {
    list: [
      { id: "main", workspace: "~/.openclaw/workspace-main" },
      { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
    ],
  },
  bindings: [
    { agentId: "main", match: { channel: "telegram", accountId: "default" } },
    { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
  ],
  channels: {
    telegram: {
      accounts: {
        default: {
          botToken: "123456:ABC...",
          dmPolicy: "pairing",
        },
        alerts: {
          botToken: "987654:XYZ...",
          dmPolicy: "allowlist",
          allowFrom: ["tg:123456789"],
        },
      },
    },
  },
}
```

注意事項：

- BotFatherでエージェントごとに1つのボットを作成し、各トークンをコピーしてください。
- トークンは`channels.telegram.accounts.<id>.botToken`にあります（デフォルトアカウントは`TELEGRAM_BOT_TOKEN`を使用可能）。

### エージェントごとのWhatsApp番号

ゲートウェイを起動する前に各アカウントをリンクしてください：

```bash
openclaw channels login --channel whatsapp --account personal
openclaw channels login --channel whatsapp --account biz
```

`~/.openclaw/openclaw.json`（JSON5）：

```js
{
  agents: {
    list: [
      {
        id: "home",
        default: true,
        name: "Home",
        workspace: "~/.openclaw/workspace-home",
        agentDir: "~/.openclaw/agents/home/agent",
      },
      {
        id: "work",
        name: "Work",
        workspace: "~/.openclaw/workspace-work",
        agentDir: "~/.openclaw/agents/work/agent",
      },
    ],
  },

  // 決定的ルーティング：最初のマッチが優先（最も具体的なものを先に）。
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

    // オプションのピアごとオーバーライド（例：特定のグループをworkエージェントに送信）。
    {
      agentId: "work",
      match: {
        channel: "whatsapp",
        accountId: "personal",
        peer: { kind: "group", id: "1203630...@g.us" },
      },
    },
  ],

  // デフォルトではオフ：エージェント間メッセージングは明示的に有効化＋許可リストが必要。
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },

  channels: {
    whatsapp: {
      accounts: {
        personal: {
          // オプションのオーバーライド。デフォルト: ~/.openclaw/credentials/whatsapp/personal
          // authDir: "~/.openclaw/credentials/whatsapp/personal",
        },
        biz: {
          // オプションのオーバーライド。デフォルト: ~/.openclaw/credentials/whatsapp/biz
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

## 例：WhatsAppデイリーチャット＋Telegramディープワーク

チャネルで分割：WhatsAppを高速な日常エージェントに、TelegramをOpusエージェントにルーティングします。

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    { agentId: "chat", match: { channel: "whatsapp" } },
    { agentId: "opus", match: { channel: "telegram" } },
  ],
}
```

注意事項：

- チャネルに複数のアカウントがある場合、バインディングに`accountId`を追加してください（例：`{ channel: "whatsapp", accountId: "personal" }`）。
- 残りをchatに保持しながら1つのDM/グループをOpusにルーティングするには、そのピア用に`match.peer`バインディングを追加してください。ピアマッチは常にチャネル全体のルールに勝ちます。

## 例：同じチャネル、1つのピアをOpusに

WhatsAppを高速エージェントに保持しつつ、1つのDMをOpusにルーティング：

```json5
{
  agents: {
    list: [
      {
        id: "chat",
        name: "Everyday",
        workspace: "~/.openclaw/workspace-chat",
        model: "anthropic/claude-sonnet-4-5",
      },
      {
        id: "opus",
        name: "Deep Work",
        workspace: "~/.openclaw/workspace-opus",
        model: "anthropic/claude-opus-4-6",
      },
    ],
  },
  bindings: [
    {
      agentId: "opus",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551234567" } },
    },
    { agentId: "chat", match: { channel: "whatsapp" } },
  ],
}
```

ピアバインディングは常に優先されるため、チャネル全体のルールの上に配置してください。

## WhatsAppグループにバインドされたファミリーエージェント

専用のファミリーエージェントを1つのWhatsAppグループにバインドし、メンションゲーティングとより厳格なツールポリシーを設定：

```json5
{
  agents: {
    list: [
      {
        id: "family",
        name: "Family",
        workspace: "~/.openclaw/workspace-family",
        identity: { name: "Family Bot" },
        groupChat: {
          mentionPatterns: ["@family", "@familybot", "@Family Bot"],
        },
        sandbox: {
          mode: "all",
          scope: "agent",
        },
        tools: {
          allow: [
            "exec",
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
        },
      },
    ],
  },
  bindings: [
    {
      agentId: "family",
      match: {
        channel: "whatsapp",
        peer: { kind: "group", id: "120363999999999999@g.us" },
      },
    },
  ],
}
```

注意事項：

- ツールの許可/拒否リストは**ツール**であり、スキルではありません。スキルがバイナリを実行する必要がある場合、`exec`が許可されていてバイナリがサンドボックス内に存在することを確認してください。
- より厳格なゲーティングには、`agents.list[].groupChat.mentionPatterns`を設定し、チャネルのグループ許可リストを有効にしてください。

## エージェントごとのサンドボックスとツール設定

v2026.1.6以降、各エージェントは独自のサンドボックスとツール制限を持つことができます：

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // パーソナルエージェントにはサンドボックスなし
        },
        // ツール制限なし - すべてのツールが利用可能
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // 常にサンドボックス化
          scope: "agent",  // エージェントごとに1つのコンテナ
          docker: {
            // コンテナ作成後のオプションのワンタイムセットアップ
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // readツールのみ
          deny: ["exec", "write", "edit", "apply_patch"],    // 他を拒否
        },
      },
    ],
  },
}
```

注意：`setupCommand`は`sandbox.docker`配下にあり、コンテナ作成時に一度だけ実行されます。
解決されたスコープが`"shared"`の場合、エージェントごとの`sandbox.docker.*`オーバーライドは無視されます。

**利点：**

- **セキュリティ分離**：信頼できないエージェントのツールを制限
- **リソース制御**：特定のエージェントをサンドボックス化しながら他をホスト上に保持
- **柔軟なポリシー**：エージェントごとに異なる権限

注意：`tools.elevated`は**グローバル**で送信者ベースです。エージェントごとには設定できません。
エージェントごとの境界が必要な場合は、`agents.list[].tools`を使用して`exec`を拒否してください。
グループターゲティングには、`agents.list[].groupChat.mentionPatterns`を使用して@メンションが意図したエージェントにクリーンにマッピングされるようにしてください。

詳細な例については[マルチエージェントサンドボックス＆ツール](/tools/multi-agent-sandbox-tools)を参照してください。
