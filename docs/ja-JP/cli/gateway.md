---
summary: "OpenClaw Gateway CLI（`openclaw gateway`）— ゲートウェイの実行・クエリ・検出"
read_when:
  - CLIからゲートウェイを実行する場合（開発またはサーバー）
  - ゲートウェイの認証、バインドモード、接続性をデバッグする場合
  - Bonjourを使用してゲートウェイを検出する場合（LANおよびtailnet）
title: "gateway"
---

# Gateway CLI

ゲートウェイは OpenClaw の WebSocket サーバーです（チャンネル、ノード、セッション、フック）。

このページのサブコマンドは `openclaw gateway …` 以下にあります。

関連ドキュメント：

- [/gateway/bonjour](/gateway/bonjour)
- [/gateway/discovery](/gateway/discovery)
- [/gateway/configuration](/gateway/configuration)

## ゲートウェイを実行する

ローカルゲートウェイプロセスを実行します：

```bash
openclaw gateway
```

フォアグラウンドのエイリアス：

```bash
openclaw gateway run
```

注意事項：

- デフォルトでは、`~/.openclaw/openclaw.json` に `gateway.mode=local` が設定されていない限り、ゲートウェイの起動は拒否されます。アドホック/開発用実行には `--allow-unconfigured` を使用してください。
- 認証なしでループバック以外へのバインドはブロックされます（安全ガードレール）。
- `SIGUSR1` は、`commands.restart` が有効な場合（デフォルトで有効；手動再起動をブロックするには `commands.restart: false` を設定、ただしゲートウェイツール/設定適用/更新は引き続き許可）、プロセス内の再起動をトリガーします。
- `SIGINT`/`SIGTERM` ハンドラーはゲートウェイプロセスを停止しますが、カスタムターミナル状態は復元されません。TUIや生入力モードでCLIをラップする場合は、終了前にターミナルを復元してください。

### オプション

- `--port <port>`：WebSocketポート（デフォルトは設定/環境変数から取得；通常は `18789`）。
- `--bind <loopback|lan|tailnet|auto|custom>`：リスナーバインドモード。
- `--auth <token|password>`：認証モードのオーバーライド。
- `--token <token>`：トークンのオーバーライド（プロセスの `OPENCLAW_GATEWAY_TOKEN` も設定）。
- `--password <password>`：パスワードのオーバーライド。警告：インラインパスワードはローカルプロセスリストに表示される可能性があります。
- `--password-file <path>`：ファイルからゲートウェイパスワードを読み取ります。
- `--tailscale <off|serve|funnel>`：Tailscale 経由でゲートウェイを公開します。
- `--tailscale-reset-on-exit`：シャットダウン時に Tailscale serve/funnel の設定をリセットします。
- `--allow-unconfigured`：設定に `gateway.mode=local` がなくてもゲートウェイの起動を許可します。
- `--dev`：devの設定とワークスペースが存在しない場合に作成します（BOOTSTRAP.md をスキップ）。
- `--reset`：devの設定、認証情報、セッション、ワークスペースをリセットします（`--dev` が必要）。
- `--force`：起動前に選択したポート上の既存リスナーを強制終了します。
- `--verbose`：詳細ログを表示します。
- `--claude-cli-logs`：コンソールに claude-cli のログのみを表示します（stdout/stderr も有効化）。
- `--ws-log <auto|full|compact>`：WebSocket ログスタイル（デフォルト `auto`）。
- `--compact`：`--ws-log compact` のエイリアス。
- `--raw-stream`：生のモデルストリームイベントを jsonl に記録します。
- `--raw-stream-path <path>`：生ストリームの jsonl パス。

## 実行中のゲートウェイへのクエリ

すべてのクエリコマンドは WebSocket RPC を使用します。

出力モード：

- デフォルト：人間が読みやすい形式（TTY ではカラー表示）。
- `--json`：機械読み取り可能な JSON（スタイリング/スピナーなし）。
- `--no-color`（または `NO_COLOR=1`）：人間向けレイアウトを保ちつつ ANSI を無効化。

共通オプション（サポートされている箇所）：

- `--url <url>`：ゲートウェイの WebSocket URL。
- `--token <token>`：ゲートウェイのトークン。
- `--password <password>`：ゲートウェイのパスワード。
- `--timeout <ms>`：タイムアウト/バジェット（コマンドによって異なります）。
- `--expect-final`：「final」レスポンスを待ちます（エージェント呼び出し）。

注意：`--url` を設定すると、CLI は設定や環境変数の認証情報にフォールバックしません。
`--token` または `--password` を明示的に指定してください。認証情報が不足している場合はエラーとなります。

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
```

### `gateway status`

`gateway status` は、ゲートウェイサービス（launchd/systemd/schtasks）とオプションの RPC プローブを表示します。

```bash
openclaw gateway status
openclaw gateway status --json
```

オプション：

- `--url <url>`：プローブ URL のオーバーライド。
- `--token <token>`：プローブのトークン認証。
- `--password <password>`：プローブのパスワード認証。
- `--timeout <ms>`：プローブタイムアウト（デフォルト `10000`）。
- `--no-probe`：RPC プローブをスキップします（サービスのみを表示）。
- `--deep`：システムレベルのサービスもスキャンします。

注意事項：

- `gateway status` は、可能な場合にプローブ認証のために設定済み SecretRef を解決します。
- このコマンドパスで必要な SecretRef が未解決の場合、プローブ認証が失敗することがあります。`--token`/`--password` を明示的に渡すか、まずシークレットソースを解決してください。
- Linux systemd インストールでは、サービス認証ドリフトチェックがユニットの `Environment=` と `EnvironmentFile=` の値を読み取ります（`%h`、クォートされたパス、複数ファイル、オプションの `-` ファイルを含む）。

### `gateway probe`

`gateway probe` は「すべてをデバッグ」するコマンドです。常に以下をプローブします：

- 設定されているリモートゲートウェイ（設定している場合）、および
- リモートが設定されている場合でも **localhost（ループバック）**。

複数のゲートウェイに到達可能な場合、すべてを表示します。隔離されたプロファイル/ポートを使用する場合（例：レスキューボット）に複数のゲートウェイがサポートされますが、ほとんどのインストールでは単一のゲートウェイを実行します。

```bash
openclaw gateway probe
openclaw gateway probe --json
```

#### SSH 経由のリモート（Mac アプリとの互換性）

macOS アプリの「Remote over SSH」モードは、ローカルポートフォワードを使用して、リモートゲートウェイ（ループバックのみにバインドされている可能性がある）を `ws://127.0.0.1:<port>` で到達可能にします。

CLIでの同等操作：

```bash
openclaw gateway probe --ssh user@gateway-host
```

オプション：

- `--ssh <target>`：`user@host` または `user@host:port`（ポートのデフォルトは `22`）。
- `--ssh-identity <path>`：識別ファイル。
- `--ssh-auto`：最初に検出されたゲートウェイホストを SSH ターゲットとして選択します（LAN/WAB のみ）。

設定（オプション、デフォルト値として使用）：

- `gateway.remote.sshTarget`
- `gateway.remote.sshIdentity`

### `gateway call <method>`

低レベルの RPC ヘルパー。

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"sinceMs": 60000}'
```

## ゲートウェイサービスの管理

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

注意事項：

- `gateway install` は `--port`、`--runtime`、`--token`、`--force`、`--json` をサポートします。
- トークン認証がトークンを必要とし、`gateway.auth.token` が SecretRef で管理されている場合、`gateway install` は SecretRef が解決可能であることを検証しますが、解決されたトークンをサービス環境メタデータに永続化しません。
- トークン認証がトークンを必要とし、設定されているトークン SecretRef が未解決の場合、インストールはフォールバックのプレーンテキストを永続化する代わりにクローズドで失敗します。
- `gateway run` でのパスワード認証には、インライン `--password` ではなく `OPENCLAW_GATEWAY_PASSWORD`、`--password-file`、または SecretRef-backed の `gateway.auth.password` を推奨します。
- 推論認証モードでは、シェルのみの `OPENCLAW_GATEWAY_PASSWORD`/`CLAWDBOT_GATEWAY_PASSWORD` は、インストールのトークン要件を緩和しません。管理サービスをインストールする場合は、永続的な設定（`gateway.auth.password` または設定 `env`）を使用してください。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定されており、`gateway.auth.mode` が未設定の場合、モードが明示的に設定されるまでインストールはブロックされます。
- ライフサイクルコマンドはスクリプト作成のために `--json` を受け付けます。

## ゲートウェイの検出（Bonjour）

`gateway discover` は、ゲートウェイビーコン（`_openclaw-gw._tcp`）をスキャンします。

- マルチキャスト DNS-SD：`local.`
- ユニキャスト DNS-SD（Wide-Area Bonjour）：ドメインを選択し（例：`openclaw.internal.`）、スプリット DNS と DNS サーバーを設定します。詳細は [/gateway/bonjour](/gateway/bonjour) を参照してください。

Bonjour 検出が有効なゲートウェイ（デフォルト）のみがビーコンをアドバタイズします。

Wide-Area 検出レコードには以下が含まれます（TXT）：

- `role`（ゲートウェイロールのヒント）
- `transport`（トランスポートのヒント、例：`gateway`）
- `gatewayPort`（WebSocket ポート、通常は `18789`）
- `sshPort`（SSH ポート；存在しない場合はデフォルト `22`）
- `tailnetDns`（MagicDNS ホスト名、利用可能な場合）
- `gatewayTls` / `gatewayTlsSha256`（TLS 有効 + 証明書フィンガープリント）
- `cliPath`（リモートインストールのオプションヒント）

### `gateway discover`

```bash
openclaw gateway discover
```

オプション：

- `--timeout <ms>`：コマンドごとのタイムアウト（browse/resolve）；デフォルト `2000`。
- `--json`：機械読み取り可能な出力（スタイリング/スピナーも無効化）。

使用例：

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```
