---
summary: "モデルCLI：一覧表示、設定、エイリアス、フォールバック、スキャン、ステータス"
read_when:
  - モデルCLI（models list/set/scan/aliases/fallbacks）の追加または変更
  - モデルフォールバック動作または選択UXの変更
  - モデルスキャンプローブ（tools/images）の更新
title: "モデルCLI"
---

# モデルCLI

認証プロファイルのローテーション、クールダウン、フォールバックとの連携については、[/concepts/model-failover](/concepts/model-failover)を参照してください。
プロバイダーの概要と例：[/concepts/model-providers](/concepts/model-providers)。

## モデル選択の仕組み

OpenClawは以下の順でモデルを選択します：

1. **プライマリ**モデル（`agents.defaults.model.primary` または `agents.defaults.model`）。
2. `agents.defaults.model.fallbacks` の**フォールバック**（順番通り）。
3. **プロバイダー認証フェイルオーバー**は次のモデルに移行する前にプロバイダー内で発生します。

関連：

- `agents.defaults.models` はOpenClawが使用できるモデルの許可リスト/カタログ（エイリアスを含む）。
- `agents.defaults.imageModel` はプライマリモデルが画像を受け付けられない**場合のみ**使用されます。
- エージェントごとのデフォルトは `agents.list[].model` とバインディング経由で `agents.defaults.model` を上書きできます（[/concepts/multi-agent](/concepts/multi-agent)を参照）。

## モデルポリシーの基本

- プライマリには利用可能な最強の最新世代モデルを設定します。
- コスト/レイテンシを重視するタスクや低リスクなチャットにはフォールバックを使用します。
- ツール対応エージェントや信頼できない入力には、古い/弱いモデル層を避けます。

## セットアップウィザード（推奨）

設定を手動編集したくない場合は、オンボーディングウィザードを実行します：

```bash
openclaw onboard
```

**OpenAI Code（Codex）サブスクリプション**（OAuth）や**Anthropic**（APIキーまたは `claude setup-token`）など、一般的なプロバイダーのモデルと認証設定ができます。

## 設定キー（概要）

- `agents.defaults.model.primary` と `agents.defaults.model.fallbacks`
- `agents.defaults.imageModel.primary` と `agents.defaults.imageModel.fallbacks`
- `agents.defaults.models`（許可リスト + エイリアス + プロバイダーパラメータ）
- `models.providers`（`models.json` に書き込まれるカスタムプロバイダー）

モデルrefは小文字に正規化されます。`z.ai/*` などのプロバイダーエイリアスは `zai/*` に正規化されます。

プロバイダー設定例（OpenCode Zenを含む）は[/gateway/configuration](/gateway/configuration#opencode-zen-multi-model-proxy)にあります。

## 「モデルが許可されていません」（返答が止まる理由）

`agents.defaults.models` が設定されている場合、`/model` とセッションオーバーライドの**許可リスト**になります。ユーザーがその許可リストにないモデルを選択すると、OpenClawは次のメッセージを返します：

```
Model "provider/model" is not allowed. Use /model to list available models.
```

これは通常の返答が生成される**前**に発生するため、「返答しなかった」ように見えることがあります。解決策：

- モデルを `agents.defaults.models` に追加する、または
- 許可リストをクリアする（`agents.defaults.models` を削除）、または
- `/model list` からモデルを選ぶ。

許可リスト設定の例：

```json5
{
  agent: {
    model: { primary: "anthropic/claude-sonnet-4-5" },
    models: {
      "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
      "anthropic/claude-opus-4-6": { alias: "Opus" },
    },
  },
}
```

## チャットでのモデル切り替え（`/model`）

再起動せずに現在のセッションのモデルを切り替えられます：

```
/model
/model list
/model 3
/model openai/gpt-5.2
/model status
```

注意：

- `/model`（および `/model list`）はコンパクトな番号付きピッカーです（モデルファミリー + 利用可能なプロバイダー）。
- Discordでは、`/model` と `/models` がプロバイダーとモデルのドロップダウン + 送信ステップを持つインタラクティブなピッカーを開きます。
- `/model <#>` でピッカーから選択します。
- `/model status` は詳細ビューです（認証候補、設定されている場合はプロバイダーエンドポイントの `baseUrl` + `api` モード）。
- モデルrefは**最初の** `/` で分割して解析されます。`/model <ref>` を入力する際は `provider/model` 形式を使用します。
- モデルID自体に `/` が含まれる場合（OpenRouterスタイル）、プロバイダープレフィックスを含める必要があります（例：`/model openrouter/moonshotai/kimi-k2`）。
- プロバイダーを省略すると、OpenClawはその入力をエイリアスまたは**デフォルトプロバイダー**のモデルとして扱います（モデルIDに `/` がない場合のみ有効）。

コマンドの完全な動作/設定：[スラッシュコマンド](/tools/slash-commands)。

## CLIコマンド

```bash
openclaw models list
openclaw models status
openclaw models set <provider/model>
openclaw models set-image <provider/model>

openclaw models aliases list
openclaw models aliases add <alias> <provider/model>
openclaw models aliases remove <alias>

openclaw models fallbacks list
openclaw models fallbacks add <provider/model>
openclaw models fallbacks remove <provider/model>
openclaw models fallbacks clear

openclaw models image-fallbacks list
openclaw models image-fallbacks add <provider/model>
openclaw models image-fallbacks remove <provider/model>
openclaw models image-fallbacks clear
```

`openclaw models`（サブコマンドなし）は `models status` のショートカットです。

### `models list`

デフォルトでは設定済みモデルを表示します。便利なフラグ：

- `--all`: 全カタログ
- `--local`: ローカルプロバイダーのみ
- `--provider <name>`: プロバイダーでフィルタ
- `--plain`: 1行1モデル
- `--json`: 機械可読出力

### `models status`

解決済みのプライマリモデル、フォールバック、画像モデル、設定済みプロバイダーの認証概要を表示します。認証ストアで見つかったOAuthプロファイルの有効期限ステータスも表示します（デフォルトで24時間以内に警告）。`--plain` は解決済みのプライマリモデルのみを表示します。
OAuthステータスは常に表示されます（`--json` 出力にも含まれます）。設定済みプロバイダーに認証情報がない場合、`models status` は**認証欠如**セクションを表示します。
JSONには `auth.oauth`（警告ウィンドウ + プロファイル）と `auth.providers`（プロバイダーごとの有効な認証）が含まれます。
自動化には `--check` を使用します（欠如/期限切れで終了コード `1`、期限切れ間近で `2`）。

認証の選択はプロバイダー/アカウントに依存します。常時稼働のGatewayホストでは、APIキーが最も予測可能です；サブスクリプショントークンフローもサポートされます。

例（Anthropic setup-token）：

```bash
claude setup-token
openclaw models status
```

## スキャン（OpenRouterの無料モデル）

`openclaw models scan` はOpenRouterの**無料モデルカタログ**を検査し、オプションでツールと画像サポートのモデルをプローブできます。

主なフラグ：

- `--no-probe`: ライブプローブをスキップ（メタデータのみ）
- `--min-params <b>`: 最小パラメータサイズ（十億）
- `--max-age-days <days>`: 古いモデルをスキップ
- `--provider <name>`: プロバイダープレフィックスフィルタ
- `--max-candidates <n>`: フォールバックリストのサイズ
- `--set-default`: 最初の選択を `agents.defaults.model.primary` に設定
- `--set-image`: 最初の画像選択を `agents.defaults.imageModel.primary` に設定

プローブにはOpenRouter APIキーが必要です（認証プロファイルまたは `OPENROUTER_API_KEY` から）。キーがない場合は `--no-probe` を使用して候補のみを一覧表示します。

スキャン結果は以下でランク付けされます：

1. 画像サポート
2. ツールレイテンシ
3. コンテキストサイズ
4. パラメータ数

入力

- OpenRouterの `/models` リスト（`:free` でフィルタ）
- 認証プロファイルまたは `OPENROUTER_API_KEY` からのOpenRouter APIキーが必要（[/environment](/help/environment)を参照）
- オプションのフィルタ：`--max-age-days`、`--min-params`、`--provider`、`--max-candidates`
- プローブ制御：`--timeout`、`--concurrency`

TTYで実行した場合、フォールバックをインタラクティブに選択できます。非インタラクティブモードでは `--yes` を渡してデフォルトを受け入れます。

## モデルレジストリ（`models.json`）

`models.providers` のカスタムプロバイダーはエージェントディレクトリ下の `models.json` に書き込まれます（デフォルト `~/.openclaw/agents/<agentId>/models.json`）。`models.mode` が `replace` に設定されていない限り、このファイルはデフォルトでマージされます。

一致するプロバイダーIDのマージモード優先順位：

- エージェントの `models.json` に既にある空でない `baseUrl` が優先されます。
- エージェントの `models.json` の空でない `apiKey` は、そのプロバイダーが現在のconfig/認証プロファイルコンテキストでSecretRef管理されていない場合のみ優先されます。
- SecretRef管理のプロバイダー `apiKey` 値は、ソースマーカー（env refの場合は `ENV_VAR_NAME`、file/exec refの場合は `secretref-managed`）からリフレッシュされ、解決済みのシークレットは永続化されません。
- 空または欠落したエージェントの `apiKey`/`baseUrl` はconfig `models.providers` にフォールバックします。
- その他のプロバイダーフィールドはconfigと正規化されたカタログデータからリフレッシュされます。

このマーカーベースの永続化は、`openclaw agent` などのコマンド駆動パスを含め、OpenClawが `models.json` を再生成するたびに適用されます。
