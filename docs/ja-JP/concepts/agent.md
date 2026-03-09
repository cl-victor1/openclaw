---
summary: "エージェントランタイム（埋め込み pi-mono）、ワークスペース契約、セッションブートストラップ"
read_when:
  - エージェントランタイム、ワークスペースブートストラップ、またはセッション動作を変更する場合
title: "エージェントランタイム"
---

# エージェントランタイム 🤖

OpenClaw は **pi-mono** から派生した単一の埋め込みエージェントランタイムを実行します。

## ワークスペース（必須）

OpenClaw は単一のエージェントワークスペースディレクトリ（`agents.defaults.workspace`）を、ツールとコンテキストのためのエージェントの**唯一の**作業ディレクトリ（`cwd`）として使用します。

推奨：`openclaw setup` を使用して、`~/.openclaw/openclaw.json` が存在しない場合に作成し、ワークスペースファイルを初期化してください。

完全なワークスペースレイアウトとバックアップガイド：[エージェントワークスペース](/concepts/agent-workspace)

`agents.defaults.sandbox` が有効な場合、メインでないセッションは `agents.defaults.sandbox.workspaceRoot` 配下のセッションごとのワークスペースでこれを上書きできます（[ゲートウェイ設定](/gateway/configuration)を参照）。

## ブートストラップファイル（注入済み）

`agents.defaults.workspace` の中で、OpenClaw は以下のユーザー編集可能なファイルを期待します：

- `AGENTS.md` — 動作指示と「メモリ」
- `SOUL.md` — ペルソナ、境界線、トーン
- `TOOLS.md` — ユーザーが管理するツールのメモ（例：`imsg`、`sag`、慣習）
- `BOOTSTRAP.md` — 初回限定の起動儀式（完了後に削除）
- `IDENTITY.md` — エージェント名・雰囲気・絵文字
- `USER.md` — ユーザープロファイルと好ましい呼称

新しいセッションの最初のターンで、OpenClaw はこれらのファイルの内容をエージェントのコンテキストに直接注入します。

空のファイルはスキップされます。大きなファイルはトリミングおよび切り詰められ、プロンプトをスリムに保つためにマーカーが付与されます（完全な内容はファイルを読んでください）。

ファイルが存在しない場合、OpenClaw は単一の「ファイル欠落」マーカー行を注入します（`openclaw setup` は安全なデフォルトテンプレートを作成します）。

`BOOTSTRAP.md` は**新しいワークスペース**（他のブートストラップファイルが存在しない場合）のみ作成されます。儀式の完了後に削除した場合、後の再起動時に再作成されることはありません。

ブートストラップファイルの作成を完全に無効にするには（事前にシードされたワークスペース向け）、以下を設定してください：

```json5
{ agent: { skipBootstrap: true } }
```

## 組み込みツール

コアツール（read/exec/edit/write および関連するシステムツール）は、ツールポリシーに従って常に利用可能です。`apply_patch` はオプションで `tools.exec.applyPatch` によりゲートされています。`TOOLS.md` はどのツールが存在するかを制御しません。これはツールの使用方法に関するガイダンスです。

## スキル

OpenClaw は以下の3つの場所からスキルを読み込みます（名前が競合した場合はワークスペースが優先）：

- バンドル済み（インストールと共に出荷）
- 管理/ローカル：`~/.openclaw/skills`
- ワークスペース：`<workspace>/skills`

スキルはconfig/envによってゲートできます（[ゲートウェイ設定](/gateway/configuration)の `skills` を参照）。

## pi-mono との統合

OpenClaw は pi-mono コードベースの一部（モデル/ツール）を再利用しますが、**セッション管理、ディスカバリ、ツールの配線は OpenClaw 所有です**。

- pi-coding エージェントランタイムはありません。
- `~/.pi/agent` や `<workspace>/.pi` の設定は参照されません。

## セッション

セッションのトランスクリプトは JSONL として以下に保存されます：

- `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`

セッション ID は安定しており、OpenClaw によって選択されます。
レガシーの Pi/Tau セッションフォルダは**読み取られません**。

## ストリーミング中の操舵

キューモードが `steer` の場合、受信メッセージは現在の実行に注入されます。
キューは**各ツール呼び出しの後**にチェックされます。キューにメッセージがある場合、現在のアシスタントメッセージの残りのツール呼び出しはスキップされ（エラーツール結果に「Skipped due to queued user message.」が付与）、次のアシスタントレスポンスの前にキューに入ったユーザーメッセージが注入されます。

キューモードが `followup` または `collect` の場合、受信メッセージは現在のターンが終了するまで保留され、その後キューに入ったペイロードで新しいエージェントターンが開始されます。モードとデバウンス/キャップの動作については[キュー](/concepts/queue)を参照してください。

ブロックストリーミングは完了したアシスタントブロックをすぐに送信します。デフォルトでは**オフ**（`agents.defaults.blockStreamingDefault: "off"`）です。
境界を `agents.defaults.blockStreamingBreak`（`text_end` vs `message_end`；デフォルトは text_end）で調整できます。
ソフトブロックのチャンキングを `agents.defaults.blockStreamingChunk` で制御できます（デフォルトは 800〜1200 文字；段落区切り、改行、文章の順に優先）。
ストリーミングされたチャンクを `agents.defaults.blockStreamingCoalesce` でまとめて単行スパムを削減できます（送信前のアイドルベースのマージ）。非 Telegram チャンネルはブロックリプライを有効にするために明示的に `*.blockStreaming: true` が必要です。
詳細なツールサマリーはツール開始時に発行されます（デバウンスなし）。コントロール UI は利用可能な場合にエージェントイベント経由でツール出力をストリーミングします。
詳細：[ストリーミング + チャンキング](/concepts/streaming)。

## モデル参照

設定内のモデル参照（例：`agents.defaults.model` および `agents.defaults.models`）は**最初の** `/` で分割することで解析されます。

- モデルを設定する際は `provider/model` を使用してください。
- モデル ID 自体に `/` が含まれている場合（OpenRouter スタイル）、プロバイダープレフィックスを含めてください（例：`openrouter/moonshotai/kimi-k2`）。
- プロバイダーを省略した場合、OpenClaw はその入力をエイリアスまたは**デフォルトプロバイダー**のモデルとして扱います（モデル ID に `/` がない場合にのみ機能します）。

## 設定（最小限）

最低限、以下を設定してください：

- `agents.defaults.workspace`
- `channels.whatsapp.allowFrom`（強く推奨）

---

_次：[グループチャット](/channels/group-messages)_ 🦞
