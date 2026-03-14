---
summary: "Skills：管理済み vs ワークスペース、ゲートルール、設定/環境変数の連携"
read_when:
  - Skills を追加または変更する場合
  - Skills のゲートまたは読み込みルールを変更する場合
title: "Skills"
---

# Skills（OpenClaw）

OpenClaw は **[AgentSkills](https://agentskills.io) 互換**の Skills フォルダーを使用して、エージェントにツールの使い方を教えます。各 Skill は YAML フロントマターと説明を含む `SKILL.md` が入ったディレクトリです。OpenClaw は**バンドル済み Skills** とオプションのローカルオーバーライドを読み込み、環境、設定、バイナリの存在に基づいて読み込み時にフィルタリングします。

## 場所と優先順位

Skills は**3 つ**の場所から読み込まれます：

1. **バンドル済み Skills**：インストールと一緒に配布（npm パッケージまたは OpenClaw.app）
2. **管理済み/ローカル Skills**：`~/.openclaw/skills`
3. **ワークスペース Skills**：`<workspace>/skills`

Skill 名が競合する場合、優先順位は次のとおりです：

`<workspace>/skills`（最高）→ `~/.openclaw/skills` → バンドル済み Skills（最低）

さらに、`~/.openclaw/openclaw.json` の `skills.load.extraDirs` を使用して追加の Skills フォルダーを設定できます（最低優先順位）。

## エージェントごとの Skills vs 共有 Skills

**マルチエージェント**設定では、各エージェントは独自のワークスペースを持ちます。つまり：

- **エージェントごとの Skills** は `<workspace>/skills` に配置され、そのエージェントのみが使用できます。
- **共有 Skills** は `~/.openclaw/skills`（管理済み/ローカル）に配置され、同一マシン上の**すべてのエージェント**から参照できます。
- 複数のエージェントで共通の Skills パックを使用したい場合は、`skills.load.extraDirs`（最低優先順位）で**共有フォルダー**を追加することもできます。

同一の Skill 名が複数の場所に存在する場合、通常の優先順位が適用されます：ワークスペースが優先され、次に管理済み/ローカル、最後にバンドル済みです。

## プラグイン + Skills

プラグインは `openclaw.plugin.json` に `skills` ディレクトリを列挙することで（プラグインルートからの相対パス）、独自の Skills を配布できます。プラグイン Skills はプラグインが有効になったときに読み込まれ、通常の Skill 優先順位ルールに参加します。プラグインの設定エントリの `metadata.openclaw.requires.config` でゲートを設定できます。探索/設定については [プラグイン](/tools/plugin)、それらの Skills が教えるツール面については [ツール](/tools) を参照してください。

## ClawHub（インストール + 同期）

ClawHub は OpenClaw の公開 Skills レジストリです。[https://clawhub.com](https://clawhub.com) で閲覧できます。Skills の発見、インストール、更新、バックアップに使用します。
完全なガイド：[ClawHub](/tools/clawhub)。

一般的な操作：

- ワークスペースに Skill をインストールする：
  - `clawhub install <skill-slug>`
- インストール済みの全 Skills を更新する：
  - `clawhub update --all`
- 同期（スキャン + 更新を公開）：
  - `clawhub sync --all`

デフォルトでは、`clawhub` は現在の作業ディレクトリ下の `./skills` にインストールします（または設定済みの OpenClaw ワークスペースにフォールバック）。OpenClaw は次のセッションでそれを `<workspace>/skills` として認識します。

## セキュリティに関する注意

- サードパーティの Skills は**信頼できないコード**として扱ってください。有効にする前に内容を確認してください。
- 信頼できない入力とリスクの高いツールにはサンドボックス化された実行を推奨します。[サンドボックス化](/gateway/sandboxing) を参照してください。
- ワークスペースと extra-dir の Skill 探索は、設定済みルート内に解決されたリアルパスがある Skill ルートと `SKILL.md` ファイルのみを受け入れます。
- `skills.entries.*.env` と `skills.entries.*.apiKey` は、そのエージェントターンの**ホスト**プロセスにシークレットを注入します（サンドボックスではありません）。シークレットをプロンプトやログから遠ざけてください。
- より広範な脅威モデルとチェックリストについては、[セキュリティ](/gateway/security) を参照してください。

## フォーマット（AgentSkills + Pi 互換）

`SKILL.md` には少なくとも次の内容が必要です：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
---
```

注意事項：

- AgentSkills 仕様のレイアウト/意図に従います。
- 組み込みエージェントが使用するパーサーは**単一行**のフロントマターキーのみをサポートします。
- `metadata` は**単一行の JSON オブジェクト**である必要があります。
- 説明内の Skill フォルダーパスを参照するには `{baseDir}` を使用します。
- オプションのフロントマターキー：
  - `homepage` — macOS Skills UI で「Website」として表示される URL（`metadata.openclaw.homepage` 経由でもサポート）。
  - `user-invocable` — `true|false`（デフォルト：`true`）。`true` の場合、Skill はユーザースラッシュコマンドとして公開されます。
  - `disable-model-invocation` — `true|false`（デフォルト：`false`）。`true` の場合、Skill はモデルプロンプトから除外されます（ユーザー呼び出しでは引き続き使用可能）。
  - `command-dispatch` — `tool`（オプション）。`tool` に設定すると、スラッシュコマンドはモデルをバイパスしてツールに直接ディスパッチされます。
  - `command-tool` — `command-dispatch: tool` が設定されているときに呼び出すツール名。
  - `command-arg-mode` — `raw`（デフォルト）。ツールディスパッチの場合、生の引数文字列をツールに転送します（コア解析なし）。

    ツールは次のパラメーターで呼び出されます：
    `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }`。

## ゲート（読み込み時フィルター）

OpenClaw は `metadata`（単一行 JSON）を使用して**読み込み時に Skills をフィルタリング**します：

```markdown
---
name: nano-banana-pro
description: Generate or edit images via Gemini 3 Pro Image
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

`metadata.openclaw` 下のフィールド：

- `always: true` — 常に Skill を含める（他のゲートをスキップ）。
- `emoji` — macOS Skills UI で使用するオプションの絵文字。
- `homepage` — macOS Skills UI で「Website」として表示されるオプションの URL。
- `os` — オプションのプラットフォームリスト（`darwin`、`linux`、`win32`）。設定すると、Skill はそれらの OS でのみ有効になります。
- `requires.bins` — リスト；各項目が `PATH` 上に存在する必要があります。
- `requires.anyBins` — リスト；少なくとも 1 つが `PATH` 上に存在する必要があります。
- `requires.env` — リスト；環境変数が存在するか、設定で提供されている必要があります。
- `requires.config` — 真値である必要がある `openclaw.json` パスのリスト。
- `primaryEnv` — `skills.entries.<name>.apiKey` に関連付けられた環境変数名。
- `install` — macOS Skills UI が使用するオプションのインストーラー仕様の配列（brew/node/go/uv/download）。

サンドボックスに関する注意：

- `requires.bins` は Skill 読み込み時に**ホスト**上で確認されます。
- エージェントがサンドボックス化されている場合、バイナリは**コンテナ内**にも存在する必要があります。
  `agents.defaults.sandbox.docker.setupCommand`（またはカスタムイメージ）でインストールしてください。
  `setupCommand` はコンテナ作成後に 1 度実行されます。
  パッケージのインストールには、ネットワーク出力、書き込み可能なルート FS、サンドボックス内の root ユーザーも必要です。
  例：`summarize` Skill（`skills/summarize/SKILL.md`）はサンドボックスコンテナ内で実行するために `summarize` CLI が必要です。

インストーラーの例：

```markdown
---
name: gemini
description: Use Gemini CLI for coding assistance and Google search lookups.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "Install Gemini CLI (brew)",
            },
          ],
      },
  }
---
```

注意事項：

- 複数のインストーラーが列挙されている場合、ゲートウェイは**単一の**優先オプションを選択します（利用可能な場合は brew、そうでなければ node）。
- すべてのインストーラーが `download` の場合、OpenClaw は各エントリを一覧表示して利用可能なアーティファクトを確認できるようにします。
- インストーラー仕様には `os: ["darwin"|"linux"|"win32"]` を含めてプラットフォームでオプションをフィルタリングできます。
- Node インストールは `openclaw.json` の `skills.install.nodeManager` に従います（デフォルト：npm；オプション：npm/pnpm/yarn/bun）。
  これは **Skill のインストール**にのみ影響します；ゲートウェイランタイムは引き続き Node である必要があります
  （WhatsApp/Telegram には Bun は推奨されません）。
- Go インストール：`go` が存在せず `brew` が利用可能な場合、ゲートウェイはまず Homebrew 経由で Go をインストールし、可能であれば `GOBIN` を Homebrew の `bin` に設定します。
- Download インストール：`url`（必須）、`archive`（`tar.gz` | `tar.bz2` | `zip`）、`extract`（デフォルト：アーカイブ検出時に自動）、`stripComponents`、`targetDir`（デフォルト：`~/.openclaw/tools/<skillKey>`）。

`metadata.openclaw` が存在しない場合、Skill は常に有効です（設定で無効にされているか、バンドル済み Skills に対して `skills.allowBundled` でブロックされている場合を除く）。

## 設定オーバーライド（`~/.openclaw/openclaw.json`）

バンドル済み/管理済み Skills を切り替えて環境変数値を供給できます：

```json5
{
  skills: {
    entries: {
      "nano-banana-pro": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // またはプレーンテキスト文字列
        env: {
          GEMINI_API_KEY: "GEMINI_KEY_HERE",
        },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

注意：Skill 名にハイフンが含まれる場合は、キーを引用符で囲んでください（JSON5 では引用符付きキーが使用可能）。

設定キーはデフォルトで **Skill 名**に一致します。Skill が `metadata.openclaw.skillKey` を定義している場合は、`skills.entries` 下でそのキーを使用してください。

ルール：

- `enabled: false` は、バンドル済み/インストール済みであっても Skill を無効にします。
- `env`：変数がプロセス内にまだ設定されていない**場合にのみ**注入されます。
- `apiKey`：`metadata.openclaw.primaryEnv` を宣言する Skills の便利な設定です。
  プレーンテキスト文字列または SecretRef オブジェクト（`{ source, provider, id }`）をサポートします。
- `config`：カスタムの Skill ごとのフィールドのオプションバッグ；カスタムキーはここに配置する必要があります。
- `allowBundled`：**バンドル済み** Skills のみのオプション許可リスト。設定すると、リスト内のバンドル済み Skills のみが有効になります（管理済み/ワークスペース Skills は影響を受けません）。

## 環境変数の注入（エージェント実行ごと）

エージェント実行が開始されると、OpenClaw は：

1. Skill メタデータを読み込みます。
2. `skills.entries.<key>.env` または `skills.entries.<key>.apiKey` を `process.env` に適用します。
3. **有効な** Skills でシステムプロンプトを構築します。
4. 実行終了後に元の環境を復元します。

これはエージェント実行に**スコープされており**、グローバルなシェル環境ではありません。

## セッションスナップショット（パフォーマンス）

OpenClaw は**セッション開始時**に有効な Skills のスナップショットを作成し、同一セッションの後続のターンでそのリストを再利用します。Skills または設定への変更は、次の新しいセッションで有効になります。

Skills ウォッチャーが有効になっているとき、または新しい有効なリモートノードが現れたとき（以下を参照）、Skills はセッション途中でも更新されることがあります。これを**ホットリロード**と考えてください：更新されたリストは次のエージェントターンで取得されます。

## リモート macOS ノード（Linux ゲートウェイ）

ゲートウェイが Linux 上で動作しているが、**`system.run` が許可された macOS ノード**が接続されている場合（Exec 承認のセキュリティが `deny` に設定されていない）、必要なバイナリがそのノードに存在するとき、OpenClaw は macOS 専用の Skills を有効として扱うことができます。エージェントはそれらの Skills を `nodes` ツール（通常は `nodes.run`）経由で実行する必要があります。

これはノードがコマンドサポートを報告すること、および `system.run` によるバイナリプローブに依存します。macOS ノードが後でオフラインになった場合、Skills は引き続き表示されます；ノードが再接続するまで呼び出しが失敗する可能性があります。

## Skills ウォッチャー（自動更新）

デフォルトでは、OpenClaw は Skills フォルダーを監視し、`SKILL.md` ファイルが変更されたときに Skills スナップショットを更新します。`skills.load` 下で設定します：

```json5
{
  skills: {
    load: {
      watch: true,
      watchDebounceMs: 250,
    },
  },
}
```

## トークンへの影響（Skills リスト）

Skills が有効な場合、OpenClaw は利用可能な Skills のコンパクトな XML リストをシステムプロンプトに注入します（`pi-coding-agent` の `formatSkillsForPrompt` 経由）。コストは確定的です：

- **基本オーバーヘッド（≥1 Skill の場合のみ）：** 195 文字。
- **Skill ごと：** 97 文字 + XML エスケープされた `<name>`、`<description>`、`<location>` の値の長さ。

式（文字数）：

```
total = 195 + Σ (97 + len(name_escaped) + len(description_escaped) + len(location_escaped))
```

注意事項：

- XML エスケープは `& < > " '` をエンティティ（`&amp;`、`&lt;` など）に展開し、長さが増加します。
- トークン数はモデルのトークナイザーによって異なります。OpenAI スタイルの大まかな見積もりは ~4 文字/トークンで、**97 文字 ≈ 24 トークン** per Skill に実際のフィールド長が加わります。

## 管理済み Skills のライフサイクル

OpenClaw はインストールの一部として（npm パッケージまたは OpenClaw.app）基本的な Skills セットを**バンドル済み Skills** として配布します。`~/.openclaw/skills` はローカルオーバーライド用に存在します（例：バンドル済みのコピーを変更せずに Skill をピン留め/パッチ適用する）。ワークスペース Skills はユーザーが所有し、名前の競合時に両方をオーバーライドします。

## 設定リファレンス

完全な設定スキーマについては [Skills 設定](/tools/skills-config) を参照してください。

## さらに多くの Skills をお探しですか？

[https://clawhub.com](https://clawhub.com) を閲覧してください。

---
