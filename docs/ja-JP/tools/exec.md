---
summary: "exec ツールの使い方、stdin モード、TTY サポート"
read_when:
  - exec ツールを使用または変更するとき
  - stdin または TTY の動作をデバッグするとき
title: "Exec ツール"
---

# Exec ツール

ワークスペースでシェルコマンドを実行します。`process` を介したフォアグラウンド + バックグラウンド実行をサポートします。
`process` が無効化されている場合、`exec` は同期的に実行され、`yieldMs`/`background` を無視します。
バックグラウンドセッションはエージェントごとにスコープされます；`process` は同じエージェントのセッションのみを参照できます。

## パラメーター

- `command`（必須）
- `workdir`（デフォルトは cwd）
- `env`（キー/値の上書き）
- `yieldMs`（デフォルト 10000）：遅延後に自動バックグラウンド
- `background`（ブール値）：即座にバックグラウンド
- `timeout`（秒、デフォルト 1800）：期限切れでプロセスを終了
- `pty`（ブール値）：利用可能な場合に擬似端末で実行（TTY 専用 CLI、コーディングエージェント、ターミナル UI）
- `host`（`sandbox | gateway | node`）：実行場所
- `security`（`deny | allowlist | full`）：`gateway`/`node` の強制モード
- `ask`（`off | on-miss | always`）：`gateway`/`node` の承認プロンプト
- `node`（文字列）：`host=node` のノード id/名前
- `elevated`（ブール値）：昇格モードのリクエスト（gateway ホスト）；`elevated` が `full` に解決された場合にのみ `security=full` が強制される

注意事項：

- `host` のデフォルトは `sandbox`。
- サンドボックスがオフの場合、`elevated` は無視される（exec はすでにホスト上で実行中）。
- `gateway`/`node` の承認は `~/.openclaw/exec-approvals.json` によって制御される。
- `node` にはペアリングされたノード（コンパニオンアプリまたはヘッドレスノードホスト）が必要。
- 複数のノードが利用可能な場合、`exec.node` または `tools.exec.node` を設定して選択する。
- 非 Windows ホストでは、`SHELL` が設定されている場合 exec はそれを使用します；`SHELL` が `fish` の場合、fish 非互換スクリプトを避けるために `PATH` から `bash`（または `sh`）を優先し、両方存在しない場合は `SHELL` にフォールバックします。
- Windows ホストでは、exec は PowerShell 7（`pwsh`）の検出を優先し（Program Files、ProgramW6432、次に PATH）、見つからない場合は Windows PowerShell 5.1 にフォールバックします。
- ホスト実行（`gateway`/`node`）は、バイナリハイジャックや注入コードを防ぐために `env.PATH` とローダーの上書き（`LD_*`/`DYLD_*`）を拒否します。
- OpenClaw はスポーンされたコマンド環境（PTY とサンドボックス実行を含む）に `OPENCLAW_SHELL=exec` を設定するため、シェル/プロファイルルールが exec ツールコンテキストを検出できます。
- 重要：サンドボックスは**デフォルトでオフ**です。サンドボックスがオフで `host=sandbox` が明示的に設定/要求された場合、exec はサイレントに gateway ホスト上で実行する代わりに、クローズドフェイルします。サンドボックスを有効にするか、承認付きで `host=gateway` を使用してください。
- スクリプトの事前チェック（一般的な Python/Node シェル構文エラー用）は、有効な `workdir` 境界内のファイルのみを検査します。スクリプトパスが `workdir` の外に解決される場合、そのファイルの事前チェックはスキップされます。

## 設定

- `tools.exec.notifyOnExit`（デフォルト：true）：true の場合、バックグラウンド exec セッションは終了時にシステムイベントをキューに入れ、ハートビートをリクエストします。
- `tools.exec.approvalRunningNoticeMs`（デフォルト：10000）：承認ゲートされた exec がこれより長く実行される場合、単一の「実行中」通知を発します（0 で無効）。
- `tools.exec.host`（デフォルト：`sandbox`）
- `tools.exec.security`（デフォルト：sandbox では `deny`、未設定の場合 gateway + node では `allowlist`）
- `tools.exec.ask`（デフォルト：`on-miss`）
- `tools.exec.node`（デフォルト：未設定）
- `tools.exec.pathPrepend`：exec 実行時に `PATH` の先頭に追加するディレクトリのリスト（gateway + sandbox のみ）。
- `tools.exec.safeBins`：明示的な許可リストエントリなしで実行できる stdin 専用の安全なバイナリ。動作の詳細については、[Safe bins](/tools/exec-approvals#safe-bins-stdin-only) を参照。
- `tools.exec.safeBinTrustedDirs`：`safeBins` パスチェックのために信頼される追加の明示的ディレクトリ。`PATH` エントリは自動的に信頼されることはありません。組み込みのデフォルトは `/bin` と `/usr/bin` です。
- `tools.exec.safeBinProfiles`：安全なバイナリごとのオプションのカスタム argv ポリシー（`minPositional`、`maxPositional`、`allowedValueFlags`、`deniedFlags`）。

例：

```json5
{
  tools: {
    exec: {
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
  },
}
```

### PATH の処理

- `host=gateway`：ログインシェルの `PATH` を exec 環境にマージします。ホスト実行では `env.PATH` の上書きは拒否されます。デーモン自体は最小の `PATH` で実行されます：
  - macOS：`/opt/homebrew/bin`、`/usr/local/bin`、`/usr/bin`、`/bin`
  - Linux：`/usr/local/bin`、`/usr/bin`、`/bin`
- `host=sandbox`：コンテナ内で `sh -lc`（ログインシェル）を実行するため、`/etc/profile` が `PATH` をリセットする場合があります。OpenClaw はプロファイルのソーシング後に内部環境変数を介して `env.PATH` を先頭に追加します（シェル補間なし）；`tools.exec.pathPrepend` もここに適用されます。
- `host=node`：渡した非ブロックの env 上書きのみがノードに送信されます。ホスト実行では `env.PATH` の上書きは拒否され、ノードホストによって無視されます。ノードに追加の PATH エントリが必要な場合は、ノードホストサービス環境（systemd/launchd）を設定するか、標準の場所にツールをインストールしてください。

エージェントごとのノードバインディング（config でエージェントリストのインデックスを使用）：

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

Control UI：Nodes タブには同じ設定のための小さな「Exec ノードバインディング」パネルがあります。

## セッション上書き（`/exec`）

`/exec` を使用して `host`、`security`、`ask`、`node` の**セッションごとの**デフォルトを設定します。
引数なしで `/exec` を送信すると現在の値が表示されます。

例：

```
/exec host=gateway security=allowlist ask=on-miss node=mac-1
```

## 認可モデル

`/exec` は**認可された送信者**のみが利用できます（チャンネル許可リスト/ペアリングと `commands.useAccessGroups`）。
**セッション状態のみ**を更新し、設定を書き込みません。exec を完全に無効にするには、ツールポリシーを介して拒否します（`tools.deny: ["exec"]` またはエージェントごと）。`security=full` と `ask=off` を明示的に設定しない限り、ホスト承認は引き続き適用されます。

## Exec 承認（コンパニオンアプリ / ノードホスト）

サンドボックス化されたエージェントは、`exec` が gateway またはノードホスト上で実行される前にリクエストごとの承認を要求できます。
ポリシー、許可リスト、UI フローについては [Exec 承認](/tools/exec-approvals) を参照してください。

承認が必要な場合、exec ツールは即座に `status: "approval-pending"` と承認 id を返します。承認（または拒否/タイムアウト）されると、Gateway はシステムイベント（`Exec finished` / `Exec denied`）を発します。`tools.exec.approvalRunningNoticeMs` 後もコマンドが実行中の場合、単一の `Exec running` 通知が発されます。

## 許可リスト + Safe bins

手動の許可リストの強制は**解決されたバイナリパスのみ**に一致します（ベース名の一致なし）。`security=allowlist` の場合、すべてのパイプラインセグメントが許可リストに入っているか Safe bin である場合にのみ、シェルコマンドが自動的に許可されます。許可リストモードでは、チェーン（`;`、`&&`、`||`）とリダイレクトは、すべてのトップレベルセグメントが許可リスト（Safe bins を含む）を満たさない限り拒否されます。リダイレクトはサポートされていません。

`autoAllowSkills` は exec 承認における別の便利なパスです。手動パス許可リストエントリとは異なります。厳密な明示的信頼のためには、`autoAllowSkills` を無効に保ってください。

2つのコントロールをそれぞれの目的に使用してください：

- `tools.exec.safeBins`：小さな stdin 専用ストリームフィルター。
- `tools.exec.safeBinTrustedDirs`：safe-bin 実行ファイルパスのための追加の明示的な信頼ディレクトリ。
- `tools.exec.safeBinProfiles`：カスタム safe bins のための明示的な argv ポリシー。
- 許可リスト：実行ファイルパスの明示的な信頼。

`safeBins` を汎用の許可リストとして扱わず、インタープリター/ランタイムバイナリ（例：`python3`、`node`、`ruby`、`bash`）を追加しないでください。それらが必要な場合は、明示的な許可リストエントリを使用して承認プロンプトを有効に保ってください。
`openclaw security audit` はインタープリター/ランタイムの `safeBins` エントリに明示的なプロファイルがない場合に警告し、`openclaw doctor --fix` は不足しているカスタム `safeBinProfiles` エントリをスキャフォールドできます。

完全なポリシーの詳細と例については、[Exec 承認](/tools/exec-approvals#safe-bins-stdin-only) と [Safe bins vs 許可リスト](/tools/exec-approvals#safe-bins-versus-allowlist) を参照してください。

## 例

フォアグラウンド：

```json
{ "tool": "exec", "command": "ls -la" }
```

バックグラウンド + ポーリング：

```json
{"tool":"exec","command":"npm run build","yieldMs":1000}
{"tool":"process","action":"poll","sessionId":"<id>"}
```

キー送信（tmux スタイル）：

```json
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Enter"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["C-c"]}
{"tool":"process","action":"send-keys","sessionId":"<id>","keys":["Up","Up","Enter"]}
```

送信（CR のみ送信）：

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

貼り付け（デフォルトはブラケット付き）：

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## apply_patch（実験的）

`apply_patch` は構造化されたマルチファイル編集のための `exec` のサブツールです。
明示的に有効にする必要があります：

```json5
{
  tools: {
    exec: {
      applyPatch: { enabled: true, workspaceOnly: true, allowModels: ["gpt-5.2"] },
    },
  },
}
```

注意事項：

- OpenAI/OpenAI Codex モデルのみ利用可能。
- ツールポリシーは引き続き適用されます；`allow: ["exec"]` は暗黙的に `apply_patch` を許可します。
- 設定は `tools.exec.applyPatch` の下にあります。
- `tools.exec.applyPatch.workspaceOnly` のデフォルトは `true`（ワークスペース内に制限）。`apply_patch` がワークスペースディレクトリの外に書き込み/削除することを意図的に許可する場合のみ `false` に設定してください。
