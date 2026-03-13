---
summary: "OpenClaw Gateway CLI（`openclaw gateway`）— Gateway の実行、クエリ、ディスカバリー"
read_when:
  - CLI から Gateway を実行する場合（開発またはサーバー）
  - Gateway の認証、バインドモード、接続性のデバッグ
  - Bonjour による Gateway のディスカバリー（LAN + tailnet）
title: "gateway"
---

# Gateway CLI

Gateway は OpenClaw の WebSocket サーバーです（チャンネル、ノード、セッション、フック）。

このページのサブコマンドは `openclaw gateway …` 配下にあります。

関連ドキュメント:

- [/gateway/bonjour](/gateway/bonjour)
- [/gateway/discovery](/gateway/discovery)
- [/gateway/configuration](/gateway/configuration)

## Gateway の実行

ローカル Gateway プロセスを実行します:

```bash
openclaw gateway
```

フォアグラウンドエイリアス:

```bash
openclaw gateway run
```

備考:

- デフォルトでは、`~/.openclaw/openclaw.json` に `gateway.mode=local` が設定されていない限り、Gateway は起動を拒否します。アドホック/開発用の実行には `--allow-unconfigured` を使用してください。
- 認証なしでループバック以外にバインドすることはブロックされます（安全ガードレール）。
- `SIGUSR1` は認可されている場合にインプロセスリスタートをトリガーします（`commands.restart` はデフォルトで有効。手動リスタートをブロックするには `commands.restart: false` に設定。gateway ツール/config apply/update は引き続き許可されます）。
- `SIGINT`/`SIGTERM` ハンドラーは gateway プロセスを停止しますが、カスタムターミナル状態は復元しません。CLI を TUI やローモード入力でラップしている場合は、終了前にターミナルを復元してください。

### オプション

- `--port <port>`: WebSocket ポート（デフォルトは設定/環境変数から取得。通常 `18789`）。
- `--bind <loopback|lan|tailnet|auto|custom>`: リスナーバインドモード。
- `--auth <token|password>`: 認証モードのオーバーライド。
- `--token <token>`: トークンのオーバーライド（プロセスの `OPENCLAW_GATEWAY_TOKEN` も設定されます）。
- `--password <password>`: パスワードのオーバーライド（プロセスの `OPENCLAW_GATEWAY_PASSWORD` も設定されます）。
- `--tailscale <off|serve|funnel>`: Tailscale 経由で Gateway を公開します。
- `--tailscale-reset-on-exit`: シャットダウン時に Tailscale serve/funnel 設定をリセットします。
- `--allow-unconfigured`: 設定に `gateway.mode=local` がなくても gateway の起動を許可します。
- `--dev`: 存在しない場合、dev 設定 + ワークスペースを作成します（BOOTSTRAP.md をスキップ）。
- `--reset`: dev 設定 + 認証情報 + セッション + ワークスペースをリセットします（`--dev` が必要）。
- `--force`: 起動前に選択したポートの既存リスナーを強制終了します。
- `--verbose`: 詳細ログを出力します。
- `--claude-cli-logs`: コンソールに claude-cli ログのみを表示します（stdout/stderr も有効化）。
- `--ws-log <auto|full|compact>`: WebSocket ログスタイル（デフォルト `auto`）。
- `--compact`: `--ws-log compact` のエイリアス。
- `--raw-stream`: 生のモデルストリームイベントを jsonl にログ出力します。
- `--raw-stream-path <path>`: 生のストリーム jsonl パス。

## 実行中の Gateway へのクエリ

すべてのクエリコマンドは WebSocket RPC を使用します。

出力モード:

- デフォルト: 人間が読みやすい形式（TTY ではカラー表示）。
- `--json`: マシンリーダブルな JSON（スタイリング/スピナーなし）。
- `--no-color`（または `NO_COLOR=1`）: 人間向けレイアウトを維持しつつ ANSI を無効化。

共通オプション（対応する場合）:

- `--url <url>`: Gateway WebSocket URL。
- `--token <token>`: Gateway トークン。
- `--password <password>`: Gateway パスワード。
- `--timeout <ms>`: タイムアウト/バジェット（コマンドにより異なる）。
- `--expect-final`: 「final」レスポンスを待機します（エージェント呼び出し）。

注意: `--url` を指定すると、CLI は設定や環境の認証情報にフォールバックしません。
`--token` または `--password` を明示的に指定してください。明示的な認証情報がない場合はエラーとなります。

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

### `gateway status`

`gateway status` は Gateway サービス（launchd/systemd/schtasks）とオプションの RPC プローブを表示します。

```bash
openclaw gateway status
openclaw gateway status --json
```

オプション:

- `--url <url>`: プローブ URL をオーバーライドします。
- `--token <token>`: プローブ用のトークン認証。
- `--password <password>`: プローブ用のパスワード認証。
- `--timeout <ms>`: プローブタイムアウト（デフォルト `10000`）。
- `--no-probe`: RPC プローブをスキップします（サービスのみの表示）。
- `--deep`: システムレベルのサービスもスキャンします。

備考:

- `gateway status` は可能な場合、設定済み認証の SecretRef をプローブ認証用に解決します。
- このコマンドパスで必要な認証 SecretRef が未解決の場合、プローブ認証が失敗する可能性があります。`--token`/`--password` を明示的に指定するか、先にシークレットソースを解決してください。

### `gateway probe`

`gateway probe` は「すべてをデバッグする」コマンドです。常に以下をプローブします:

- 設定済みのリモート gateway（設定されている場合）、および
- リモートが設定されていても **localhost（ループバック）**。

複数の gateway が到達可能な場合、すべて表示されます。分離プロファイル/ポートを使用する場合（例: レスキューボット）、複数の gateway がサポートされますが、ほとんどのインストールでは単一の gateway を実行します。

```bash
openclaw gateway probe
openclaw gateway probe --json
```

#### SSH 経由のリモート（Mac アプリと同等）

macOS アプリの「Remote over SSH」モードはローカルポートフォワードを使用するため、リモート gateway（ループバックのみにバインドされている場合がある）が `ws://127.0.0.1:<port>` で到達可能になります。

CLI での同等の操作:

```bash
openclaw gateway probe --ssh user@gateway-host
```

オプション:

- `--ssh <target>`: `user@host` または `user@host:port`（ポートのデフォルトは `22`）。
- `--ssh-identity <path>`: アイデンティティファイル。
- `--ssh-auto`: 最初に検出された gateway ホストを SSH ターゲットとして選択します（LAN/WAB のみ）。

設定（オプション、デフォルトとして使用）:

- `gateway.remote.sshTarget`
- `gateway.remote.sshIdentity`

### `gateway call <method>`

低レベル RPC ヘルパー。

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"sinceMs": 60000}'
```

## Gateway サービスの管理

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

備考:

- `gateway install` は `--port`、`--runtime`、`--token`、`--force`、`--json` をサポートします。
- トークン認証でトークンが必要な場合、`gateway.auth.token` が SecretRef で管理されていると、`gateway install` は SecretRef が解決可能であることを検証しますが、解決されたトークンをサービス環境メタデータに永続化しません。
- トークン認証でトークンが必要で、設定済みトークンの SecretRef が未解決の場合、フォールバックのプレーンテキストを永続化する代わりにインストールは失敗します。
- 推論された認証モードでは、シェルのみの `OPENCLAW_GATEWAY_PASSWORD`/`CLAWDBOT_GATEWAY_PASSWORD` はインストールのトークン要件を緩和しません。マネージドサービスをインストールする場合は、永続的な設定（`gateway.auth.password` または設定 `env`）を使用してください。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定され、`gateway.auth.mode` が未設定の場合、モードが明示的に設定されるまでインストールはブロックされます。
- ライフサイクルコマンドはスクリプティング用に `--json` を受け付けます。

## Gateway のディスカバリー（Bonjour）

`gateway discover` は Gateway ビーコン（`_openclaw-gw._tcp`）をスキャンします。

- マルチキャスト DNS-SD: `local.`
- ユニキャスト DNS-SD（ワイドエリア Bonjour）: ドメインを選択し（例: `openclaw.internal.`）、スプリット DNS + DNS サーバーを設定します。[/gateway/bonjour](/gateway/bonjour) を参照

Bonjour ディスカバリーが有効（デフォルト）な gateway のみがビーコンをアドバタイズします。

ワイドエリアディスカバリーレコードに含まれる情報（TXT）:

- `role`（gateway ロールのヒント）
- `transport`（トランスポートのヒント。例: `gateway`）
- `gatewayPort`（WebSocket ポート。通常 `18789`）
- `sshPort`（SSH ポート。存在しない場合のデフォルトは `22`）
- `tailnetDns`（MagicDNS ホスト名。利用可能な場合）
- `gatewayTls` / `gatewayTlsSha256`（TLS 有効 + 証明書フィンガープリント）
- `cliPath`（リモートインストール用のオプションヒント）

### `gateway discover`

```bash
openclaw gateway discover
```

オプション:

- `--timeout <ms>`: コマンドごとのタイムアウト（browse/resolve）。デフォルト `2000`。
- `--json`: マシンリーダブルな出力（スタイリング/スピナーも無効化）。

例:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```
