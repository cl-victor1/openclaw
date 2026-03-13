---
title: "メモリ"
summary: "OpenClaw のメモリの仕組み（ワークスペースファイル + 自動メモリフラッシュ）"
read_when:
  - メモリファイルのレイアウトとワークフローを確認したいとき
  - 自動プレコンパクションメモリフラッシュを調整したいとき
---

# メモリ

OpenClaw のメモリは**エージェントワークスペース内のプレーン Markdown**です。ファイルが唯一の情報源であり、モデルはディスクに書き込まれた内容のみを「記憶」します。

メモリ検索ツールはアクティブなメモリプラグイン（デフォルト: `memory-core`）によって提供されます。メモリプラグインを無効にするには `plugins.slots.memory = "none"` を設定してください。

## メモリファイル（Markdown）

デフォルトのワークスペースレイアウトでは 2 層のメモリを使用します。

- `memory/YYYY-MM-DD.md`
  - 日次ログ（追記専用）。
  - セッション開始時に今日と昨日のログを読み込む。
- `MEMORY.md`（任意）
  - 厳選された長期メモリ。
  - **メインのプライベートセッションにのみ読み込む**（グループコンテキストでは使用しない）。

これらのファイルはワークスペース（`agents.defaults.workspace`、デフォルト `~/.openclaw/workspace`）下に保存されます。完全なレイアウトは [エージェントワークスペース](/concepts/agent-workspace) を参照してください。

## メモリツール

OpenClaw はこれらの Markdown ファイルに対して、エージェント向けの 2 つのツールを公開しています。

- `memory_search` — インデックス化されたスニペットに対するセマンティック検索。
- `memory_get` — 特定の Markdown ファイル/行範囲の対象読み取り。

`memory_get` は、ファイルが存在しない場合（例えば、最初の書き込み前の今日の日次ログ）でも**グレースフルに動作するようになりました**。ビルトインマネージャーと QMD バックエンドのどちらも `ENOENT` をスローする代わりに `{ text: "", path }` を返すため、エージェントはツール呼び出しを try/catch でラップせずに「まだ記録なし」を処理してワークフローを継続できます。

## メモリへの書き込みタイミング

- 決定事項、設定、永続的な事実は `MEMORY.md` に書き込む。
- 日々のメモや実行中のコンテキストは `memory/YYYY-MM-DD.md` に書き込む。
- 「これを覚えておいて」と言われたら書き留める（RAM に保持しない）。
- この領域はまだ進化中です。モデルにメモリを保存するよう促すと効果的です。
- 何かを定着させたい場合は、**ボットに書き込むよう依頼してください**。

## 自動メモリフラッシュ（プレコンパクション ping）

セッションが**自動コンパクションに近づく**と、OpenClaw はコンテキストがコンパクトされる**前に**モデルが永続的なメモリを書き込むことを促す**サイレントなエージェントターン**をトリガーします。デフォルトのプロンプトはモデルが返信しても_よい_と明示していますが、通常 `NO_REPLY` が正しい応答であり、ユーザーにはこのターンが見えません。

これは `agents.defaults.compaction.memoryFlush` で制御されます。

```json5
{
  agents: {
    defaults: {
      compaction: {
        reserveTokensFloor: 20000,
        memoryFlush: {
          enabled: true,
          softThresholdTokens: 4000,
          systemPrompt: "Session nearing compaction. Store durable memories now.",
          prompt: "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store.",
        },
      },
    },
  },
}
```

詳細：

- **ソフト閾値**: セッションのトークン推定値が `contextWindow - reserveTokensFloor - softThresholdTokens` を超えるとフラッシュがトリガーされます。
- **デフォルトではサイレント**: プロンプトに `NO_REPLY` が含まれているため、何も配信されません。
- **2 つのプロンプト**: ユーザープロンプトとシステムプロンプトの両方でリマインダーを追加します。
- **コンパクションサイクルごとに 1 回のフラッシュ**（`sessions.json` で追跡）。
- **ワークスペースは書き込み可能である必要があります**: セッションが `workspaceAccess: "ro"` または `"none"` でサンドボックス化されている場合、フラッシュはスキップされます。

コンパクションのライフサイクル全体については [セッション管理 + コンパクション](/reference/session-management-compaction) を参照してください。

## ベクターメモリ検索

OpenClaw は `MEMORY.md` と `memory/*.md` に対して小さなベクターインデックスを構築でき、言葉が異なっていても関連するメモをセマンティッククエリで見つけることができます。

デフォルト設定：

- デフォルトで有効。
- メモリファイルの変更を監視（デバウンス済み）。
- メモリ検索は（トップレベルの `memorySearch` ではなく）`agents.defaults.memorySearch` で設定。
- デフォルトではリモート埋め込みを使用。`memorySearch.provider` が設定されていない場合、OpenClaw は自動的に選択します：
  1. `memorySearch.local.modelPath` が設定されており、ファイルが存在する場合は `local`。
  2. OpenAI キーが解決できる場合は `openai`。
  3. Gemini キーが解決できる場合は `gemini`。
  4. Voyage キーが解決できる場合は `voyage`。
  5. Mistral キーが解決できる場合は `mistral`。
  6. それ以外の場合、設定されるまでメモリ検索は無効のまま。
- ローカルモードは node-llama-cpp を使用し、`pnpm approve-builds` が必要な場合があります。
- SQLite 内のベクター検索を高速化するために sqlite-vec（利用可能な場合）を使用。
- `memorySearch.provider = "ollama"` もローカル/セルフホスト Ollama 埋め込み（`/api/embeddings`）でサポートされていますが、自動選択はされません。

リモート埋め込みには埋め込みプロバイダーの API キーが**必要です**。OpenClaw は認証プロファイル、`models.providers.*.apiKey`、または環境変数からキーを解決します。Codex OAuth はチャット/補完のみを対象としており、メモリ検索の埋め込みには**対応していません**。Gemini の場合は `GEMINI_API_KEY` または `models.providers.google.apiKey` を使用してください。Voyage の場合は `VOYAGE_API_KEY` または `models.providers.voyage.apiKey` を使用してください。Mistral の場合は `MISTRAL_API_KEY` または `models.providers.mistral.apiKey` を使用してください。Ollama は通常、実際の API キーを必要としません（ローカルポリシーで必要な場合は `OLLAMA_API_KEY=ollama-local` のようなプレースホルダーで十分です）。
カスタム OpenAI 互換エンドポイントを使用する場合は、`memorySearch.remote.apiKey`（および任意の `memorySearch.remote.headers`）を設定してください。

### QMD バックエンド（実験的）

`memory.backend = "qmd"` を設定すると、ビルトイン SQLite インデクサーを [QMD](https://github.com/tobi/qmd)（BM25 + ベクター + リランキングを組み合わせたローカルファーストの検索サイドカー）に切り替えることができます。Markdown が情報源のままであり、OpenClaw は検索のために QMD にシェルアウトします。主なポイント：

**前提条件**

- デフォルトでは無効。設定ごとにオプトイン（`memory.backend = "qmd"`）。
- QMD CLI を別途インストール（`bun install -g https://github.com/tobi/qmd` またはリリースを取得）し、`qmd` バイナリをゲートウェイの `PATH` に追加。
- QMD は拡張機能を許可する SQLite ビルドが必要（macOS では `brew install sqlite`）。
- QMD は Bun + `node-llama-cpp` を介して完全にローカルで実行され、初回使用時に HuggingFace から GGUF モデルを自動ダウンロード（別途 Ollama デーモンは不要）。
- ゲートウェイは `XDG_CONFIG_HOME` と `XDG_CACHE_HOME` を設定することで、`~/.openclaw/agents/<agentId>/qmd/` 配下の自己完結型 XDG ホームで QMD を実行。
- OS サポート：Bun + SQLite がインストールされていれば macOS と Linux はそのまま動作。Windows は WSL2 経由で最適にサポート。

**サイドカーの動作方法**

- ゲートウェイは `~/.openclaw/agents/<agentId>/qmd/` 配下に自己完結型 QMD ホームを書き込みます（設定 + キャッシュ + sqlite DB）。
- コレクションは `memory.qmd.paths` から `qmd collection add` で作成され（デフォルトのワークスペースメモリファイルも含む）、その後 `qmd update` + `qmd embed` が起動時と設定可能な間隔（`memory.qmd.update.interval`、デフォルト 5 分）で実行されます。
- ゲートウェイは起動時に QMD マネージャーを初期化するため、最初の `memory_search` 呼び出し前でも定期更新タイマーが有効になります。
- デフォルトでは起動リフレッシュがバックグラウンドで実行されるため、チャット起動はブロックされません。以前のブロッキング動作を維持するには `memory.qmd.update.waitForBootSync = true` を設定してください。
- 検索は `memory.qmd.searchMode` で実行されます（デフォルト `qmd search --json`、`vsearch` と `query` もサポート）。選択したモードが QMD ビルドでフラグを拒否した場合、OpenClaw は `qmd query` で再試行します。QMD が失敗するか、バイナリが見つからない場合、OpenClaw は自動的にビルトイン SQLite マネージャーにフォールバックするため、メモリツールは引き続き動作します。
- OpenClaw は現在、QMD の埋め込みバッチサイズの調整を公開していません。バッチ動作は QMD 自体によって制御されます。
- **初回検索は遅い場合があります**：QMD は最初の `qmd query` 実行時にローカル GGUF モデル（リランカー/クエリ拡張）をダウンロードする場合があります。
  - OpenClaw は QMD 実行時に `XDG_CONFIG_HOME`/`XDG_CACHE_HOME` を自動設定します。
  - モデルを手動で事前ダウンロードし、OpenClaw が使用するインデックスをウォームアップしたい場合は、エージェントの XDG ディレクトリを使用してワンオフクエリを実行してください。

    OpenClaw の QMD ステートは**ステートディレクトリ**（デフォルト `~/.openclaw`）に保存されます。
    OpenClaw が使用するのと同じ XDG 変数をエクスポートすることで、`qmd` を同じインデックスに向けることができます：

    ```bash
    # OpenClaw が使用するのと同じステートディレクトリを選択
    STATE_DIR="${OPENCLAW_STATE_DIR:-$HOME/.openclaw}"

    export XDG_CONFIG_HOME="$STATE_DIR/agents/main/qmd/xdg-config"
    export XDG_CACHE_HOME="$STATE_DIR/agents/main/qmd/xdg-cache"

    # （任意）インデックスの更新 + 埋め込みを強制
    qmd update
    qmd embed

    # ウォームアップ / 初回モデルダウンロードのトリガー
    qmd query "test" -c memory-root --json >/dev/null 2>&1
    ```

**設定サーフェス（`memory.qmd.*`）**

- `command`（デフォルト `qmd`）：実行ファイルのパスを上書き。
- `searchMode`（デフォルト `search`）：`memory_search` のバックエンドとなる QMD コマンドを選択（`search`、`vsearch`、`query`）。
- `includeDefaultMemory`（デフォルト `true`）：`MEMORY.md` + `memory/**/*.md` を自動インデックス化。
- `paths[]`：追加のディレクトリ/ファイルを追加（`path`、任意の `pattern`、任意の安定した `name`）。
- `sessions`：セッション JSONL インデックス化をオプトイン（`enabled`、`retentionDays`、`exportDir`）。
- `update`：リフレッシュのリズムとメンテナンス実行を制御：（`interval`、`debounceMs`、`onBoot`、`waitForBootSync`、`embedInterval`、`commandTimeoutMs`、`updateTimeoutMs`、`embedTimeoutMs`）。
- `limits`：検索ペイロードの上限を設定（`maxResults`、`maxSnippetChars`、`maxInjectedChars`、`timeoutMs`）。
- `scope`：[`session.sendPolicy`](/gateway/configuration#session) と同じスキーマ。デフォルトは DM のみ（全て `deny`、ダイレクトチャットのみ `allow`）。グループ/チャンネルで QMD ヒットを表示するには緩和してください。
  - `match.keyPrefix` は**正規化された**セッションキー（小文字化、先頭の `agent:<id>:` を除去）に一致します。例：`discord:channel:`。
  - `match.rawKeyPrefix` は`agent:<id>:` を含む**生の**セッションキー（小文字化）に一致します。例：`agent:main:discord:`。
  - レガシー：`match.keyPrefix: "agent:..."` は引き続き生のキープレフィックスとして扱われますが、明確さのために `rawKeyPrefix` を推奨します。
- `scope` が検索を拒否した場合、OpenClaw は空の結果をデバッグしやすいよう導出された `channel`/`chatType` を含む警告をログに記録します。
- ワークスペース外からのスニペットは `memory_search` の結果に `qmd/<collection>/<relative-path>` として表示されます。`memory_get` はそのプレフィックスを理解し、設定済みの QMD コレクションルートから読み取ります。
- `memory.qmd.sessions.enabled = true` の場合、OpenClaw はサニタイズされたセッションのトランスクリプト（ユーザー/アシスタントのターン）を `~/.openclaw/agents/<id>/qmd/sessions/` 配下の専用 QMD コレクションにエクスポートするため、ビルトイン SQLite インデックスに触れることなく `memory_search` で最近の会話を検索できます。
- `memory.citations` が `auto`/`on` の場合、`memory_search` のスニペットには `Source: <path#line>` フッターが含まれます。`memory.citations = "off"` を設定するとパスメタデータを内部に保持します（エージェントは `memory_get` 用のパスを受け取りますが、スニペットテキストにはフッターが含まれず、システムプロンプトがエージェントに引用しないよう警告します）。

**例**

```json5
memory: {
  backend: "qmd",
  citations: "auto",
  qmd: {
    includeDefaultMemory: true,
    update: { interval: "5m", debounceMs: 15000 },
    limits: { maxResults: 6, timeoutMs: 4000 },
    scope: {
      default: "deny",
      rules: [
        { action: "allow", match: { chatType: "direct" } },
        // 正規化されたセッションキープレフィックス（`agent:<id>:` を除去）。
        { action: "deny", match: { keyPrefix: "discord:channel:" } },
        // 生のセッションキープレフィックス（`agent:<id>:` を含む）。
        { action: "deny", match: { rawKeyPrefix: "agent:main:discord:" } },
      ]
    },
    paths: [
      { name: "docs", path: "~/notes", pattern: "**/*.md" }
    ]
  }
}
```

**引用とフォールバック**

- `memory.citations` はバックエンドに関わらず適用されます（`auto`/`on`/`off`）。
- `qmd` が実行されると `status().backend = "qmd"` とタグ付けされるため、診断でどのエンジンが結果を提供したかが分かります。QMD サブプロセスが終了するか、JSON 出力が解析できない場合、検索マネージャーは警告をログに記録し、QMD が回復するまでビルトインプロバイダー（既存の Markdown 埋め込み）を返します。

### 追加メモリパス

デフォルトのワークスペースレイアウト以外の場所にある Markdown ファイルをインデックス化したい場合は、明示的なパスを追加してください：

```json5
agents: {
  defaults: {
    memorySearch: {
      extraPaths: ["../team-docs", "/srv/shared-notes/overview.md"]
    }
  }
}
```

注意事項：

- パスは絶対パスまたはワークスペース相対パスで指定できます。
- ディレクトリは `.md` ファイルを再帰的にスキャンします。
- Markdown ファイルのみがインデックス化されます。
- シンボリックリンクは無視されます（ファイルとディレクトリの両方）。

### Gemini 埋め込み（ネイティブ）

プロバイダーを `gemini` に設定すると、Gemini 埋め込み API を直接使用できます：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "gemini",
      model: "gemini-embedding-001",
      remote: {
        apiKey: "YOUR_GEMINI_API_KEY"
      }
    }
  }
}
```

注意事項：

- `remote.baseUrl` は任意です（デフォルトは Gemini API ベース URL）。
- `remote.headers` を使用して、必要に応じて追加ヘッダーを追加できます。
- デフォルトモデル：`gemini-embedding-001`。

**カスタム OpenAI 互換エンドポイント**（OpenRouter、vLLM、またはプロキシ）を使用したい場合は、OpenAI プロバイダーで `remote` 設定を使用できます：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_OPENAI_COMPAT_API_KEY",
        headers: { "X-Custom-Header": "value" }
      }
    }
  }
}
```

API キーを設定したくない場合は、`memorySearch.provider = "local"` を使用するか、`memorySearch.fallback = "none"` を設定してください。

フォールバック：

- `memorySearch.fallback` には `openai`、`gemini`、`voyage`、`mistral`、`ollama`、`local`、または `none` を指定できます。
- フォールバックプロバイダーは、プライマリ埋め込みプロバイダーが失敗した場合にのみ使用されます。

バッチインデックス化（OpenAI + Gemini + Voyage）：

- デフォルトでは無効。大容量コーパスのインデックス化（OpenAI、Gemini、Voyage）を有効にするには `agents.defaults.memorySearch.remote.batch.enabled = true` を設定してください。
- デフォルトの動作はバッチ完了を待機します。必要に応じて `remote.batch.wait`、`remote.batch.pollIntervalMs`、`remote.batch.timeoutMinutes` を調整してください。
- `remote.batch.concurrency` を設定して、並行して送信するバッチジョブの数を制御できます（デフォルト：2）。
- バッチモードは `memorySearch.provider = "openai"` または `"gemini"` のときに適用され、対応する API キーを使用します。
- Gemini バッチジョブは非同期埋め込みバッチエンドポイントを使用し、Gemini Batch API の利用可能性が必要です。

OpenAI バッチが速くて安価な理由：

- 大規模なバックフィルの場合、OpenAI は通常、私たちがサポートする最速のオプションです。多数の埋め込みリクエストを単一のバッチジョブに送信し、OpenAI に非同期で処理させることができるためです。
- OpenAI はバッチ API ワークロードに割引価格を提供しているため、大規模なインデックス化実行は通常、同じリクエストを同期的に送信するよりも安価です。
- 詳細については OpenAI バッチ API ドキュメントと価格ページを参照してください：
  - [https://platform.openai.com/docs/api-reference/batch](https://platform.openai.com/docs/api-reference/batch)
  - [https://platform.openai.com/pricing](https://platform.openai.com/pricing)

設定例：

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      fallback: "openai",
      remote: {
        batch: { enabled: true, concurrency: 2 }
      },
      sync: { watch: true }
    }
  }
}
```

ツール：

- `memory_search` — ファイル + 行範囲付きのスニペットを返す。
- `memory_get` — パスによるメモリファイルコンテンツの読み取り。

ローカルモード：

- `agents.defaults.memorySearch.provider = "local"` を設定。
- `agents.defaults.memorySearch.local.modelPath` を指定（GGUF または `hf:` URI）。
- 任意：リモートフォールバックを避けるために `agents.defaults.memorySearch.fallback = "none"` を設定。

### メモリツールの仕組み

- `memory_search` は `MEMORY.md` + `memory/**/*.md` の Markdown チャンク（約 400 トークン目標、80 トークンのオーバーラップ）に対してセマンティック検索を行います。スニペットテキスト（最大約 700 文字）、ファイルパス、行範囲、スコア、プロバイダー/モデル、ローカル → リモート埋め込みへのフォールバックの有無を返します。完全なファイルペイロードは返しません。
- `memory_get` は特定のメモリ Markdown ファイル（ワークスペース相対）を読み取ります。開始行から N 行を指定できます。`MEMORY.md` / `memory/` 以外のパスは拒否されます。
- 両ツールは、エージェントに対して `memorySearch.enabled` が true と解決された場合にのみ有効です。

### インデックス化されるもの（とタイミング）

- ファイルタイプ：Markdown のみ（`MEMORY.md`、`memory/**/*.md`）。
- インデックスストレージ：`~/.openclaw/memory/<agentId>.sqlite` のエージェントごとの SQLite（`agents.defaults.memorySearch.store.path` で設定可能、`{agentId}` トークンをサポート）。
- 鮮度：`MEMORY.md` + `memory/` のウォッチャーがインデックスをダーティとマーク（デバウンス 1.5 秒）。同期はセッション開始時、検索時、または間隔でスケジュールされ、非同期で実行されます。セッションのトランスクリプトはデルタ閾値を使用してバックグラウンド同期をトリガーします。
- 再インデックストリガー：インデックスは埋め込み**プロバイダー/モデル + エンドポイントフィンガープリント + チャンキングパラメータ**を保存します。これらが変更されると、OpenClaw は自動的にストア全体をリセットして再インデックス化します。

### ハイブリッド検索（BM25 + ベクター）

有効にすると、OpenClaw は以下を組み合わせます：

- **ベクター類似性**（セマンティックマッチ、言葉が異なっていても可）
- **BM25 キーワード関連性**（ID、環境変数、コードシンボルなどの正確なトークン）

フルテキスト検索がプラットフォームで利用できない場合、OpenClaw はベクターのみの検索にフォールバックします。

#### なぜハイブリッドなのか？

ベクター検索は「同じ意味のもの」に優れています：

- 「Mac Studio ゲートウェイホスト」vs「ゲートウェイを実行しているマシン」
- 「ファイル更新をデバウンスする」vs「毎回の書き込みでインデックス化を避ける」

しかし、正確な高シグナルトークンでは弱い場合があります：

- ID（`a828e60`、`b3b9895a…`）
- コードシンボル（`memorySearch.query.hybrid`）
- エラー文字列（"sqlite-vec unavailable"）

BM25（フルテキスト）はその逆：正確なトークンに強く、言い換えには弱い。
ハイブリッド検索は実用的な中間点です：**両方の検索シグナルを使用**することで、「自然言語」クエリと「干し草の山の中の針」クエリの両方で良い結果を得られます。

#### 結果のマージ方法（現在の設計）

実装の概要：

1. 両側から候補プールを取得：

- **ベクター**：コサイン類似度による上位 `maxResults * candidateMultiplier`。
- **BM25**：FTS5 BM25 ランク（低いほど良い）による上位 `maxResults * candidateMultiplier`。

2. BM25 ランクを 0..1 のスコアに変換：

- `textScore = 1 / (1 + max(0, bm25Rank))`

3. チャンク ID で候補を統合し、加重スコアを計算：

- `finalScore = vectorWeight * vectorScore + textWeight * textScore`

注意事項：

- `vectorWeight` + `textWeight` は設定解決時に 1.0 に正規化されるため、重みはパーセンテージとして機能します。
- 埋め込みが利用できない場合（またはプロバイダーがゼロベクターを返す場合）でも、BM25 を実行してキーワードマッチを返します。
- FTS5 を作成できない場合は、ベクターのみの検索を維持します（ハードな失敗はありません）。

これは「IR 理論的に完璧」ではありませんが、シンプルで高速であり、実際のメモに対して再現率/精度を改善する傾向があります。
さらに高度にしたい場合、一般的な次のステップは Reciprocal Rank Fusion（RRF）またはスコア正規化（最小/最大または z スコア）を混合前に適用することです。

#### ポストプロセスパイプライン

ベクターとキーワードのスコアをマージした後、2 つの任意のポストプロセスステージがエージェントに届く前に結果リストを改善します：

```
ベクター + キーワード → 加重マージ → 時間的減衰 → ソート → MMR → トップ K 結果
```

両ステージは**デフォルトで無効**で、独立して有効化できます。

#### MMR 再ランキング（多様性）

ハイブリッド検索が結果を返すと、複数のチャンクに類似または重複したコンテンツが含まれる場合があります。
例えば、「ホームネットワーク設定」を検索すると、同じルーター設定について言及している異なる日次メモからほぼ同一の 5 つのスニペットが返される場合があります。

**MMR（Maximal Marginal Relevance）**は、関連性と多様性のバランスを取るように結果を再ランキングし、
トップの結果が同じ情報を繰り返すのではなく、クエリの異なる側面をカバーするようにします。

動作方法：

1. 結果は元の関連性スコア（ベクター + BM25 加重スコア）でスコアリングされます。
2. MMR は次の最大化を繰り返し選択します：`λ × 関連性 − (1−λ) × 選択済みとの最大類似性`。
3. 結果間の類似性は、トークン化されたコンテンツに対する Jaccard テキスト類似性を使用して測定されます。

`lambda` パラメータはトレードオフを制御します：

- `lambda = 1.0` → 純粋な関連性（多様性ペナルティなし）
- `lambda = 0.0` → 最大多様性（関連性を無視）
- デフォルト：`0.7`（バランス取れた、わずかに関連性寄り）

**例 — クエリ：「ホームネットワーク設定」**

以下のメモリファイルがあるとします：

```
memory/2026-02-10.md  → "Configured Omada router, set VLAN 10 for IoT devices"
memory/2026-02-08.md  → "Configured Omada router, moved IoT to VLAN 10"
memory/2026-02-05.md  → "Set up AdGuard DNS on 192.168.10.2"
memory/network.md     → "Router: Omada ER605, AdGuard: 192.168.10.2, VLAN 10: IoT"
```

MMR なし — トップ 3 の結果：

```
1. memory/2026-02-10.md  (score: 0.92)  ← ルーター + VLAN
2. memory/2026-02-08.md  (score: 0.89)  ← ルーター + VLAN（ほぼ重複！）
3. memory/network.md     (score: 0.85)  ← リファレンスドキュメント
```

MMR あり（λ=0.7）— トップ 3 の結果：

```
1. memory/2026-02-10.md  (score: 0.92)  ← ルーター + VLAN
2. memory/network.md     (score: 0.85)  ← リファレンスドキュメント（多様！）
3. memory/2026-02-05.md  (score: 0.78)  ← AdGuard DNS（多様！）
```

2 月 8 日のほぼ重複したメモが除外され、エージェントは 3 つの異なる情報を取得します。

**有効化のタイミング：** 特に毎日似たような情報を繰り返す日次メモで、`memory_search` が冗長またはほぼ重複したスニペットを返す場合。

#### 時間的減衰（最近性ブースト）

日次メモを持つエージェントは時間とともに何百もの日付付きファイルを蓄積します。減衰なしでは、6 か月前の適切に表現されたメモが同じトピックの昨日の更新よりも上位になる可能性があります。

**時間的減衰**は、各結果の年齢に基づいてスコアに指数的な乗数を適用し、古いものが徐々に薄れる一方で最近のメモが自然に高くランクされるようにします：

```
decayedScore = score × e^(-λ × ageInDays)
```

ここで `λ = ln(2) / halfLifeDays`。

デフォルトの半減期 30 日の場合：

- 今日のメモ：元のスコアの **100%**
- 7 日前：**約 84%**
- 30 日前：**50%**
- 90 日前：**12.5%**
- 180 日前：**約 1.6%**

**エバーグリーンファイルは決して減衰しません：**

- `MEMORY.md`（ルートメモリファイル）
- `memory/` 内の日付なしファイル（例：`memory/projects.md`、`memory/network.md`）
- これらは常に通常どおりランクされるべき永続的な参照情報を含んでいます。

**日付付き日次ファイル**（`memory/YYYY-MM-DD.md`）はファイル名から抽出された日付を使用します。
その他のソース（例：セッションのトランスクリプト）はファイルの変更時刻（`mtime`）にフォールバックします。

**例 — クエリ：「Rod の仕事のスケジュールは？」**

以下のメモリファイルがあるとします（今日は 2 月 10 日）：

```
memory/2025-09-15.md  → "Rod works Mon-Fri, standup at 10am, pairing at 2pm"  (148日前)
memory/2026-02-10.md  → "Rod has standup at 14:15, 1:1 with Zeb at 14:45"    (今日)
memory/2026-02-03.md  → "Rod started new team, standup moved to 14:15"        (7日前)
```

減衰なし：

```
1. memory/2025-09-15.md  (score: 0.91)  ← 最高のセマンティックマッチだが古い！
2. memory/2026-02-10.md  (score: 0.82)
3. memory/2026-02-03.md  (score: 0.80)
```

減衰あり（halfLife=30）：

```
1. memory/2026-02-10.md  (score: 0.82 × 1.00 = 0.82)  ← 今日、減衰なし
2. memory/2026-02-03.md  (score: 0.80 × 0.85 = 0.68)  ← 7日前、わずかな減衰
3. memory/2025-09-15.md  (score: 0.91 × 0.03 = 0.03)  ← 148日前、ほぼ消滅
```

9 月の古いメモは最高の生のセマンティックマッチを持ちながらも最下位に落ちます。

**有効化のタイミング：** エージェントが数か月分の日次メモを持っており、古い情報が最近のコンテキストより上位にランクされると感じる場合。半減期 30 日は日次メモが多いワークフローに適しています。古いメモを頻繁に参照する場合は増やしてください（例：90 日）。

#### 設定

両機能は `memorySearch.query.hybrid` で設定されます：

```json5
agents: {
  defaults: {
    memorySearch: {
      query: {
        hybrid: {
          enabled: true,
          vectorWeight: 0.7,
          textWeight: 0.3,
          candidateMultiplier: 4,
          // 多様性：冗長な結果を減らす
          mmr: {
            enabled: true,    // デフォルト: false
            lambda: 0.7       // 0 = 最大多様性, 1 = 最大関連性
          },
          // 最近性：新しいメモリをブースト
          temporalDecay: {
            enabled: true,    // デフォルト: false
            halfLifeDays: 30  // 30 日ごとにスコアが半分になる
          }
        }
      }
    }
  }
}
```

各機能を独立して有効化できます：

- **MMR のみ** — 多くの類似したメモがあるが年齢は重要でない場合に有効。
- **時間的減衰のみ** — 最近性が重要だが結果がすでに多様な場合に有効。
- **両方** — 長期間の日次メモ履歴を持つエージェントに推奨。

### 埋め込みキャッシュ

OpenClaw は**チャンク埋め込み**を SQLite にキャッシュできるため、変更されていないテキストの再インデックス化や頻繁な更新（特にセッションのトランスクリプト）で再埋め込みが不要になります。

設定：

```json5
agents: {
  defaults: {
    memorySearch: {
      cache: {
        enabled: true,
        maxEntries: 50000
      }
    }
  }
}
```

### セッションメモリ検索（実験的）

オプションで**セッションのトランスクリプト**をインデックス化し、`memory_search` で表示できます。
これは実験的フラグの後ろに隠されています。

```json5
agents: {
  defaults: {
    memorySearch: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"]
    }
  }
}
```

注意事項：

- セッションのインデックス化は**オプトイン**（デフォルトで無効）。
- セッションの更新はデバウンスされ、デルタ閾値を超えると**非同期でインデックス化**されます（ベストエフォート）。
- `memory_search` はインデックス化をブロックしません。バックグラウンド同期が完了するまで結果がわずかに古い場合があります。
- 結果にはスニペットのみが含まれます。`memory_get` はメモリファイルに限定されたままです。
- セッションのインデックス化はエージェントごとに分離されています（そのエージェントのセッションログのみがインデックス化されます）。
- セッションログはディスク上に保存されます（`~/.openclaw/agents/<agentId>/sessions/*.jsonl`）。ファイルシステムアクセス権を持つプロセス/ユーザーはそれらを読み取ることができるため、ディスクアクセスを信頼の境界として扱ってください。より厳格な分離のために、エージェントを別々の OS ユーザーまたはホストで実行してください。

デルタ閾値（デフォルト）：

```json5
agents: {
  defaults: {
    memorySearch: {
      sync: {
        sessions: {
          deltaBytes: 100000,   // 約 100 KB
          deltaMessages: 50     // JSONL 行数
        }
      }
    }
  }
}
```

### SQLite ベクター高速化（sqlite-vec）

sqlite-vec 拡張機能が利用可能な場合、OpenClaw は SQLite 仮想テーブル（`vec0`）に埋め込みを保存し、データベース内でベクター距離クエリを実行します。これにより、すべての埋め込みを JS にロードせずに検索を高速に保てます。

設定（任意）：

```json5
agents: {
  defaults: {
    memorySearch: {
      store: {
        vector: {
          enabled: true,
          extensionPath: "/path/to/sqlite-vec"
        }
      }
    }
  }
}
```

注意事項：

- `enabled` はデフォルトで true。無効にすると、検索は保存された埋め込みに対するインプロセスのコサイン類似性にフォールバックします。
- sqlite-vec 拡張機能が見つからないか、ロードに失敗した場合、OpenClaw はエラーをログに記録し、JS フォールバックで継続します（ベクターテーブルなし）。
- `extensionPath` はバンドルされた sqlite-vec パスを上書きします（カスタムビルドや標準外のインストール場所に便利）。

### ローカル埋め込みの自動ダウンロード

- デフォルトのローカル埋め込みモデル：`hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf`（約 0.6 GB）。
- `memorySearch.provider = "local"` の場合、`node-llama-cpp` が `modelPath` を解決します。GGUF が見つからない場合は、キャッシュ（または `local.modelCacheDir` が設定されている場合はそのディレクトリ）に**自動ダウンロード**してからロードします。ダウンロードは再試行時に再開されます。
- ネイティブビルドの要件：`pnpm approve-builds` を実行し、`node-llama-cpp` を選択して `pnpm rebuild node-llama-cpp` を実行。
- フォールバック：ローカルセットアップが失敗し、`memorySearch.fallback = "openai"` の場合、自動的にリモート埋め込み（`openai/text-embedding-3-small`、上書きしない限り）に切り替え、その理由を記録します。

### カスタム OpenAI 互換エンドポイントの例

```json5
agents: {
  defaults: {
    memorySearch: {
      provider: "openai",
      model: "text-embedding-3-small",
      remote: {
        baseUrl: "https://api.example.com/v1/",
        apiKey: "YOUR_REMOTE_API_KEY",
        headers: {
          "X-Organization": "org-id",
          "X-Project": "project-id"
        }
      }
    }
  }
}
```

注意事項：

- `remote.*` は `models.providers.openai.*` よりも優先されます。
- `remote.headers` は OpenAI ヘッダーとマージされます。キーが競合する場合は remote が優先されます。OpenAI のデフォルトを使用するには `remote.headers` を省略してください。
