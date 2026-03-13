---
summary: "`openclaw` コマンド、サブコマンド、オプションの OpenClaw CLI リファレンス"
read_when:
  - CLI コマンドやオプションの追加・変更時
  - 新しいコマンドサーフェスのドキュメント作成時
title: "CLI リファレンス"
---

# CLI リファレンス

このページでは現在の CLI の動作について説明します。コマンドが変更された場合は、このドキュメントを更新してください。

## コマンドページ

- [`setup`](/cli/setup)
- [`onboard`](/cli/onboard)
- [`configure`](/cli/configure)
- [`config`](/cli/config)
- [`completion`](/cli/completion)
- [`doctor`](/cli/doctor)
- [`dashboard`](/cli/dashboard)
- [`reset`](/cli/reset)
- [`uninstall`](/cli/uninstall)
- [`update`](/cli/update)
- [`message`](/cli/message)
- [`agent`](/cli/agent)
- [`agents`](/cli/agents)
- [`acp`](/cli/acp)
- [`status`](/cli/status)
- [`health`](/cli/health)
- [`sessions`](/cli/sessions)
- [`gateway`](/cli/gateway)
- [`logs`](/cli/logs)
- [`system`](/cli/system)
- [`models`](/cli/models)
- [`memory`](/cli/memory)
- [`directory`](/cli/directory)
- [`nodes`](/cli/nodes)
- [`devices`](/cli/devices)
- [`node`](/cli/node)
- [`approvals`](/cli/approvals)
- [`sandbox`](/cli/sandbox)
- [`tui`](/cli/tui)
- [`browser`](/cli/browser)
- [`cron`](/cli/cron)
- [`dns`](/cli/dns)
- [`docs`](/cli/docs)
- [`hooks`](/cli/hooks)
- [`webhooks`](/cli/webhooks)
- [`pairing`](/cli/pairing)
- [`qr`](/cli/qr)
- [`plugins`](/cli/plugins)（プラグインコマンド）
- [`channels`](/cli/channels)
- [`security`](/cli/security)
- [`secrets`](/cli/secrets)
- [`skills`](/cli/skills)
- [`daemon`](/cli/daemon)（gateway サービスコマンドのレガシーエイリアス）
- [`clawbot`](/cli/clawbot)（レガシーエイリアス名前空間）
- [`voicecall`](/cli/voicecall)（プラグイン；インストール済みの場合）

## グローバルフラグ

- `--dev`: `~/.openclaw-dev` 配下に状態を分離し、デフォルトポートをシフトします。
- `--profile <name>`: `~/.openclaw-<name>` 配下に状態を分離します。
- `--no-color`: ANSI カラーを無効にします。
- `--update`: `openclaw update` のショートハンド（ソースインストールのみ）。
- `-V`, `--version`, `-v`: バージョンを表示して終了します。

## 出力スタイル

- ANSI カラーとプログレスインジケーターは TTY セッションでのみレンダリングされます。
- OSC-8 ハイパーリンクは対応ターミナルでクリック可能なリンクとして表示されます。それ以外ではプレーン URL にフォールバックします。
- `--json`（および対応する場合は `--plain`）でスタイリングを無効にし、クリーンな出力を得られます。
- `--no-color` で ANSI スタイリングを無効にします。`NO_COLOR=1` も認識されます。
- 長時間実行コマンドはプログレスインジケーターを表示します（対応時は OSC 9;4）。

## カラーパレット

OpenClaw は CLI 出力にロブスターパレットを使用します。

- `accent` (#FF5A2D): 見出し、ラベル、プライマリハイライト。
- `accentBright` (#FF7A3D): コマンド名、強調。
- `accentDim` (#D14A22): セカンダリハイライトテキスト。
- `info` (#FF8A5B): 情報表示値。
- `success` (#2FBF71): 成功状態。
- `warn` (#FFB020): 警告、フォールバック、注意。
- `error` (#E23D2D): エラー、失敗。
- `muted` (#8B7F77): 控えめ表示、メタデータ。

パレットの信頼できるソース: `src/terminal/palette.ts`（別名「lobster seam」）。

## コマンドツリー

```
openclaw [--dev] [--profile <name>] <command>
  setup
  onboard
  configure
  config
    get
    set
    unset
  completion
  doctor
  dashboard
  security
    audit
  secrets
    reload
    migrate
  reset
  uninstall
  update
  channels
    list
    status
    logs
    add
    remove
    login
    logout
  directory
  skills
    list
    info
    check
  plugins
    list
    info
    install
    enable
    disable
    doctor
  memory
    status
    index
    search
  message
  agent
  agents
    list
    add
    delete
  acp
  status
  health
  sessions
  gateway
    call
    health
    status
    probe
    discover
    install
    uninstall
    start
    stop
    restart
    run
  daemon
    status
    install
    uninstall
    start
    stop
    restart
  logs
  system
    event
    heartbeat last|enable|disable
    presence
  models
    list
    status
    set
    set-image
    aliases list|add|remove
    fallbacks list|add|remove|clear
    image-fallbacks list|add|remove|clear
    scan
    auth add|setup-token|paste-token
    auth order get|set|clear
  sandbox
    list
    recreate
    explain
  cron
    status
    list
    add
    edit
    rm
    enable
    disable
    runs
    run
  nodes
  devices
  node
    run
    status
    install
    uninstall
    start
    stop
    restart
  approvals
    get
    set
    allowlist add|remove
  browser
    status
    start
    stop
    reset-profile
    tabs
    open
    focus
    close
    profiles
    create-profile
    delete-profile
    screenshot
    snapshot
    navigate
    resize
    click
    type
    press
    hover
    drag
    select
    upload
    fill
    dialog
    wait
    evaluate
    console
    pdf
  hooks
    list
    info
    check
    enable
    disable
    install
    update
  webhooks
    gmail setup|run
  pairing
    list
    approve
  qr
  clawbot
    qr
  docs
  dns
    setup
  tui
```

注: プラグインは追加のトップレベルコマンドを追加できます（例: `openclaw voicecall`）。

## セキュリティ

- `openclaw security audit` — 設定とローカル状態の一般的なセキュリティ問題を監査します。
- `openclaw security audit --deep` — ベストエフォートで実行中の Gateway をプローブします。
- `openclaw security audit --fix` — 安全なデフォルトを適用し、状態/設定ファイルの chmod を実行します。

## シークレット

- `openclaw secrets reload` — 参照を再解決し、ランタイムスナップショットをアトミックにスワップします。
- `openclaw secrets audit` — プレーンテキストの残留、未解決の参照、優先順位のドリフトをスキャンします。
- `openclaw secrets configure` — プロバイダーセットアップ + SecretRef マッピング + プリフライト/適用のインタラクティブヘルパーです。
- `openclaw secrets apply --from <plan.json>` — 事前に生成されたプランを適用します（`--dry-run` 対応）。

## プラグイン

拡張機能とその設定を管理します:

- `openclaw plugins list` — プラグインを検出します（マシン出力には `--json` を使用）。
- `openclaw plugins info <id>` — プラグインの詳細を表示します。
- `openclaw plugins install <path|.tgz|npm-spec>` — プラグインをインストールします（またはプラグインパスを `plugins.load.paths` に追加）。
- `openclaw plugins enable <id>` / `disable <id>` — `plugins.entries.<id>.enabled` を切り替えます。
- `openclaw plugins doctor` — プラグインのロードエラーを報告します。

ほとんどのプラグイン変更には gateway の再起動が必要です。[/plugin](/tools/plugin) を参照してください。

## メモリ

`MEMORY.md` + `memory/*.md` に対するベクトル検索:

- `openclaw memory status` — インデックスの統計を表示します。
- `openclaw memory index` — メモリファイルを再インデックスします。
- `openclaw memory search "<query>"`（または `--query "<query>"`）— メモリに対するセマンティック検索を実行します。

## チャットスラッシュコマンド

チャットメッセージは `/...` コマンド（テキストおよびネイティブ）をサポートします。[/tools/slash-commands](/tools/slash-commands) を参照してください。

ハイライト:

- `/status` で簡易診断。
- `/config` で永続化される設定変更。
- `/debug` でランタイム限定の設定オーバーライド（メモリのみ、ディスクには保存されません。`commands.debug: true` が必要）。

## セットアップとオンボーディング

### `setup`

設定とワークスペースを初期化します。

オプション:

- `--workspace <dir>`: エージェントのワークスペースパス（デフォルト `~/.openclaw/workspace`）。
- `--wizard`: オンボーディングウィザードを実行します。
- `--non-interactive`: プロンプトなしでウィザードを実行します。
- `--mode <local|remote>`: ウィザードモード。
- `--remote-url <url>`: リモート Gateway URL。
- `--remote-token <token>`: リモート Gateway トークン。

ウィザードフラグ（`--non-interactive`、`--mode`、`--remote-url`、`--remote-token`）が指定されると、ウィザードが自動実行されます。

### `onboard`

Gateway、ワークスペース、スキルのセットアップ用インタラクティブウィザードです。

オプション:

- `--workspace <dir>`
- `--reset`（ウィザード前に設定 + 認証情報 + セッションをリセット）
- `--reset-scope <config|config+creds+sessions|full>`（デフォルト `config+creds+sessions`。ワークスペースも削除するには `full` を使用）
- `--non-interactive`
- `--mode <local|remote>`
- `--flow <quickstart|advanced|manual>`（manual は advanced のエイリアス）
- `--auth-choice <setup-token|token|chutes|openai-codex|openai-api-key|openrouter-api-key|ai-gateway-api-key|moonshot-api-key|moonshot-api-key-cn|kimi-code-api-key|synthetic-api-key|venice-api-key|gemini-api-key|zai-api-key|mistral-api-key|apiKey|minimax-api|minimax-api-lightning|opencode-zen|custom-api-key|skip>`
- `--token-provider <id>`（非インタラクティブ。`--auth-choice token` と共に使用）
- `--token <token>`（非インタラクティブ。`--auth-choice token` と共に使用）
- `--token-profile-id <id>`（非インタラクティブ。デフォルト: `<provider>:manual`）
- `--token-expires-in <duration>`（非インタラクティブ。例: `365d`、`12h`）
- `--secret-input-mode <plaintext|ref>`（デフォルト `plaintext`。プレーンテキストキーの代わりにプロバイダーのデフォルト env ref を保存するには `ref` を使用）
- `--anthropic-api-key <key>`
- `--openai-api-key <key>`
- `--mistral-api-key <key>`
- `--openrouter-api-key <key>`
- `--ai-gateway-api-key <key>`
- `--moonshot-api-key <key>`
- `--kimi-code-api-key <key>`
- `--gemini-api-key <key>`
- `--zai-api-key <key>`
- `--minimax-api-key <key>`
- `--opencode-zen-api-key <key>`
- `--custom-base-url <url>`（非インタラクティブ。`--auth-choice custom-api-key` と共に使用）
- `--custom-model-id <id>`（非インタラクティブ。`--auth-choice custom-api-key` と共に使用）
- `--custom-api-key <key>`（非インタラクティブ。オプション。`--auth-choice custom-api-key` と共に使用。省略時は `CUSTOM_API_KEY` にフォールバック）
- `--custom-provider-id <id>`（非インタラクティブ。オプションのカスタムプロバイダー ID）
- `--custom-compatibility <openai|anthropic>`（非インタラクティブ。オプション。デフォルト `openai`）
- `--gateway-port <port>`
- `--gateway-bind <loopback|lan|tailnet|auto|custom>`
- `--gateway-auth <token|password>`
- `--gateway-token <token>`
- `--gateway-token-ref-env <name>`（非インタラクティブ。`gateway.auth.token` を env SecretRef として保存。その環境変数が設定されている必要あり。`--gateway-token` と併用不可）
- `--gateway-password <password>`
- `--remote-url <url>`
- `--remote-token <token>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--install-daemon`
- `--no-install-daemon`（エイリアス: `--skip-daemon`）
- `--daemon-runtime <node|bun>`
- `--skip-channels`
- `--skip-skills`
- `--skip-health`
- `--skip-ui`
- `--node-manager <npm|pnpm|bun>`（pnpm 推奨。bun は Gateway ランタイムには非推奨）
- `--json`

### `configure`

インタラクティブな設定ウィザード（モデル、チャンネル、スキル、gateway）。

### `config`

非インタラクティブな設定ヘルパー（get/set/unset/file/validate）。サブコマンドなしで `openclaw config` を実行するとウィザードが起動します。

サブコマンド:

- `config get <path>`: 設定値を表示します（ドット/ブラケット記法パス）。
- `config set <path> <value>`: 値を設定します（JSON5 またはプレーン文字列）。
- `config unset <path>`: 値を削除します。
- `config file`: アクティブな設定ファイルパスを表示します。
- `config validate`: gateway を起動せずに、現在の設定をスキーマに対して検証します。
- `config validate --json`: マシンリーダブルな JSON 出力を出力します。

### `doctor`

ヘルスチェックと簡易修正（設定 + gateway + レガシーサービス）。

オプション:

- `--no-workspace-suggestions`: ワークスペースメモリのヒントを無効にします。
- `--yes`: プロンプトなしでデフォルトを受け入れます（ヘッドレス）。
- `--non-interactive`: プロンプトをスキップし、安全なマイグレーションのみ適用します。
- `--deep`: システムサービスをスキャンして追加の gateway インストールを検出します。

## チャンネルヘルパー

### `channels`

チャットチャンネルアカウントの管理（WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost（プラグイン）/Signal/iMessage/MS Teams）。

サブコマンド:

- `channels list`: 設定済みのチャンネルと認証プロファイルを表示します。
- `channels status`: gateway の到達性とチャンネルの健全性を確認します（`--probe` で追加チェックを実行。gateway のヘルスプローブには `openclaw health` または `openclaw status --deep` を使用）。
- ヒント: `channels status` は一般的な設定ミスを検出した場合、修正案付きの警告を表示します（その後 `openclaw doctor` を案内します）。
- `channels logs`: gateway ログファイルから最近のチャンネルログを表示します。
- `channels add`: フラグなしの場合はウィザード形式のセットアップ。フラグを指定すると非インタラクティブモードに切り替わります。
  - まだシングルアカウントのトップレベル設定を使用しているチャンネルにデフォルト以外のアカウントを追加する場合、OpenClaw は新しいアカウントを書き込む前にアカウントスコープの値を `channels.<channel>.accounts.default` に移動します。
  - 非インタラクティブの `channels add` ではバインディングの自動作成/アップグレードは行われません。チャンネル限定のバインディングは引き続きデフォルトアカウントにマッチします。
- `channels remove`: デフォルトでは無効化のみ。プロンプトなしで設定エントリを削除するには `--delete` を指定します。
- `channels login`: インタラクティブなチャンネルログイン（WhatsApp Web のみ）。
- `channels logout`: チャンネルセッションからログアウトします（対応している場合）。

共通オプション:

- `--channel <name>`: `whatsapp|telegram|discord|googlechat|slack|mattermost|signal|imessage|msteams`
- `--account <id>`: チャンネルアカウント ID（デフォルト `default`）
- `--name <label>`: アカウントの表示名

`channels login` オプション:

- `--channel <channel>`（デフォルト `whatsapp`。`whatsapp`/`web` をサポート）
- `--account <id>`
- `--verbose`

`channels logout` オプション:

- `--channel <channel>`（デフォルト `whatsapp`）
- `--account <id>`

`channels list` オプション:

- `--no-usage`: モデルプロバイダーの使用量/クォータスナップショットをスキップします（OAuth/API ベースのみ）。
- `--json`: JSON を出力します（`--no-usage` が設定されていない限り使用量を含みます）。

`channels logs` オプション:

- `--channel <name|all>`（デフォルト `all`）
- `--lines <n>`（デフォルト `200`）
- `--json`

詳細: [/concepts/oauth](/concepts/oauth)

例:

```bash
openclaw channels add --channel telegram --account alerts --name "Alerts Bot" --token $TELEGRAM_BOT_TOKEN
openclaw channels add --channel discord --account work --name "Work Bot" --token $DISCORD_BOT_TOKEN
openclaw channels remove --channel discord --account work --delete
openclaw channels status --probe
openclaw status --deep
```

### `skills`

利用可能なスキルと準備状況の情報をリスト・検査します。

サブコマンド:

- `skills list`: スキルの一覧を表示します（サブコマンドなしの場合のデフォルト）。
- `skills info <name>`: 1 つのスキルの詳細を表示します。
- `skills check`: 準備完了と要件不足のサマリーを表示します。

オプション:

- `--eligible`: 準備完了のスキルのみ表示します。
- `--json`: JSON を出力します（スタイリングなし）。
- `-v`, `--verbose`: 不足している要件の詳細を含めます。

ヒント: `npx clawhub` を使用して、スキルの検索、インストール、同期を行えます。

### `pairing`

チャンネル間の DM ペアリングリクエストを承認します。

サブコマンド:

- `pairing list [channel] [--channel <channel>] [--account <id>] [--json]`
- `pairing approve <channel> <code> [--account <id>] [--notify]`
- `pairing approve --channel <channel> [--account <id>] <code> [--notify]`

### `devices`

Gateway デバイスペアリングエントリとロールごとのデバイストークンを管理します。

サブコマンド:

- `devices list [--json]`
- `devices approve [requestId] [--latest]`
- `devices reject <requestId>`
- `devices remove <deviceId>`
- `devices clear --yes [--pending]`
- `devices rotate --device <id> --role <role> [--scope <scope...>]`
- `devices revoke --device <id> --role <role>`

### `webhooks gmail`

Gmail Pub/Sub フックのセットアップとランナー。[/automation/gmail-pubsub](/automation/gmail-pubsub) を参照してください。

サブコマンド:

- `webhooks gmail setup`（`--account <email>` が必要。`--project`、`--topic`、`--subscription`、`--label`、`--hook-url`、`--hook-token`、`--push-token`、`--bind`、`--port`、`--path`、`--include-body`、`--max-bytes`、`--renew-minutes`、`--tailscale`、`--tailscale-path`、`--tailscale-target`、`--push-endpoint`、`--json` をサポート）
- `webhooks gmail run`（同じフラグのランタイムオーバーライド）

### `dns setup`

ワイドエリアディスカバリー DNS ヘルパー（CoreDNS + Tailscale）。[/gateway/discovery](/gateway/discovery) を参照してください。

オプション:

- `--apply`: CoreDNS 設定をインストール/更新します（sudo が必要。macOS のみ）。

## メッセージングとエージェント

### `message`

統合的なアウトバウンドメッセージング + チャンネルアクション。

参照: [/cli/message](/cli/message)

サブコマンド:

- `message send|poll|react|reactions|read|edit|delete|pin|unpin|pins|permissions|search|timeout|kick|ban`
- `message thread <create|list|reply>`
- `message emoji <list|upload>`
- `message sticker <send|upload>`
- `message role <info|add|remove>`
- `message channel <info|list>`
- `message member info`
- `message voice status`
- `message event <list|create>`

例:

- `openclaw message send --target +15555550123 --message "Hi"`
- `openclaw message poll --channel discord --target channel:123 --poll-question "Snack?" --poll-option Pizza --poll-option Sushi`

### `agent`

Gateway（または `--local` 埋め込み）経由でエージェントを 1 ターン実行します。

必須:

- `--message <text>`

オプション:

- `--to <dest>`（セッションキーおよびオプションの配信先）
- `--session-id <id>`
- `--thinking <off|minimal|low|medium|high|xhigh>`（GPT-5.2 + Codex モデルのみ）
- `--verbose <on|full|off>`
- `--channel <whatsapp|telegram|discord|slack|mattermost|signal|imessage|msteams>`
- `--local`
- `--deliver`
- `--json`
- `--timeout <seconds>`

### `agents`

分離されたエージェント（ワークスペース + 認証 + ルーティング）を管理します。

#### `agents list`

設定済みのエージェントの一覧を表示します。

オプション:

- `--json`
- `--bindings`

#### `agents add [name]`

新しい分離エージェントを追加します。フラグ（または `--non-interactive`）が指定されない限り、ガイド付きウィザードを実行します。非インタラクティブモードでは `--workspace` が必須です。

オプション:

- `--workspace <dir>`
- `--model <id>`
- `--agent-dir <dir>`
- `--bind <channel[:accountId]>`（繰り返し指定可能）
- `--non-interactive`
- `--json`

バインディングの指定は `channel[:accountId]` の形式を使用します。`accountId` を省略すると、OpenClaw はチャンネルのデフォルト/プラグインフック経由でアカウントスコープを解決する場合があります。そうでない場合は、明示的なアカウントスコープなしのチャンネルバインディングとなります。

#### `agents bindings`

ルーティングバインディングの一覧を表示します。

オプション:

- `--agent <id>`
- `--json`

#### `agents bind`

エージェントにルーティングバインディングを追加します。

オプション:

- `--agent <id>`
- `--bind <channel[:accountId]>`（繰り返し指定可能）
- `--json`

#### `agents unbind`

エージェントのルーティングバインディングを削除します。

オプション:

- `--agent <id>`
- `--bind <channel[:accountId]>`（繰り返し指定可能）
- `--all`
- `--json`

#### `agents delete <id>`

エージェントを削除し、そのワークスペースと状態をクリーンアップします。

オプション:

- `--force`
- `--json`

### `acp`

IDE を Gateway に接続する ACP ブリッジを実行します。

完全なオプションと例については [`acp`](/cli/acp) を参照してください。

### `status`

リンクされたセッションの健全性と最近の受信者を表示します。

オプション:

- `--json`
- `--all`（完全な診断。読み取り専用、貼り付け可能）
- `--deep`（チャンネルをプローブ）
- `--usage`（モデルプロバイダーの使用量/クォータを表示）
- `--timeout <ms>`
- `--verbose`
- `--debug`（`--verbose` のエイリアス）

備考:

- 概要には、利用可能な場合、Gateway + ノードホストサービスの状態が含まれます。

### 使用量トラッキング

OAuth/API 認証情報が利用可能な場合、OpenClaw はプロバイダーの使用量/クォータを表示できます。

表示場所:

- `/status`（利用可能な場合、短いプロバイダー使用量行を追加）
- `openclaw status --usage`（完全なプロバイダー内訳を表示）
- macOS メニューバー（コンテキスト配下の使用量セクション）

備考:

- データはプロバイダーの使用量エンドポイントから直接取得されます（推定値ではありません）。
- プロバイダー: Anthropic、GitHub Copilot、OpenAI Codex OAuth、さらに Gemini CLI/Antigravity（これらのプロバイダープラグインが有効な場合）。
- 一致する認証情報が存在しない場合、使用量は非表示になります。
- 詳細: [Usage tracking](/concepts/usage-tracking) を参照。

### `health`

実行中の Gateway からヘルス情報を取得します。

オプション:

- `--json`
- `--timeout <ms>`
- `--verbose`

### `sessions`

保存された会話セッションの一覧を表示します。

オプション:

- `--json`
- `--verbose`
- `--store <path>`
- `--active <minutes>`

## リセット / アンインストール

### `reset`

ローカルの設定/状態をリセットします（CLI はインストールされたまま残ります）。

オプション:

- `--scope <config|config+creds+sessions|full>`
- `--yes`
- `--non-interactive`
- `--dry-run`

備考:

- `--non-interactive` には `--scope` と `--yes` が必要です。

### `uninstall`

gateway サービスとローカルデータをアンインストールします（CLI は残ります）。

オプション:

- `--service`
- `--state`
- `--workspace`
- `--app`
- `--all`
- `--yes`
- `--non-interactive`
- `--dry-run`

備考:

- `--non-interactive` には `--yes` と明示的なスコープ（または `--all`）が必要です。

## Gateway

### `gateway`

WebSocket Gateway を実行します。

オプション:

- `--port <port>`
- `--bind <loopback|tailnet|lan|auto|custom>`
- `--token <token>`
- `--auth <token|password>`
- `--password <password>`
- `--tailscale <off|serve|funnel>`
- `--tailscale-reset-on-exit`
- `--allow-unconfigured`
- `--dev`
- `--reset`（dev の設定 + 認証情報 + セッション + ワークスペースをリセット）
- `--force`（ポート上の既存リスナーを強制終了）
- `--verbose`
- `--claude-cli-logs`
- `--ws-log <auto|full|compact>`
- `--compact`（`--ws-log compact` のエイリアス）
- `--raw-stream`
- `--raw-stream-path <path>`

### `gateway service`

Gateway サービスの管理（launchd/systemd/schtasks）。

サブコマンド:

- `gateway status`（デフォルトで Gateway RPC をプローブ）
- `gateway install`（サービスのインストール）
- `gateway uninstall`
- `gateway start`
- `gateway stop`
- `gateway restart`

備考:

- `gateway status` はデフォルトで、サービスの解決済みポート/設定を使用して Gateway RPC をプローブします（`--url/--token/--password` でオーバーライド可能）。
- `gateway status` はスクリプティング用に `--no-probe`、`--deep`、`--json` をサポートします。
- `gateway status` は検出可能な場合、レガシーまたは追加の gateway サービスも表示します（`--deep` でシステムレベルのスキャンを追加）。プロファイル名付きの OpenClaw サービスはファーストクラスとして扱われ、「追加」としてフラグ付けされません。
- `gateway status` は CLI が使用する設定パスとサービスが使用する可能性のある設定（サービス env）、および解決済みのプローブターゲット URL を表示します。
- `gateway install|uninstall|start|stop|restart` はスクリプティング用に `--json` をサポートします（デフォルト出力は人間向けのまま）。
- `gateway install` のデフォルトランタイムは Node です。bun は**非推奨**です（WhatsApp/Telegram のバグ）。
- `gateway install` オプション: `--port`、`--runtime`、`--token`、`--force`、`--json`。

### `logs`

RPC 経由で Gateway ファイルログをテールします。

備考:

- TTY セッションでは色付きの構造化ビューをレンダリングします。非 TTY ではプレーンテキストにフォールバックします。
- `--json` は行区切り JSON を出力します（1 行に 1 ログイベント）。

例:

```bash
openclaw logs --follow
openclaw logs --limit 200
openclaw logs --plain
openclaw logs --json
openclaw logs --no-color
```

### `gateway <subcommand>`

Gateway CLI ヘルパー（RPC サブコマンドには `--url`、`--token`、`--password`、`--timeout`、`--expect-final` を使用）。
`--url` を指定すると、CLI は設定や環境の認証情報を自動適用しません。
`--token` または `--password` を明示的に指定してください。明示的な認証情報がない場合はエラーとなります。

サブコマンド:

- `gateway call <method> [--params <json>]`
- `gateway health`
- `gateway status`
- `gateway probe`
- `gateway discover`
- `gateway install|uninstall|start|stop|restart`
- `gateway run`

一般的な RPC:

- `config.apply`（設定の検証 + 書き込み + 再起動 + ウェイク）
- `config.patch`（部分的な更新のマージ + 再起動 + ウェイク）
- `update.run`（更新の実行 + 再起動 + ウェイク）

ヒント: `config.set`/`config.apply`/`config.patch` を直接呼び出す場合、既に設定が存在する場合は `config.get` から `baseHash` を渡してください。

## モデル

フォールバック動作とスキャン戦略については [/concepts/models](/concepts/models) を参照してください。

Anthropic setup-token（サポート済み）:

```bash
claude setup-token
openclaw models auth setup-token --provider anthropic
openclaw models status
```

ポリシーに関する注意: これは技術的な互換性です。Anthropic は過去に Claude Code 以外でのサブスクリプション利用をブロックしたことがあります。本番環境で setup-token を使用する前に、現在の Anthropic の利用規約を確認してください。

### `models`（ルート）

`openclaw models` は `models status` のエイリアスです。

ルートオプション:

- `--status-json`（`models status --json` のエイリアス）
- `--status-plain`（`models status --plain` のエイリアス）

### `models list`

オプション:

- `--all`
- `--local`
- `--provider <name>`
- `--json`
- `--plain`

### `models status`

オプション:

- `--json`
- `--plain`
- `--check`（終了コード 1=期限切れ/未設定、2=まもなく期限切れ）
- `--probe`（設定済み認証プロファイルのライブプローブ）
- `--probe-provider <name>`
- `--probe-profile <id>`（繰り返しまたはカンマ区切り）
- `--probe-timeout <ms>`
- `--probe-concurrency <n>`
- `--probe-max-tokens <n>`

認証ストア内のプロファイルの認証概要と OAuth 有効期限ステータスを常に含みます。
`--probe` はライブリクエストを実行します（トークンを消費し、レート制限をトリガーする可能性があります）。

### `models set <model>`

`agents.defaults.model.primary` を設定します。

### `models set-image <model>`

`agents.defaults.imageModel.primary` を設定します。

### `models aliases list|add|remove`

オプション:

- `list`: `--json`、`--plain`
- `add <alias> <model>`
- `remove <alias>`

### `models fallbacks list|add|remove|clear`

オプション:

- `list`: `--json`、`--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models image-fallbacks list|add|remove|clear`

オプション:

- `list`: `--json`、`--plain`
- `add <model>`
- `remove <model>`
- `clear`

### `models scan`

オプション:

- `--min-params <b>`
- `--max-age-days <days>`
- `--provider <name>`
- `--max-candidates <n>`
- `--timeout <ms>`
- `--concurrency <n>`
- `--no-probe`
- `--yes`
- `--no-input`
- `--set-default`
- `--set-image`
- `--json`

### `models auth add|setup-token|paste-token`

オプション:

- `add`: インタラクティブ認証ヘルパー
- `setup-token`: `--provider <name>`（デフォルト `anthropic`）、`--yes`
- `paste-token`: `--provider <name>`、`--profile-id <id>`、`--expires-in <duration>`

### `models auth order get|set|clear`

オプション:

- `get`: `--provider <name>`、`--agent <id>`、`--json`
- `set`: `--provider <name>`、`--agent <id>`、`<profileIds...>`
- `clear`: `--provider <name>`、`--agent <id>`

## システム

### `system event`

システムイベントをキューに追加し、オプションでハートビートをトリガーします（Gateway RPC）。

必須:

- `--text <text>`

オプション:

- `--mode <now|next-heartbeat>`
- `--json`
- `--url`、`--token`、`--timeout`、`--expect-final`

### `system heartbeat last|enable|disable`

ハートビートコントロール（Gateway RPC）。

オプション:

- `--json`
- `--url`、`--token`、`--timeout`、`--expect-final`

### `system presence`

システムプレゼンスエントリの一覧を表示します（Gateway RPC）。

オプション:

- `--json`
- `--url`、`--token`、`--timeout`、`--expect-final`

## Cron

スケジュール済みジョブの管理（Gateway RPC）。[/automation/cron-jobs](/automation/cron-jobs) を参照してください。

サブコマンド:

- `cron status [--json]`
- `cron list [--all] [--json]`（デフォルトはテーブル出力。生データには `--json` を使用）
- `cron add`（エイリアス: `create`。`--name` および `--at` | `--every` | `--cron` のいずれか 1 つ、かつ `--system-event` | `--message` のいずれか 1 つのペイロードが必要）
- `cron edit <id>`（フィールドのパッチ）
- `cron rm <id>`（エイリアス: `remove`、`delete`）
- `cron enable <id>`
- `cron disable <id>`
- `cron runs --id <id> [--limit <n>]`
- `cron run <id> [--force]`

すべての `cron` コマンドは `--url`、`--token`、`--timeout`、`--expect-final` を受け付けます。

## ノードホスト

`node` は**ヘッドレスノードホスト**を実行するか、バックグラウンドサービスとして管理します。[`openclaw node`](/cli/node) を参照してください。

サブコマンド:

- `node run --host <gateway-host> --port 18789`
- `node status`
- `node install [--host <gateway-host>] [--port <port>] [--tls] [--tls-fingerprint <sha256>] [--node-id <id>] [--display-name <name>] [--runtime <node|bun>] [--force]`
- `node uninstall`
- `node stop`
- `node restart`

## ノード

`nodes` は Gateway と通信し、ペアリング済みノードを対象とします。[/nodes](/nodes) を参照してください。

共通オプション:

- `--url`、`--token`、`--timeout`、`--json`

サブコマンド:

- `nodes status [--connected] [--last-connected <duration>]`
- `nodes describe --node <id|name|ip>`
- `nodes list [--connected] [--last-connected <duration>]`
- `nodes pending`
- `nodes approve <requestId>`
- `nodes reject <requestId>`
- `nodes rename --node <id|name|ip> --name <displayName>`
- `nodes invoke --node <id|name|ip> --command <command> [--params <json>] [--invoke-timeout <ms>] [--idempotency-key <key>]`
- `nodes run --node <id|name|ip> [--cwd <path>] [--env KEY=VAL] [--command-timeout <ms>] [--needs-screen-recording] [--invoke-timeout <ms>] <command...>`（Mac ノードまたはヘッドレスノードホスト）
- `nodes notify --node <id|name|ip> [--title <text>] [--body <text>] [--sound <name>] [--priority <passive|active|timeSensitive>] [--delivery <system|overlay|auto>] [--invoke-timeout <ms>]`（Mac のみ）

カメラ:

- `nodes camera list --node <id|name|ip>`
- `nodes camera snap --node <id|name|ip> [--facing front|back|both] [--device-id <id>] [--max-width <px>] [--quality <0-1>] [--delay-ms <ms>] [--invoke-timeout <ms>]`
- `nodes camera clip --node <id|name|ip> [--facing front|back] [--device-id <id>] [--duration <ms|10s|1m>] [--no-audio] [--invoke-timeout <ms>]`

Canvas とスクリーン:

- `nodes canvas snapshot --node <id|name|ip> [--format png|jpg|jpeg] [--max-width <px>] [--quality <0-1>] [--invoke-timeout <ms>]`
- `nodes canvas present --node <id|name|ip> [--target <urlOrPath>] [--x <px>] [--y <px>] [--width <px>] [--height <px>] [--invoke-timeout <ms>]`
- `nodes canvas hide --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas navigate <url> --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes canvas eval [<js>] --node <id|name|ip> [--js <code>] [--invoke-timeout <ms>]`
- `nodes canvas a2ui push --node <id|name|ip> (--jsonl <path> | --text <text>) [--invoke-timeout <ms>]`
- `nodes canvas a2ui reset --node <id|name|ip> [--invoke-timeout <ms>]`
- `nodes screen record --node <id|name|ip> [--screen <index>] [--duration <ms|10s>] [--fps <n>] [--no-audio] [--out <path>] [--invoke-timeout <ms>]`

ロケーション:

- `nodes location get --node <id|name|ip> [--max-age <ms>] [--accuracy <coarse|balanced|precise>] [--location-timeout <ms>] [--invoke-timeout <ms>]`

## ブラウザ

ブラウザコントロール CLI（専用 Chrome/Brave/Edge/Chromium）。[`openclaw browser`](/cli/browser) および [Browser tool](/tools/browser) を参照してください。

共通オプション:

- `--url`、`--token`、`--timeout`、`--json`
- `--browser-profile <name>`

管理:

- `browser status`
- `browser start`
- `browser stop`
- `browser reset-profile`
- `browser tabs`
- `browser open <url>`
- `browser focus <targetId>`
- `browser close [targetId]`
- `browser profiles`
- `browser create-profile --name <name> [--color <hex>] [--cdp-url <url>]`
- `browser delete-profile --name <name>`

検査:

- `browser screenshot [targetId] [--full-page] [--ref <ref>] [--element <selector>] [--type png|jpeg]`
- `browser snapshot [--format aria|ai] [--target-id <id>] [--limit <n>] [--interactive] [--compact] [--depth <n>] [--selector <sel>] [--out <path>]`

アクション:

- `browser navigate <url> [--target-id <id>]`
- `browser resize <width> <height> [--target-id <id>]`
- `browser click <ref> [--double] [--button <left|right|middle>] [--modifiers <csv>] [--target-id <id>]`
- `browser type <ref> <text> [--submit] [--slowly] [--target-id <id>]`
- `browser press <key> [--target-id <id>]`
- `browser hover <ref> [--target-id <id>]`
- `browser drag <startRef> <endRef> [--target-id <id>]`
- `browser select <ref> <values...> [--target-id <id>]`
- `browser upload <paths...> [--ref <ref>] [--input-ref <ref>] [--element <selector>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser fill [--fields <json>] [--fields-file <path>] [--target-id <id>]`
- `browser dialog --accept|--dismiss [--prompt <text>] [--target-id <id>] [--timeout-ms <ms>]`
- `browser wait [--time <ms>] [--text <value>] [--text-gone <value>] [--target-id <id>]`
- `browser evaluate --fn <code> [--ref <ref>] [--target-id <id>]`
- `browser console [--level <error|warn|info>] [--target-id <id>]`
- `browser pdf [--target-id <id>]`

## ドキュメント検索

### `docs [query...]`

ライブドキュメントインデックスを検索します。

## TUI

### `tui`

Gateway に接続されたターミナル UI を開きます。

オプション:

- `--url <url>`
- `--token <token>`
- `--password <password>`
- `--session <key>`
- `--deliver`
- `--thinking <level>`
- `--message <text>`
- `--timeout-ms <ms>`（デフォルトは `agents.defaults.timeoutSeconds`）
- `--history-limit <n>`
