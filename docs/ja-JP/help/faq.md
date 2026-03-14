---
summary: "OpenClaw のセットアップ、設定、使用方法に関するよくある質問"
read_when:
  - よくあるセットアップ、インストール、オンボーディング、またはランタイムのサポート質問に回答する場合
  - 詳しいデバッグの前にユーザーが報告した問題をトリアージする場合
title: "FAQ"
---

# FAQ

一般的な質問への簡単な回答と、実際のセットアップ（ローカル開発、VPS、マルチエージェント、OAuth/API キー、モデルフェイルオーバー）における詳細なトラブルシューティングです。ランタイム診断については [トラブルシューティング](/gateway/troubleshooting) を参照してください。完全な設定リファレンスについては [設定](/gateway/configuration) を参照してください。

## 目次

- [クイックスタートと初回セットアップ](#quick-start-and-first-run-setup)
- [OpenClaw とは？](#what-is-openclaw)
- [Skills と自動化](#skills-and-automation)
- [サンドボックスとメモリ](#sandboxing-and-memory)
- [ディスク上のファイルの場所](#where-things-live-on-disk)
- [設定の基本](#config-basics)
- [リモートゲートウェイとノード](#remote-gateways-and-nodes)
- [環境変数と .env 読み込み](#env-vars-and-env-loading)
- [セッションと複数チャット](#sessions-and-multiple-chats)
- [モデル：デフォルト、選択、エイリアス、切り替え](#models-defaults-selection-aliases-switching)
- [モデルフェイルオーバーと「All models failed」](#model-failover-and-all-models-failed)
- [認証プロファイル：概念と管理方法](#auth-profiles-what-they-are-and-how-to-manage-them)
- [ゲートウェイ：ポート、「既に実行中」、リモートモード](#gateway-ports-already-running-and-remote-mode)
- [ログとデバッグ](#logging-and-debugging)
- [メディアと添付ファイル](#media-and-attachments)
- [セキュリティとアクセス制御](#security-and-access-control)
- [チャットコマンド、タスクの中止、「止まらない」](#chat-commands-aborting-tasks-and-it-wont-stop)

## 問題発生後の最初の 60 秒

1. **クイックステータス（最初に確認）**

   ```bash
   openclaw status
   ```

   OS + アップデート、ゲートウェイ/サービスの到達性、エージェント/セッション、プロバイダー設定 + ランタイム問題の素早いローカルサマリーです。

2. **貼り付け可能なレポート（安全に共有可能）**

   ```bash
   openclaw status --all
   ```

   ログの末尾付きの読み取り専用診断（トークンはマスク済み）。

3. **デーモン + ポートの状態**

   ```bash
   openclaw gateway status
   ```

   supervisor のランタイム状態と RPC の到達性、プローブターゲット URL、サービスが使用した可能性の高い設定を表示します。

4. **詳細プローブ**

   ```bash
   openclaw status --deep
   ```

   ゲートウェイのヘルスチェック + プロバイダープローブを実行します（到達可能なゲートウェイが必要）。[ヘルス](/gateway/health) を参照してください。

5. **最新ログの追跡**

   ```bash
   openclaw logs --follow
   ```

   RPC が利用できない場合は、以下にフォールバックしてください：

   ```bash
   tail -f "$(ls -t /tmp/openclaw/openclaw-*.log | head -1)"
   ```

6. **doctor の実行（修復）**

   ```bash
   openclaw doctor
   ```

   設定/状態の修復/移行 + ヘルスチェックの実行。[Doctor](/gateway/doctor) を参照してください。

7. **ゲートウェイスナップショット**

   ```bash
   openclaw health --json
   openclaw health --verbose   # エラー時にターゲット URL + 設定パスを表示
   ```

## クイックスタートと初回セットアップ

### 行き詰まったとき、最速の解決方法は？

**マシンを見ることができる**ローカル AI エージェントを使用してください。これは Discord で質問するよりもはるかに効果的です。ほとんどの「行き詰まり」ケースはリモートのヘルパーが検査できないローカルの設定や環境の問題です。

- **Claude Code**：https://www.anthropic.com/claude-code/
- **OpenAI Codex**：https://openai.com/codex/

hackable（git）インストールから**完全なソースチェックアウト**を提供してください：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --install-method git
```

まず以下のコマンドを実行してください（助けを求めるときに出力を共有してください）：

```bash
openclaw status
openclaw models status
openclaw doctor
```

### OpenClaw のインストールと設定の推奨方法は？

リポジトリはソースから実行し、オンボーディングウィザードを使用することを推奨しています：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw onboard --install-daemon
```

ウィザードは UI アセットを自動的にビルドすることもできます。オンボーディング後、通常はポート **18789** でゲートウェイを実行します。

### オンボーディング後にダッシュボードを開くには？

ウィザードはオンボーディング完了直後にトークン付きのダッシュボード URL でブラウザを開き、サマリーにリンクを印刷します。

### ダッシュボードトークンの認証（localhost 対リモート）は？

**localhost（同じマシン）：**

- `http://127.0.0.1:18789/` を開きます。
- 認証が求められたら、`gateway.auth.token`（または `OPENCLAW_GATEWAY_TOKEN`）のトークンをコントロール UI の設定に貼り付けます。

**localhost 以外：**

- **Tailscale Serve**（推奨）：loopback にバインドしたまま `openclaw gateway --tailscale serve` を実行し、`https://<magicdns>/` を開きます。
- **Tailnet バインド**：`openclaw gateway --bind tailnet --token "<token>"` を実行します。
- **SSH トンネル**：`ssh -N -L 18789:127.0.0.1:18789 user@host` で実行します。

### 必要なランタイムは？

Node **>= 22** が必要です。`pnpm` を推奨します。ゲートウェイには Bun は**推奨されません**。

### Raspberry Pi で動きますか？

はい。ゲートウェイは軽量です。ドキュメントでは個人使用に **512MB〜1GB RAM**、**1 コア**、約 **500MB** のディスクで十分であり、**Raspberry Pi 4 で実行できる**と記載されています。

### Raspberry Pi インストールのヒントは？

- **64 ビット** OS を使用し、Node >= 22 を維持してください。
- ログを確認して迅速に更新できる **hackable（git）インストール**を優先してください。
- チャンネル/Skills なしで開始し、1 つずつ追加してください。

### 「wake up my friend」で止まる / オンボーディングが開始しない場合は？

ゲートウェイを再起動します：

```bash
openclaw gateway restart
```

ステータスと認証を確認します：

```bash
openclaw status
openclaw models status
openclaw logs --follow
```

まだハングする場合：

```bash
openclaw doctor
```

### セットアップを新しいマシン（Mac mini）に再オンボーディングせずに移行できますか？

はい。**状態ディレクトリ**と**ワークスペース**をコピーし、Doctor を 1 回実行します：

1. 新しいマシンに OpenClaw をインストールします。
2. 古いマシンから `$OPENCLAW_STATE_DIR`（デフォルト：`~/.openclaw`）をコピーします。
3. ワークスペース（デフォルト：`~/.openclaw/workspace`）をコピーします。
4. `openclaw doctor` を実行してゲートウェイサービスを再起動します。

### 最新バージョンの変更点はどこで確認できますか？

GitHub の変更ログを確認してください：
https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md

### stable と beta の違いは？

**Stable** と **beta** は **npm dist-tags** であり、別のコードラインではありません：

- `latest` = stable
- `beta` = テスト用の早期ビルド

### beta バージョンのインストール方法と beta と dev の違いは？

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
```

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
```

### インストールとオンボーディングにはどのくらいかかりますか？

- **インストール：** 2〜5 分
- **オンボーディング：** 設定するチャンネル/モデルの数によって 5〜15 分

### インストーラーが止まった場合の詳細な情報の取得方法は？

**詳細出力**でインストーラーを再実行します：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --verbose
```

### Windows でのインストール問題（git が見つからない、openclaw が認識されない）

**1) git not found**：Git for Windows をインストールし、`git` が PATH 上にあることを確認します。

**2) openclaw が認識されない**：npm グローバル bin フォルダーが PATH にありません。`npm config get prefix` を実行してディレクトリを PATH に追加してください。

### Linux に OpenClaw をインストールする方法は？

[Linux](/platforms/linux) および [はじめに](/start/getting-started) ガイドに従ってください。

### VPS に OpenClaw をインストールする方法は？

任意の Linux VPS が動作します。サーバーにインストールし、SSH/Tailscale でゲートウェイに到達します。

ガイド：[exe.dev](/install/exe-dev)、[Hetzner](/install/hetzner)、[Fly.io](/install/fly)。

### Mac mini を購入しなければなりませんか？

いいえ。OpenClaw は macOS または Linux（Windows は WSL2 経由）で動作します。Mac mini はオプションです。

### iMessage サポートに Mac mini は必要ですか？

**Messages にサインインした macOS デバイス**が必要です。Mac mini である必要はありません。[BlueBubbles](/channels/bluebubbles) の使用を推奨します。

### Claude または OpenAI のサブスクリプションが必要ですか？

いいえ。**API キー**（Anthropic/OpenAI/その他）または**ローカルモデルのみ**で OpenClaw を実行できます。

### Codex 認証はどのように機能しますか？

OpenClaw は OAuth（ChatGPT サインイン）経由で **OpenAI Code（Codex）** をサポートしています。

### OpenAI サブスクリプション認証（Codex OAuth）はサポートされていますか？

はい。OpenClaw は **OpenAI Code（Codex）サブスクリプション OAuth** を完全にサポートしています。

### Gemini CLI OAuth の設定方法は？

Gemini CLI はプラグイン認証フローを使用します：

1. プラグインを有効にします：`openclaw plugins enable google-gemini-cli-auth`
2. ログイン：`openclaw models auth login --provider google-gemini-cli --set-default`

### OpenClaw が自分自身を更新するように頼めますか？

**可能ですが、推奨しません**。更新フローによりゲートウェイが再起動し、アクティブなセッションが切断される場合があります。

```bash
openclaw update
openclaw update --channel stable|beta|dev
```

### オンボーディングウィザードは実際に何をしますか？

`openclaw onboard` は**ローカルモード**で以下を案内します：

- モデル/認証セットアップ
- ワークスペースの場所 + ブートストラップファイル
- ゲートウェイ設定
- チャンネル設定（WhatsApp、Telegram、Discord など）
- デーモンインストール
- ヘルスチェックと Skills の選択

## OpenClaw とは？

### OpenClaw を一段落で説明すると？

OpenClaw は自分のデバイスで動かす個人 AI アシスタントです。既に使用しているメッセージプラットフォーム（WhatsApp、Telegram、Slack、Discord、Google Chat、Signal、iMessage、WebChat）で返信し、サポートされているプラットフォームでは音声とライブ Canvas も可能です。**ゲートウェイ**は常時稼働のコントロールプレーンです。

### 価値提案は？

OpenClaw は「単なる Claude ラッパー」ではありません。それは**ローカルファースト**なコントロールプレーンです：

- **あなたのデバイス、あなたのデータ**：ゲートウェイをどこでも実行し、ワークスペース + セッション履歴をローカルに保持します。
- **本物のチャンネル**：WhatsApp/Telegram/Slack/Discord/Signal/iMessage など。
- **モデル非依存**：Anthropic、OpenAI、MiniMax、OpenRouter などを使用。
- **マルチエージェントルーティング**：チャンネル、アカウント、またはタスクごとに別々のエージェント。
- **オープンソースでハッカブル**：ベンダーロックインなしで検査、拡張、セルフホスト。

## Skills と自動化

### リポジトリを汚さずに Skills をカスタマイズする方法は？

`~/.openclaw/skills/<name>/SKILL.md` にマネージドオーバーライドを配置します。優先順位は `<workspace>/skills` > `~/.openclaw/skills` > バンドル済みです。

### カスタムフォルダーから Skills を読み込めますか？

はい。`~/.openclaw/openclaw.json` の `skills.load.extraDirs` で追加ディレクトリを追加します（最低優先順位）。

### 定時タスクやリマインダーが実行されない場合は？

定時タスクはゲートウェイプロセス内で実行されます。ゲートウェイが継続して実行されていない場合、スケジュールされたジョブは実行されません。

```bash
openclaw cron run <jobId> --force
openclaw cron runs --id <jobId> --limit 50
```

### OpenClaw はスケジュールまたはバックグラウンドで継続的にタスクを実行できますか？

はい。ゲートウェイスケジューラーを使用します：

- スケジュールまたは定期タスクには**定時タスク**
- 「メインセッション」の定期チェックには**ハートビート**
- サマリーを投稿する自律エージェントには**隔離ジョブ**

### Chrome 拡張機能（ブラウザーコントロール）のインストール方法は？

```bash
openclaw browser extension install
openclaw browser extension path
```

その後、Chrome → `chrome://extensions` → 「デベロッパーモード」を有効化 → 「パッケージ化されていない拡張機能を読み込む」 → そのフォルダーを選択します。

## サンドボックスとメモリ

### 専用のサンドボックスドキュメントはありますか？

はい。[サンドボックス化](/gateway/sandboxing) を参照してください。Docker 固有のセットアップには [Docker](/install/docker) を参照してください。

### ホストフォルダーをサンドボックスにバインドする方法は？

`agents.defaults.sandbox.docker.binds` を `["host:path:mode"]` に設定します（例：`"/home/user/src:/src:ro"`）。

### メモリはどのように機能しますか？

OpenClaw のメモリはエージェントワークスペース内の Markdown ファイルです：

- 日次メモは `memory/YYYY-MM-DD.md`
- 厳選された長期メモは `MEMORY.md`（メイン/プライベートセッションのみ）

### セマンティックメモリ検索に OpenAI API キーは必要ですか？

**OpenAI エンベディング**を使用する場合のみ必要です。Codex OAuth は chat/completions をカバーしますが、エンベディングアクセスは付与しません。OpenAI エンベディングには実際の API キーが必要です。

## ディスク上のファイルの場所

### OpenClaw はデータをどこに保存しますか？

すべては `$OPENCLAW_STATE_DIR`（デフォルト：`~/.openclaw`）下にあります：

| パス | 目的 |
| --- | --- |
| `$OPENCLAW_STATE_DIR/openclaw.json` | メイン設定（JSON5） |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json` | 認証プロファイル |
| `$OPENCLAW_STATE_DIR/credentials/` | プロバイダーの状態 |
| `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/` | 会話履歴と状態 |

### AGENTS.md / SOUL.md / USER.md / MEMORY.md はどこに置くべきですか？

これらのファイルは `~/.openclaw` ではなく**エージェントワークスペース**にあります。

- **ワークスペース（エージェントごと）**：`AGENTS.md`、`SOUL.md`、`IDENTITY.md`、`USER.md`、`MEMORY.md`、`memory/YYYY-MM-DD.md`

デフォルトのワークスペースは `~/.openclaw/workspace` で、以下で設定可能です：

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

### 推奨のバックアップ戦略は？

**エージェントワークスペース**を**プライベート** git リポジトリに置き、プライベートな場所にバックアップします。`~/.openclaw` 下のものはコミットしないでください（認証情報、セッション、トークン）。

### OpenClaw を完全にアンインストールする方法は？

専用ガイドを参照してください：[アンインストール](/install/uninstall)。

## 設定の基本

### 設定はどのフォーマットですか？どこにありますか？

OpenClaw は `$OPENCLAW_CONFIG_PATH`（デフォルト：`~/.openclaw/openclaw.json`）からオプションの **JSON5** 設定を読み込みます。

### `gateway.bind: "lan"` または `"tailnet"` を設定したが、何もリッスンしない / UI が unauthorized と表示される

非 loopback バインドには**認証が必要**です。`gateway.auth.mode` + `gateway.auth.token`（または `OPENCLAW_GATEWAY_TOKEN`）を設定してください：

```json5
{
  gateway: {
    bind: "lan",
    auth: {
      mode: "token",
      token: "replace-me",
    },
  },
}
```

### 設定変更後に再起動が必要ですか？

ゲートウェイは設定ファイルを監視し、ホットリロードをサポートします：

- `gateway.reload.mode: "hybrid"`（デフォルト）：安全な変更をホットアプライ、クリティカルな変更は再起動

### Web 検索（および Web フェッチ）を有効にする方法は？

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        provider: "brave",
        apiKey: "BRAVE_API_KEY_HERE",
      },
      fetch: { enabled: true },
    },
  },
}
```

### config.apply が設定を消去した場合の回復方法は？

`config.apply` は**設定全体**を置き換えます。

回復方法：

- バックアップから復元します。
- バックアップがない場合は `openclaw doctor` を再実行してチャンネル/モデルを再設定します。

今後の防止方法：

- 小さな変更には `openclaw config set` を使用します。
- インタラクティブな編集には `openclaw configure` を使用します。

## リモートゲートウェイとノード

### コマンドは Telegram、ゲートウェイ、ノード間をどのように伝播しますか？

Telegram メッセージは**ゲートウェイ**が処理します。ゲートウェイはエージェントを実行し、ノードツールが必要な場合にのみ**ゲートウェイ WebSocket** 経由でノードを呼び出します：

`Telegram → ゲートウェイ → エージェント → node.* → ノード → ゲートウェイ → Telegram`

### ゲートウェイがリモートにホストされている場合、エージェントはどのようにコンピューターにアクセスしますか？

**コンピューターをノードとしてペアリング**してください。ゲートウェイはゲートウェイ WebSocket 経由でローカルマシンの `node.*` ツールを呼び出せます。

### 複数のエージェントに別々の VPS が必要ですか？

いいえ。1 つのゲートウェイで複数のエージェントをホストでき、各エージェントに独自のワークスペース、モデルデフォルト、ルーティングがあります。

### ノードはゲートウェイサービスを実行しますか？

いいえ。意図的に隔離されたプロファイルを実行する場合を除き、ホストごとに**1 つのゲートウェイ**のみを実行すべきです。

## 環境変数と .env 読み込み

### OpenClaw は環境変数をどのように読み込みますか？

OpenClaw は親プロセス（シェル、launchd/systemd、CI など）から環境変数を読み込み、さらに以下を読み込みます：

- 現在の作業ディレクトリの `.env`
- `~/.openclaw/.env`（`$OPENCLAW_STATE_DIR/.env`）のグローバルフォールバック

どちらの `.env` ファイルも既存の環境変数を上書きしません。

### サービス経由でゲートウェイを起動したが環境変数が消えた場合は？

2 つの一般的な修正方法：

1. 不足しているキーを `~/.openclaw/.env` に置きます。
2. シェルインポートを有効にします（オプトイン）：

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

## セッションと複数チャット

### 新しい会話を開始する方法は？

`/new` または `/reset` をスタンドアロンメッセージとして送信します。

### `/new` を送信しない場合、セッションは自動的にリセットされますか？

はい。セッションは `session.idleMinutes`（デフォルト **60**）後に期限切れになります。

### コンテキストがタスクの途中で切り捨てられた場合の防止方法は？

- ボットに現在の状態を要約してファイルに書き込むように依頼します。
- 長いタスクの前に `/compact` を使用し、トピックを切り替えるときは `/new` を使用します。
- 長いまたは並行作業にサブエージェントを使用します。

### WhatsApp グループに「ボットアカウント」を追加する必要がありますか？

いいえ。OpenClaw は**あなた自身のアカウント**で動作するため、グループにいれば OpenClaw もそれを見ることができます。

## モデル：デフォルト、選択、エイリアス、切り替え

### デフォルトモデルとは？

OpenClaw のデフォルトモデルは以下で設定したものです：

```
agents.defaults.model.primary
```

モデルは `provider/model` として参照されます（例：`anthropic/claude-opus-4-6`）。

### 設定を消去せずにモデルを切り替える方法は？

- チャットで `/model`（クイック、セッションごと）
- `openclaw models set ...`（モデル設定のみ更新）
- `openclaw configure --section model`（インタラクティブ）

### 実行中にモデルを切り替える方法（再起動なし）は？

スタンドアロンメッセージとして `/model` コマンドを使用します：

```
/model sonnet
/model opus
/model gpt
/model gemini
```

### モデルエイリアス（ショートカット）を定義/オーバーライドする方法は？

エイリアスは `agents.defaults.models.<modelId>.alias` から来ます：

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-4-6" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
      },
    },
  },
}
```

## モデルフェイルオーバーと「All models failed」

### フェイルオーバーはどのように機能しますか？

フェイルオーバーは 2 段階で発生します：

1. 同じプロバイダー内での**認証プロファイルのローテーション**。
2. `agents.defaults.model.fallbacks` の次のモデルへの**モデルフォールバック**。

### `No credentials found for profile "anthropic:default"` の修正チェックリスト

- **認証プロファイルの場所を確認**（新 vs レガシーパス）
  - 現在：`~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **環境変数がゲートウェイに読み込まれているか確認**
  - システムd/launchd 経由でゲートウェイを実行している場合、シェル環境を継承しない可能性があります。`~/.openclaw/.env` に配置するか `env.shellEnv` を有効にしてください。
- **正しいエージェントを編集していることを確認**
- **モデル/認証ステータスのサニティーチェック**
  - `openclaw models status` を使用します。

## 認証プロファイル：概念と管理方法

### 認証プロファイルとは？

認証プロファイルはプロバイダーに紐付けられた名前付き認証レコード（OAuth または API キー）です。プロファイルは以下にあります：

```
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

### どの認証プロファイルを最初に試すかを制御できますか？

はい。CLI 経由で**エージェントごとの**順序オーバーライドを設定できます：

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

## ゲートウェイ：ポート、「既に実行中」、リモートモード

### ゲートウェイが使用するポートは？

`gateway.port` は WebSocket + HTTP（コントロール UI、フックなど）の単一多重ポートを制御します。

優先順位：

```
--port > OPENCLAW_GATEWAY_PORT > gateway.port > デフォルト 18789
```

### OpenClaw をリモートモードで実行する方法（クライアントが別の場所のゲートウェイに接続）は？

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://gateway.tailnet:18789",
      token: "your-token",
    },
  },
}
```

### コントロール UI が「unauthorized」または接続し続ける場合は？

最速の修正：

```bash
openclaw dashboard
```

または `openclaw doctor --generate-gateway-token` でトークンを生成します。

### 同一ホストで複数のゲートウェイを実行できますか？

通常はいいえ。1 つのゲートウェイで複数のメッセージチャンネルとエージェントを実行できます。冗長性やハード隔離が必要な場合のみ複数のゲートウェイを使用してください。

使用する場合は、`OPENCLAW_CONFIG_PATH`、`OPENCLAW_STATE_DIR`、`agents.defaults.workspace`、`gateway.port` を隔離してください。

## ログとデバッグ

### ログはどこにありますか？

ファイルログ（構造化）：

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

最速のログ追跡：

```bash
openclaw logs --follow
```

サービス/supervisor ログ：

- macOS：`$OPENCLAW_STATE_DIR/logs/gateway.log`
- Linux：`journalctl --user -u openclaw-gateway.service -n 200 --no-pager`

### ゲートウェイサービスを開始/停止/再起動する方法は？

```bash
openclaw gateway status
openclaw gateway restart
```

### ゲートウェイは起動しているが返信が届かない場合に確認すべきことは？

```bash
openclaw status
openclaw models status
openclaw channels status
openclaw logs --follow
```

## メディアと添付ファイル

### Skill が画像/PDF を生成したが何も送信されない

エージェントからの送信添付ファイルには `MEDIA:<path-or-url>` 行（単独行）が含まれている必要があります。

```bash
openclaw message send --target +15555550123 --message "Here you go" --media /path/to/file.png
```

## セキュリティとアクセス制御

### OpenClaw をインバウンド DM に公開することは安全ですか？

インバウンド DM を信頼できない入力として扱ってください。デフォルトはリスクを低減するよう設計されています：

- DM 対応チャンネルのデフォルト動作は**ペアリング**です。
- DM を公開で開くには明示的なオプトイン（`dmPolicy: "open"` と許可リスト `"*"`）が必要です。

### プロンプトインジェクションは公開ボットだけの懸念事項ですか？

いいえ。プロンプトインジェクションは**信頼できないコンテンツ**に関するものであり、誰がボットに DM できるかだけではありません。アシスタントが外部コンテンツ（Web 検索/フェッチ、ブラウザーページ、メール、ドキュメント）を読む場合、そのコンテンツにはモデルをハイジャックしようとする指示が含まれている可能性があります。

## チャットコマンド、タスクの中止、「止まらない」

### 実行中のタスクを停止/キャンセルする方法は？

以下のいずれかを**スタンドアロンメッセージ**として送信します（スラッシュなし）：

```
stop
abort
esc
wait
exit
interrupt
```

### ボットが高速連投メッセージを「無視する」ように感じる理由は？

キューモードは新しいメッセージが進行中の実行とどのように対話するかを制御します。`/queue` を使用してモードを変更します：

- `steer` - 新しいメッセージが現在のタスクをリダイレクト
- `followup` - メッセージを 1 つずつ実行
- `collect` - メッセージをバッチ処理して一度に返信（デフォルト）
- `interrupt` - 現在の実行を中止して新規開始

---

それでも行き詰まっていますか？[Discord](https://discord.com/invite/clawd) で質問するか、[GitHub ディスカッション](https://github.com/openclaw/openclaw/discussions)を開いてください。
