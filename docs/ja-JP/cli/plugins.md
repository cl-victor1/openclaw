---
summary: "`openclaw plugins` の CLI リファレンス（list、install、uninstall、enable/disable、doctor）"
read_when:
  - インプロセスの Gateway プラグインをインストールまたは管理したいとき
  - プラグインの読み込み失敗をデバッグしたいとき
title: "plugins"
---

# `openclaw plugins`

Gateway プラグイン/拡張機能の管理（インプロセスで読み込み）。

関連ドキュメント:

- プラグインシステム: [プラグイン](/tools/plugin)
- プラグインマニフェスト + スキーマ: [プラグインマニフェスト](/plugins/manifest)
- セキュリティ強化: [セキュリティ](/gateway/security)

## コマンド

```bash
openclaw plugins list
openclaw plugins info <id>
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id>
openclaw plugins doctor
openclaw plugins update <id>
openclaw plugins update --all
```

バンドルプラグインは OpenClaw に同梱されていますが、無効な状態で起動します。`plugins enable` で
有効化してください。

すべてのプラグインは `openclaw.plugin.json` ファイルにインライン JSON Schema
（`configSchema`、空でも可）を含める必要があります。マニフェストやスキーマが欠落/無効な場合、
プラグインの読み込みが防止され、設定の検証に失敗します。

### インストール

```bash
openclaw plugins install <path-or-spec>
openclaw plugins install <npm-spec> --pin
```

セキュリティに関する注意: プラグインのインストールはコードの実行と同等に扱ってください。固定バージョンを推奨します。

npm スペックは**レジストリのみ**（パッケージ名 + オプションの**正確なバージョン**または
**dist-tag**）です。Git/URL/ファイルスペックと semver 範囲は拒否されます。依存関係のインストールは安全のため `--ignore-scripts` で実行されます。

ベアスペックと `@latest` は安定トラックを使用します。npm がいずれかをプレリリースに解決した場合、
OpenClaw は停止し、`@beta`/`@rc` などのプレリリースタグまたは `@1.2.3-beta.4` などの正確なプレリリースバージョンでの明示的なオプトインを求めます。

ベアインストールスペックがバンドルプラグイン ID（例: `diffs`）と一致する場合、OpenClaw は
バンドルプラグインを直接インストールします。同じ名前の npm パッケージをインストールするには、
明示的なスコープ付きスペック（例: `@scope/diffs`）を使用してください。

サポートされるアーカイブ: `.zip`、`.tgz`、`.tar.gz`、`.tar`。

ローカルディレクトリのコピーを避けるには `--link` を使用してください（`plugins.load.paths` に追加されます）:

```bash
openclaw plugins install -l ./my-plugin
```

npm インストールで `--pin` を使用すると、解決された正確なスペック（`name@version`）が
`plugins.installs` に保存され、デフォルト動作は非固定のままになります。

### アンインストール

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
```

`uninstall` は `plugins.entries`、`plugins.installs`、プラグイン許可リスト、および該当する場合はリンクされた `plugins.load.paths` エントリからプラグインレコードを削除します。
アクティブなメモリプラグインの場合、メモリスロットは `memory-core` にリセットされます。

デフォルトでは、アンインストールはアクティブなステートディレクトリの拡張機能ルート（`$OPENCLAW_STATE_DIR/extensions/<id>`）下のプラグインインストールディレクトリも削除します。ファイルをディスク上に保持するには `--keep-files` を使用してください。

`--keep-config` は `--keep-files` の非推奨エイリアスとしてサポートされています。

### アップデート

```bash
openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update <id> --dry-run
```

アップデートは npm からインストールされたプラグイン（`plugins.installs` で追跡）にのみ適用されます。

保存された整合性ハッシュが存在し、取得したアーティファクトのハッシュが変更された場合、
OpenClaw は警告を表示し、続行前に確認を求めます。CI/非対話型実行では
グローバル `--yes` を使用してプロンプトをバイパスしてください。
