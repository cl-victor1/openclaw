---
summary: "フック: コマンドとライフサイクルイベントのためのイベント駆動自動化"
read_when:
  - /new、/reset、/stop、エージェントライフサイクルイベントのイベント駆動自動化が必要な場合
  - フックのビルド、インストール、またはデバッグを行う場合
title: "Hooks"
---

# Hooks

フックはエージェントコマンドとイベントに応じてアクションを自動化するための拡張可能なイベント駆動システムを提供します。フックはディレクトリから自動的に検出され、OpenClawでのスキルの動作と同様にCLIコマンドで管理できます。

## 概要

フックはある事が起きた時に実行される小さなスクリプトです。2種類あります:

- **Hooks**（このページ）: `/new`、`/reset`、`/stop`、またはライフサイクルイベントのようなエージェントイベントが発火するときにGateway内で実行されます。
- **Webhooks**: 外部システムがOpenClawで作業をトリガーできるようにする外部HTTPウェブフック。[Webhook Hooks](/automation/webhook) または Gmail ヘルパーコマンドの `openclaw webhooks` を参照してください。

フックはプラグイン内にバンドルすることもできます。[Plugins](/tools/plugin#plugin-hooks) を参照してください。

一般的な用途:

- セッションをリセットする際にメモリのスナップショットを保存する
- トラブルシューティングやコンプライアンスのためにコマンドの監査証跡を保持する
- セッションの開始または終了時にフォローアップ自動化をトリガーする
- イベントが発火するときにエージェントワークスペースにファイルを書き込んだり外部APIを呼び出したりする

小さなTypeScript関数を書けるなら、フックを書けます。フックは自動的に検出され、CLIで有効化・無効化できます。

## 概要

フックシステムを使用すると:

- `/new` が発行されたときにセッションコンテキストをメモリに保存できる
- 監査のためにすべてのコマンドをログに記録できる
- エージェントライフサイクルイベントでカスタム自動化をトリガーできる
- コアコードを変更せずにOpenClawの動作を拡張できる

## はじめに

### バンドルされたフック

OpenClawには4つのバンドルされたフックが付属しており、自動的に検出されます:

- **💾 session-memory**: `/new` を発行したときにエージェントワークスペース（デフォルト `~/.openclaw/workspace/memory/`）にセッションコンテキストを保存します
- **📎 bootstrap-extra-files**: `agent:bootstrap` 中に設定されたグロブ/パスパターンから追加のワークスペースブートストラップファイルを注入します
- **📝 command-logger**: すべてのコマンドイベントを `~/.openclaw/logs/commands.log` に記録します
- **🚀 boot-md**: Gatewayが起動したとき（内部フックが有効な場合）に `BOOT.md` を実行します

使用可能なフックを一覧表示:

```bash
openclaw hooks list
```

フックを有効化:

```bash
openclaw hooks enable session-memory
```

フックの状態を確認:

```bash
openclaw hooks check
```

詳細情報を取得:

```bash
openclaw hooks info session-memory
```

### オンボーディング

オンボーディング（`openclaw onboard`）中に、推奨フックを有効にするよう促されます。ウィザードは対象となるフックを自動的に検出し、選択のために提示します。

## フック検出

フックは3つのディレクトリから優先度順に自動的に検出されます:

1. **ワークスペースフック**: `<workspace>/hooks/`（エージェントごと、最高優先度）
2. **管理フック**: `~/.openclaw/hooks/`（ユーザーインストール、ワークスペース間で共有）
3. **バンドルフック**: `<openclaw>/dist/hooks/bundled/`（OpenClawに同梱）

管理フックディレクトリは**単一フック**または**フックパック**（パッケージディレクトリ）のいずれかです。

各フックは以下を含むディレクトリです:

```
my-hook/
├── HOOK.md          # メタデータ + ドキュメント
└── handler.ts       # ハンドラー実装
```

## フックパック（npm/アーカイブ）

フックパックは `package.json` の `openclaw.hooks` を通じて1つ以上のフックをエクスポートする標準的なnpmパッケージです。次のコマンドでインストールできます:

```bash
openclaw hooks install <path-or-spec>
```

Npmスペックはレジストリのみです（パッケージ名 + オプションの正確なバージョンまたはdistタグ）。
Git/URL/ファイルスペックとsemverレンジは拒否されます。

ベアスペックと `@latest` は安定版トラックに留まります。npmがいずれかをプレリリースに解決した場合、OpenClawは停止し、`@beta`/`@rc` のようなプレリリースタグや正確なプレリリースバージョンで明示的にオプトインするよう求めます。

`package.json` の例:

```json
{
  "name": "@acme/my-hooks",
  "version": "0.1.0",
  "openclaw": {
    "hooks": ["./hooks/my-hook", "./hooks/other-hook"]
  }
}
```

各エントリは `HOOK.md` と `handler.ts`（または `index.ts`）を含むフックディレクトリを指します。
フックパックは依存関係を同梱できます。`~/.openclaw/hooks/<id>` の下にインストールされます。
各 `openclaw.hooks` エントリはシンボリックリンク解決後もパッケージディレクトリ内に留まる必要があります。エスケープするエントリは拒否されます。

セキュリティ注記: `openclaw hooks install` は `npm install --ignore-scripts`（ライフサイクルスクリプトなし）で依存関係をインストールします。フックパックの依存関係ツリーを「純粋JS/TS」に保ち、`postinstall` ビルドに依存するパッケージを避けてください。

## フック構造

### HOOK.mdフォーマット

`HOOK.md` ファイルにはYAMLフロントマターとMarkdownドキュメントが含まれます:

```markdown
---
name: my-hook
description: "このフックが何をするかの短い説明"
homepage: https://docs.openclaw.ai/automation/hooks#my-hook
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# My Hook

詳細なドキュメントはここに...

## What It Does

- `/new` コマンドをリッスンする
- 何らかのアクションを実行する
- 結果をログに記録する

## Requirements

- Node.jsがインストールされている必要があります

## Configuration

設定不要。
```

### メタデータフィールド

`metadata.openclaw` オブジェクトは以下をサポートします:

- **`emoji`**: CLI表示の絵文字（例: `"💾"`）
- **`events`**: リッスンするイベントの配列（例: `["command:new", "command:reset"]`）
- **`export`**: 使用する名前付きエクスポート（デフォルトは `"default"`）
- **`homepage`**: ドキュメントURL
- **`requires`**: オプションの要件
  - **`bins`**: PATH上の必須バイナリ（例: `["git", "node"]`）
  - **`anyBins`**: これらのバイナリのうち少なくとも1つが必要
  - **`env`**: 必須環境変数
  - **`config`**: 必須設定パス（例: `["workspace.dir"]`）
  - **`os`**: 必須プラットフォーム（例: `["darwin", "linux"]`）
- **`always`**: 適格性チェックをバイパス（ブール値）
- **`install`**: インストール方法（バンドルフックの場合: `[{"id":"bundled","kind":"bundled"}]`）

### ハンドラー実装

`handler.ts` ファイルは `HookHandler` 関数をエクスポートします:

```typescript
const myHandler = async (event) => {
  // 'new' コマンドのみトリガー
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] New command triggered`);
  console.log(`  Session: ${event.sessionKey}`);
  console.log(`  Timestamp: ${event.timestamp.toISOString()}`);

  // カスタムロジックをここに

  // オプションでユーザーにメッセージを送信
  event.messages.push("✨ My hook executed!");
};

export default myHandler;
```

#### イベントコンテキスト

各イベントには以下が含まれます:

```typescript
{
  type: 'command' | 'session' | 'agent' | 'gateway' | 'message',
  action: string,              // 例: 'new', 'reset', 'stop', 'received', 'sent'
  sessionKey: string,          // セッション識別子
  timestamp: Date,             // イベントが発生した時刻
  messages: string[],          // ユーザーに送信するメッセージをここにプッシュ
  context: {
    // コマンドイベント:
    sessionEntry?: SessionEntry,
    sessionId?: string,
    sessionFile?: string,
    commandSource?: string,    // 例: 'whatsapp', 'telegram'
    senderId?: string,
    workspaceDir?: string,
    bootstrapFiles?: WorkspaceBootstrapFile[],
    cfg?: OpenClawConfig,
    // メッセージイベント（全詳細はMessage Eventsセクション参照）:
    from?: string,             // message:received
    to?: string,               // message:sent
    content?: string,
    channelId?: string,
    success?: boolean,         // message:sent
  }
}
```

## イベントタイプ

### コマンドイベント

エージェントコマンドが発行されたときにトリガーされます:

- **`command`**: すべてのコマンドイベント（汎用リスナー）
- **`command:new`**: `/new` コマンドが発行されたとき
- **`command:reset`**: `/reset` コマンドが発行されたとき
- **`command:stop`**: `/stop` コマンドが発行されたとき

### セッションイベント

- **`session:compact:before`**: コンパクションが履歴をサマリー化する直前
- **`session:compact:after`**: コンパクションがサマリーメタデータとともに完了した後

内部フックペイロードはこれらを `type: "session"` として `action: "compact:before"` / `action: "compact:after"` とともにエミットします。リスナーは上記の組み合わせキーでサブスクライブします。
特定のハンドラー登録は `${type}:${action}` のリテラルキー形式を使用します。これらのイベントには `session:compact:before` と `session:compact:after` を登録します。

### エージェントイベント

- **`agent:bootstrap`**: ワークスペースブートストラップファイルが注入される前（フックは `context.bootstrapFiles` を変更できます）

### Gatewayイベント

Gatewayが起動したときにトリガーされます:

- **`gateway:startup`**: チャンネルが起動してフックがロードされた後

### メッセージイベント

メッセージが受信または送信されたときにトリガーされます:

- **`message`**: すべてのメッセージイベント（汎用リスナー）
- **`message:received`**: 任意のチャンネルからのインバウンドメッセージが受信されたとき。メディア理解の前の処理の早い段階で発火します。コンテンツには、まだ処理されていないメディア添付のための `<media:audio>` のような生のプレースホルダーが含まれる場合があります。
- **`message:transcribed`**: 音声トランスクリプションとリンク理解を含む、メッセージが完全に処理されたとき。この時点で `transcript` には音声メッセージの完全なトランスクリプトテキストが含まれます。トランスクリプトされた音声コンテンツへのアクセスが必要な場合はこのフックを使用してください。
- **`message:preprocessed`**: すべてのメディア+リンク理解が完了した後、すべてのメッセージに対して発火し、エージェントが見る前に完全にリッチ化された本文（トランスクリプト、画像の説明、リンクサマリー）へのアクセスをフックに提供します。
- **`message:sent`**: アウトバウンドメッセージが正常に送信されたとき

#### メッセージイベントコンテキスト

メッセージイベントにはメッセージに関するリッチなコンテキストが含まれます:

```typescript
// message:received コンテキスト
{
  from: string,           // 送信者識別子（電話番号、ユーザーIDなど）
  content: string,        // メッセージコンテンツ
  timestamp?: number,     // 受信時のUnixタイムスタンプ
  channelId: string,      // チャンネル（例: "whatsapp", "telegram", "discord"）
  accountId?: string,     // マルチアカウントセットアップ用プロバイダーアカウントID
  conversationId?: string, // チャット/コンバセーションID
  messageId?: string,     // プロバイダーからのメッセージID
  metadata?: {            // 追加のプロバイダー固有データ
    to?: string,
    provider?: string,
    surface?: string,
    threadId?: string,
    senderId?: string,
    senderName?: string,
    senderUsername?: string,
    senderE164?: string,
  }
}

// message:sent コンテキスト
{
  to: string,             // 受信者識別子
  content: string,        // 送信されたメッセージコンテンツ
  success: boolean,       // 送信が成功したかどうか
  error?: string,         // 送信失敗時のエラーメッセージ
  channelId: string,      // チャンネル（例: "whatsapp", "telegram", "discord"）
  accountId?: string,     // プロバイダーアカウントID
  conversationId?: string, // チャット/コンバセーションID
  messageId?: string,     // プロバイダーから返されたメッセージID
  isGroup?: boolean,      // このアウトバウンドメッセージがグループ/チャンネルコンテキストに属するか
  groupId?: string,       // message:receivedとの相関用グループ/チャンネル識別子
}

// message:transcribed コンテキスト
{
  body?: string,          // リッチ化前の生のインバウンド本文
  bodyForAgent?: string,  // エージェントに表示されるリッチ化された本文
  transcript: string,     // 音声トランスクリプトテキスト
  channelId: string,      // チャンネル（例: "telegram", "whatsapp"）
  conversationId?: string,
  messageId?: string,
}

// message:preprocessed コンテキスト
{
  body?: string,          // 生のインバウンド本文
  bodyForAgent?: string,  // メディア/リンク理解後の最終リッチ化本文
  transcript?: string,    // 音声が存在した場合のトランスクリプト
  channelId: string,      // チャンネル（例: "telegram", "whatsapp"）
  conversationId?: string,
  messageId?: string,
  isGroup?: boolean,
  groupId?: string,
}
```

#### 例: メッセージロガーフック

```typescript
const isMessageReceivedEvent = (event: { type: string; action: string }) =>
  event.type === "message" && event.action === "received";
const isMessageSentEvent = (event: { type: string; action: string }) =>
  event.type === "message" && event.action === "sent";

const handler = async (event) => {
  if (isMessageReceivedEvent(event as { type: string; action: string })) {
    console.log(`[message-logger] Received from ${event.context.from}: ${event.context.content}`);
  } else if (isMessageSentEvent(event as { type: string; action: string })) {
    console.log(`[message-logger] Sent to ${event.context.to}: ${event.context.content}`);
  }
};

export default handler;
```

### ツール結果フック（Plugin API）

これらのフックはイベントストリームリスナーではありません。プラグインがOpenClawがセッションの記録に永続化する前に同期的にツール結果を調整できるようにします。

- **`tool_result_persist`**: セッションのトランスクリプトに書き込まれる前にツール結果を変換します。同期的である必要があります。更新されたツール結果ペイロードを返すか、そのままにする場合は `undefined` を返します。[Agent Loop](/concepts/agent-loop) を参照してください。

### プラグインフックイベント

プラグインフックランナーを通じて公開されるコンパクションライフサイクルフック:

- **`before_compaction`**: カウント/トークンメタデータとともにコンパクション前に実行
- **`after_compaction`**: コンパクションサマリーメタデータとともにコンパクション後に実行

### 将来のイベント

計画中のイベントタイプ:

- **`session:start`**: 新しいセッションが始まるとき
- **`session:end`**: セッションが終了するとき
- **`agent:error`**: エージェントがエラーに遭遇するとき

## カスタムフックの作成

### 1. 場所を選ぶ

- **ワークスペースフック** (`<workspace>/hooks/`): エージェントごと、最高優先度
- **管理フック** (`~/.openclaw/hooks/`): ワークスペース間で共有

### 2. ディレクトリ構造を作成

```bash
mkdir -p ~/.openclaw/hooks/my-hook
cd ~/.openclaw/hooks/my-hook
```

### 3. HOOK.mdを作成

```markdown
---
name: my-hook
description: "役に立つことをする"
metadata: { "openclaw": { "emoji": "🎯", "events": ["command:new"] } }
---

# My Custom Hook

このフックは `/new` を発行したときに役に立つことをします。
```

### 4. handler.tsを作成

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log("[my-hook] Running!");
  // ロジックをここに
};

export default handler;
```

### 5. 有効化とテスト

```bash
# フックが検出されることを確認
openclaw hooks list

# 有効化
openclaw hooks enable my-hook

# Gatewayプロセスを再起動（macOSではメニューバーアプリを再起動、または開発プロセスを再起動）

# イベントをトリガー
# メッセージングチャンネルで /new を送信
```

## 設定

### 新しい設定フォーマット（推奨）

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

### フックごとの設定

フックはカスタム設定を持てます:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": {
            "MY_CUSTOM_VAR": "value"
          }
        }
      }
    }
  }
}
```

### 追加ディレクトリ

追加のディレクトリからフックをロード:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

### レガシー設定フォーマット（まだサポート）

古い設定フォーマットは後方互換性のためにまだ機能します:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "handlers": [
        {
          "event": "command:new",
          "module": "./hooks/handlers/my-handler.ts",
          "export": "default"
        }
      ]
    }
  }
}
```

注: `module` はワークスペース相対パスである必要があります。絶対パスとワークスペース外へのトラバーサルは拒否されます。

**移行**: 新しいフックには新しい検出ベースのシステムを使用してください。レガシーハンドラーはディレクトリベースのフックの後にロードされます。

## CLIコマンド

### フックを一覧表示

```bash
# すべてのフックを一覧表示
openclaw hooks list

# 対象フックのみ表示
openclaw hooks list --eligible

# 詳細出力（欠落している要件を表示）
openclaw hooks list --verbose

# JSON出力
openclaw hooks list --json
```

### フック情報

```bash
# フックの詳細情報を表示
openclaw hooks info session-memory

# JSON出力
openclaw hooks info session-memory --json
```

### 適格性を確認

```bash
# 適格性サマリーを表示
openclaw hooks check

# JSON出力
openclaw hooks check --json
```

### 有効化/無効化

```bash
# フックを有効化
openclaw hooks enable session-memory

# フックを無効化
openclaw hooks disable command-logger
```

## バンドルフックリファレンス

### session-memory

`/new` を発行したときにセッションコンテキストをメモリに保存します。

**イベント**: `command:new`

**要件**: `workspace.dir` が設定されている必要があります

**出力**: `<workspace>/memory/YYYY-MM-DD-slug.md`（デフォルト `~/.openclaw/workspace`）

**動作**:

1. リセット前のセッションエントリを使用して正しいトランスクリプトを見つける
2. 会話の最後の15行を抽出する
3. LLMを使用して説明的なファイル名スラッグを生成する
4. 日付付きメモリファイルにセッションメタデータを保存する

**出力例**:

```markdown
# Session: 2026-01-16 14:30:00 UTC

- **Session Key**: agent:main:main
- **Session ID**: abc123def456
- **Source**: telegram
```

**ファイル名の例**:

- `2026-01-16-vendor-pitch.md`
- `2026-01-16-api-design.md`
- `2026-01-16-1430.md`（スラッグ生成失敗時のフォールバックタイムスタンプ）

**有効化**:

```bash
openclaw hooks enable session-memory
```

### bootstrap-extra-files

`agent:bootstrap` 中に追加のブートストラップファイル（例: モノレポローカルの `AGENTS.md` / `TOOLS.md`）を注入します。

**イベント**: `agent:bootstrap`

**要件**: `workspace.dir` が設定されている必要があります

**出力**: ファイルは書き込まれません。ブートストラップコンテキストはメモリ内でのみ変更されます。

**設定**:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

**注**:

- パスはワークスペースを基準に解決されます。
- ファイルはワークスペース内に留まる必要があります（realpathチェック済み）。
- 認識されたブートストラップのベース名のみがロードされます。
- サブエージェント許可リストは保持されます（`AGENTS.md` と `TOOLS.md` のみ）。

**有効化**:

```bash
openclaw hooks enable bootstrap-extra-files
```

### command-logger

すべてのコマンドイベントを集中型監査ファイルに記録します。

**イベント**: `command`

**要件**: なし

**出力**: `~/.openclaw/logs/commands.log`

**動作**:

1. イベント詳細（コマンドアクション、タイムスタンプ、セッションキー、送信者ID、ソース）をキャプチャ
2. JSONL形式でログファイルに追記
3. バックグラウンドで静かに実行

**ログエントリの例**:

```jsonl
{"timestamp":"2026-01-16T14:30:00.000Z","action":"new","sessionKey":"agent:main:main","senderId":"+1234567890","source":"telegram"}
{"timestamp":"2026-01-16T15:45:22.000Z","action":"stop","sessionKey":"agent:main:main","senderId":"user@example.com","source":"whatsapp"}
```

**ログを見る**:

```bash
# 最近のコマンドを表示
tail -n 20 ~/.openclaw/logs/commands.log

# jqで整形表示
cat ~/.openclaw/logs/commands.log | jq .

# アクションでフィルタリング
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .
```

**有効化**:

```bash
openclaw hooks enable command-logger
```

### boot-md

Gatewayが起動したとき（チャンネルが起動した後）に `BOOT.md` を実行します。
これを実行するには内部フックが有効になっている必要があります。

**イベント**: `gateway:startup`

**要件**: `workspace.dir` が設定されている必要があります

**動作**:

1. ワークスペースから `BOOT.md` を読み込む
2. エージェントランナーを通じて指示を実行する
3. messageツールを通じてリクエストされたアウトバウンドメッセージを送信する

**有効化**:

```bash
openclaw hooks enable boot-md
```

## ベストプラクティス

### ハンドラーを高速に保つ

フックはコマンド処理中に実行されます。軽量に保ってください:

```typescript
// ✓ 良い - 非同期作業、すぐにリターン
const handler: HookHandler = async (event) => {
  void processInBackground(event); // ファイア・アンド・フォーゲット
};

// ✗ 悪い - コマンド処理をブロック
const handler: HookHandler = async (event) => {
  await slowDatabaseQuery(event);
  await evenSlowerAPICall(event);
};
```

### エラーを優雅に処理する

リスクのある操作は常にラップしてください:

```typescript
const handler: HookHandler = async (event) => {
  try {
    await riskyOperation(event);
  } catch (err) {
    console.error("[my-handler] Failed:", err instanceof Error ? err.message : String(err));
    // スローしない - 他のハンドラーを実行させる
  }
};
```

### イベントを早くフィルタリングする

イベントが関係ない場合は早期リターンしてください:

```typescript
const handler: HookHandler = async (event) => {
  // 'new' コマンドのみ処理
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  // ロジックをここに
};
```

### 特定のイベントキーを使用する

可能な場合はメタデータで正確なイベントを指定してください:

```yaml
metadata: { "openclaw": { "events": ["command:new"] } } # 特定
```

以下の代わりに:

```yaml
metadata: { "openclaw": { "events": ["command"] } } # 汎用 - オーバーヘッドが多い
```

## デバッグ

### フックログを有効化

Gatewayは起動時にフックのロードをログに記録します:

```
Registered hook: session-memory -> command:new
Registered hook: bootstrap-extra-files -> agent:bootstrap
Registered hook: command-logger -> command
Registered hook: boot-md -> gateway:startup
```

### 検出を確認

すべての検出されたフックを一覧表示:

```bash
openclaw hooks list --verbose
```

### 登録を確認

ハンドラーで呼び出された時にログを記録:

```typescript
const handler: HookHandler = async (event) => {
  console.log("[my-handler] Triggered:", event.type, event.action);
  // ロジック
};
```

### 適格性を確認

フックが対象でない理由を確認:

```bash
openclaw hooks info my-hook
```

出力で欠落している要件を探してください。

## テスト

### Gatewayログ

Gatewayログを監視してフックの実行を確認:

```bash
# macOS
./scripts/clawlog.sh -f

# その他のプラットフォーム
tail -f ~/.openclaw/gateway.log
```

### フックを直接テスト

ハンドラーを単独でテスト:

```typescript
import { test } from "vitest";
import myHandler from "./hooks/my-hook/handler.js";

test("my handler works", async () => {
  const event = {
    type: "command",
    action: "new",
    sessionKey: "test-session",
    timestamp: new Date(),
    messages: [],
    context: { foo: "bar" },
  };

  await myHandler(event);

  // 副作用をアサート
});
```

## アーキテクチャ

### コアコンポーネント

- **`src/hooks/types.ts`**: 型定義
- **`src/hooks/workspace.ts`**: ディレクトリスキャンとロード
- **`src/hooks/frontmatter.ts`**: HOOK.mdメタデータ解析
- **`src/hooks/config.ts`**: 適格性チェック
- **`src/hooks/hooks-status.ts`**: ステータスレポート
- **`src/hooks/loader.ts`**: 動的モジュールローダー
- **`src/cli/hooks-cli.ts`**: CLIコマンド
- **`src/gateway/server-startup.ts`**: Gateway起動時にフックをロード
- **`src/auto-reply/reply/commands-core.ts`**: コマンドイベントをトリガー

### 検出フロー

```
Gatewayの起動
    ↓
ディレクトリをスキャン（workspace → managed → bundled）
    ↓
HOOK.mdファイルを解析
    ↓
適格性を確認（bins, env, config, os）
    ↓
対象フックからハンドラーをロード
    ↓
イベントのハンドラーを登録
```

### イベントフロー

```
ユーザーが /new を送信
    ↓
コマンドの検証
    ↓
フックイベントを作成
    ↓
フックをトリガー（登録済みハンドラーすべて）
    ↓
コマンド処理が続く
    ↓
セッションのリセット
```

## トラブルシューティング

### フックが検出されない

1. ディレクトリ構造を確認:

   ```bash
   ls -la ~/.openclaw/hooks/my-hook/
   # 表示されるはず: HOOK.md, handler.ts
   ```

2. HOOK.mdフォーマットを確認:

   ```bash
   cat ~/.openclaw/hooks/my-hook/HOOK.md
   # nameとmetadataを含むYAMLフロントマターがあるはず
   ```

3. 検出されたすべてのフックを一覧表示:

   ```bash
   openclaw hooks list
   ```

### フックが対象でない

要件を確認:

```bash
openclaw hooks info my-hook
```

欠落を確認:

- バイナリ（PATHを確認）
- 環境変数
- 設定値
- OSの互換性

### フックが実行されない

1. フックが有効になっていることを確認:

   ```bash
   openclaw hooks list
   # 有効なフックの横に ✓ が表示されるはず
   ```

2. Gatewayプロセスを再起動してフックを再ロード。

3. エラーのGatewayログを確認:

   ```bash
   ./scripts/clawlog.sh | grep hook
   ```

### ハンドラーエラー

TypeScript/インポートエラーを確認:

```bash
# インポートを直接テスト
node -e "import('./path/to/handler.ts').then(console.log)"
```

## 移行ガイド

### レガシー設定から検出ベースへ

**以前**:

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "handlers": [
        {
          "event": "command:new",
          "module": "./hooks/handlers/my-handler.ts"
        }
      ]
    }
  }
}
```

**以後**:

1. フックディレクトリを作成:

   ```bash
   mkdir -p ~/.openclaw/hooks/my-hook
   mv ./hooks/handlers/my-handler.ts ~/.openclaw/hooks/my-hook/handler.ts
   ```

2. HOOK.mdを作成:

   ```markdown
   ---
   name: my-hook
   description: "My custom hook"
   metadata: { "openclaw": { "emoji": "🎯", "events": ["command:new"] } }
   ---

   # My Hook

   役に立つことをする。
   ```

3. 設定を更新:

   ```json
   {
     "hooks": {
       "internal": {
         "enabled": true,
         "entries": {
           "my-hook": { "enabled": true }
         }
       }
     }
   }
   ```

4. 確認してGatewayプロセスを再起動:

   ```bash
   openclaw hooks list
   # 表示されるはず: 🎯 my-hook ✓
   ```

**移行のメリット**:

- 自動検出
- CLI管理
- 適格性チェック
- より良いドキュメント
- 一貫した構造

## 参照

- [CLIリファレンス: hooks](/cli/hooks)
- [Bundled Hooks README](https://github.com/openclaw/openclaw/tree/main/src/hooks/bundled)
- [Webhook Hooks](/automation/webhook)
- [設定](/gateway/configuration#hooks)
