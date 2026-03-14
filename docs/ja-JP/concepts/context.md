---
summary: "コンテキスト：モデルが見るもの、構築方法、検査方法"
read_when:
  - OpenClaw における「コンテキスト」の意味を理解したいとき
  - モデルが何かを「知っている」（または忘れた）理由をデバッグしているとき
  - コンテキストのオーバーヘッドを削減したいとき（/context、/status、/compact）
title: "コンテキスト"
---

# コンテキスト

「コンテキスト」とは、**OpenClaw が1回の実行でモデルに送信するすべてのもの**です。モデルの**コンテキストウィンドウ**（トークン制限）によって制限されます。

初心者向けのメンタルモデル：

- **システムプロンプト**（OpenClaw が構築）：ルール、ツール、スキルリスト、時刻/ランタイム、および注入されたワークスペースファイル。
- **会話履歴**：あなたのメッセージ + このセッションのアシスタントのメッセージ。
- **ツール呼び出し/結果 + 添付ファイル**：コマンド出力、ファイル読み取り、画像/音声など。

コンテキストは「メモリ」と_同じものではありません_：メモリはディスクに保存して後で再読み込みできますが、コンテキストはモデルの現在のウィンドウ内にあるものです。

## クイックスタート（コンテキストを検査する）

- `/status` → 「ウィンドウはどれくらい埋まっているか？」のクイックビュー + セッション設定。
- `/context list` → 注入されているもの + おおよそのサイズ（ファイルごと + 合計）。
- `/context detail` → より詳細な内訳：ファイルごと、ツールスキーマごとのサイズ、スキルエントリごとのサイズ、システムプロンプトのサイズ。
- `/usage tokens` → 通常の返信に1返信ごとの使用量フッターを追加。
- `/compact` → 古い履歴をコンパクトなエントリにまとめてウィンドウスペースを解放。

関連情報：[スラッシュコマンド](/tools/slash-commands)、[トークン使用量とコスト](/reference/token-use)、[コンパクション](/concepts/compaction)。

## 出力例

値はモデル、プロバイダー、ツールポリシー、ワークスペースの内容によって異なります。

### `/context list`

```
🧠 Context breakdown
Workspace: <workspaceDir>
Bootstrap max/file: 20,000 chars
Sandbox: mode=non-main sandboxed=false
System prompt (run): 38,412 chars (~9,603 tok) (Project Context 23,901 chars (~5,976 tok))

Injected workspace files:
- AGENTS.md: OK | raw 1,742 chars (~436 tok) | injected 1,742 chars (~436 tok)
- SOUL.md: OK | raw 912 chars (~228 tok) | injected 912 chars (~228 tok)
- TOOLS.md: TRUNCATED | raw 54,210 chars (~13,553 tok) | injected 20,962 chars (~5,241 tok)
- IDENTITY.md: OK | raw 211 chars (~53 tok) | injected 211 chars (~53 tok)
- USER.md: OK | raw 388 chars (~97 tok) | injected 388 chars (~97 tok)
- HEARTBEAT.md: MISSING | raw 0 | injected 0
- BOOTSTRAP.md: OK | raw 0 chars (~0 tok) | injected 0 chars (~0 tok)

Skills list (system prompt text): 2,184 chars (~546 tok) (12 skills)
Tools: read, edit, write, exec, process, browser, message, sessions_send, …
Tool list (system prompt text): 1,032 chars (~258 tok)
Tool schemas (JSON): 31,988 chars (~7,997 tok) (counts toward context; not shown as text)
Tools: (same as above)

Session tokens (cached): 14,250 total / ctx=32,000
```

### `/context detail`

```
🧠 Context breakdown (detailed)
…
Top skills (prompt entry size):
- frontend-design: 412 chars (~103 tok)
- oracle: 401 chars (~101 tok)
… (+10 more skills)

Top tools (schema size):
- browser: 9,812 chars (~2,453 tok)
- exec: 6,240 chars (~1,560 tok)
… (+N more tools)
```

## コンテキストウィンドウにカウントされるもの

モデルが受け取るすべてのものがカウントされます。以下を含みます：

- システムプロンプト（すべてのセクション）。
- 会話履歴。
- ツール呼び出し + ツール結果。
- 添付ファイル/トランスクリプト（画像/音声/ファイル）。
- コンパクションのサマリーとプルーニングの成果物。
- プロバイダーの「ラッパー」または隠しヘッダー（表示されないが、カウントされます）。

## OpenClaw がシステムプロンプトを構築する方法

システムプロンプトは **OpenClaw が所有**しており、実行ごとに再構築されます。以下を含みます：

- ツールリスト + 短い説明。
- スキルリスト（メタデータのみ；以下を参照）。
- ワークスペースの場所。
- 時刻（UTC + 設定されている場合はユーザー時刻に変換）。
- ランタイムメタデータ（ホスト/OS/モデル/思考）。
- **プロジェクトコンテキスト**の下に注入されたワークスペースのブートストラップファイル。

完全な内訳：[システムプロンプト](/concepts/system-prompt)。

## 注入されたワークスペースファイル（プロジェクトコンテキスト）

デフォルトでは、OpenClaw は固定のワークスペースファイルセットを注入します（存在する場合）：

- `AGENTS.md`
- `SOUL.md`
- `TOOLS.md`
- `IDENTITY.md`
- `USER.md`
- `HEARTBEAT.md`
- `BOOTSTRAP.md`（初回のみ）

大きなファイルは `agents.defaults.bootstrapMaxChars`（デフォルト `20000` 文字）を使用してファイルごとに切り捨てられます。OpenClaw はまた、`agents.defaults.bootstrapTotalMaxChars`（デフォルト `150000` 文字）でファイル全体の合計ブートストラップ注入上限を適用します。`/context` は**生のサイズ vs 注入サイズ**と切り捨てが発生したかどうかを表示します。

切り捨てが発生した場合、ランタイムはプロジェクトコンテキストの下にプロンプト内警告ブロックを注入できます。これを `agents.defaults.bootstrapPromptTruncationWarning`（`off`、`once`、`always`；デフォルト `once`）で設定します。

## スキル：注入されるもの vs オンデマンドで読み込まれるもの

システムプロンプトにはコンパクトな**スキルリスト**（名前 + 説明 + 場所）が含まれています。このリストには実際のオーバーヘッドがあります。

スキルの指示はデフォルトでは_含まれません_。モデルは**必要なときだけ**スキルの `SKILL.md` を `read` することが期待されています。

## ツール：2種類のコスト

ツールはコンテキストに2つの方法で影響します：

1. システムプロンプトの**ツールリストテキスト**（「Tooling」として見えるもの）。
2. **ツールスキーマ**（JSON）。これらはモデルがツールを呼び出せるようにモデルに送信されます。プレーンテキストとして見えなくても、コンテキストにカウントされます。

`/context detail` は最大のツールスキーマを分解するので、何が支配的かを確認できます。

## コマンド、ディレクティブ、「インラインショートカット」

スラッシュコマンドは Gateway によって処理されます。いくつかの異なる動作があります：

- **スタンドアロンコマンド**：`/...` だけのメッセージはコマンドとして実行されます。
- **ディレクティブ**：`/think`、`/verbose`、`/reasoning`、`/elevated`、`/model`、`/queue` はモデルがメッセージを見る前に削除されます。
  - ディレクティブのみのメッセージはセッション設定を持続させます。
  - 通常のメッセージ内のインラインディレクティブはメッセージごとのヒントとして機能します。
- **インラインショートカット**（許可リスト内の送信者のみ）：通常のメッセージ内の特定の `/...` トークンはすぐに実行され（例：「hey /status」）、残りのテキストをモデルが見る前に削除されます。

詳細：[スラッシュコマンド](/tools/slash-commands)。

## セッション、コンパクション、プルーニング（何が持続するか）

メッセージ間で何が持続するかはメカニズムによって異なります：

- **通常の履歴**は、ポリシーによってコンパクト/プルーニングされるまでセッショントランスクリプトに持続します。
- **コンパクション**はサマリーをトランスクリプトに持続させ、最近のメッセージはそのままにします。
- **プルーニング**は実行の_インメモリ_プロンプトから古いツール結果を削除しますが、トランスクリプトは書き換えません。

ドキュメント：[セッション](/concepts/session)、[コンパクション](/concepts/compaction)、[セッションプルーニング](/concepts/session-pruning)。

デフォルトでは、OpenClaw はアセンブリとコンパクションに組み込みの `legacy` コンテキストエンジンを使用します。`kind: "context-engine"` を提供するプラグインをインストールして `plugins.slots.contextEngine` で選択すると、OpenClaw はコンテキストアセンブリ、`/compact`、および関連するサブエージェントコンテキストライフサイクルフックをそのエンジンに委譲します。

## `/context` が実際に報告するもの

`/context` は利用可能な場合、最新の**実行時構築**システムプロンプトレポートを優先します：

- `System prompt (run)` = 最後の組み込み（ツール対応）実行から取得され、セッションストアに持続されます。
- `System prompt (estimate)` = 実行レポートが存在しない場合（またはレポートを生成しない CLI バックエンドで実行する場合）にその場で計算されます。

いずれの場合も、サイズとトップ貢献者を報告します；完全なシステムプロンプトやツールスキーマは**ダンプされません**。
