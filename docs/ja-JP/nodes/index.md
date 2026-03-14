---
summary: "ノード：canvas/camera/screen/device/notifications/system の ペアリング、機能、パーミッション、および CLI ヘルパー"
read_when:
  - iOS/Android ノードをゲートウェイにペアリングする場合
  - エージェントのコンテキスト用にノード canvas/camera を使用する場合
  - 新しいノードコマンドまたは CLI ヘルパーを追加する場合
title: "ノード"
---

# ノード

**ノード**は、`role: "node"` でゲートウェイ **WebSocket**（オペレーターと同じポート）に接続し、`node.invoke` 経由でコマンドサーフェス（例：`canvas.*`、`camera.*`、`device.*`、`notifications.*`、`system.*`）を公開するコンパニオンデバイス（macOS/iOS/Android/ヘッドレス）です。プロトコルの詳細：[ゲートウェイプロトコル](/gateway/protocol)。

レガシートランスポート：[ブリッジプロトコル](/gateway/bridge-protocol)（TCP JSONL；現在のノードでは非推奨/削除済み）。

macOS は**ノードモード**でも実行できます：メニューバーアプリがゲートウェイの WS サーバーに接続し、ローカルの canvas/camera コマンドをノードとして公開します（これにより `openclaw nodes …` がこの Mac に対して機能します）。

注意事項：

- ノードは**ペリフェラル**であり、ゲートウェイではありません。ゲートウェイサービスは実行しません。
- Telegram/WhatsApp などのメッセージはノードではなく**ゲートウェイ**に届きます。
- トラブルシューティングのランブック：[/nodes/troubleshooting](/nodes/troubleshooting)

## ペアリング + ステータス

**WS ノードはデバイスペアリングを使用します。** ノードは `connect` 中にデバイス ID を提示し、ゲートウェイは `role: node` のデバイスペアリングリクエストを作成します。デバイス CLI（または UI）経由で承認します。

クイック CLI：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

注意事項：

- ノードのデバイスペアリングロールに `node` が含まれている場合、`nodes status` はノードを**ペアリング済み**としてマークします。
- `node.pair.*`（CLI：`openclaw nodes pending/approve/reject`）はゲートウェイ所有の別のノードペアリングストアです。WS `connect` ハンドシェイクはこれによってゲートされ**ません**。

## リモートノードホスト（system.run）

ゲートウェイが 1 台のマシンで動作していて、コマンドを別のマシンで実行したい場合は**ノードホスト**を使用します。モデルは引き続き**ゲートウェイ**と通信します。`host=node` が選択された場合、ゲートウェイは `exec` 呼び出しを**ノードホスト**に転送します。

### 何がどこで実行されるか

- **ゲートウェイホスト**：メッセージを受信し、モデルを実行し、ツール呼び出しをルーティングします。
- **ノードホスト**：ノードマシンで `system.run`/`system.which` を実行します。
- **承認**：`~/.openclaw/exec-approvals.json` 経由でノードホスト上で実施されます。

承認の注意事項：

- 承認に基づくノード実行は正確なリクエストコンテキストをバインドします。
- 直接のシェル/ランタイムファイル実行の場合、OpenClaw は 1 つの具体的なローカルファイルオペランドをベストエフォートでバインドし、実行前にそのファイルが変更された場合は実行を拒否します。
- インタープリター/ランタイムコマンドに対して正確に 1 つの具体的なローカルファイルを特定できない場合、承認に基づく実行は完全なランタイムカバレッジを偽ることなく拒否されます。

### ノードホストの起動（フォアグラウンド）

ノードマシン上で：

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

### SSH トンネル経由のリモートゲートウェイ（loopback バインド）

ゲートウェイが loopback にバインドしている場合（`gateway.bind=loopback`、ローカルモードのデフォルト）、リモートノードホストは直接接続できません。SSH トンネルを作成し、ノードホストをトンネルのローカル端に向けます。

例（ノードホスト → ゲートウェイホスト）：

```bash
# ターミナル A（実行し続ける）：ローカル 18790 → ゲートウェイ 127.0.0.1:18789 を転送
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# ターミナル B：ゲートウェイトークンをエクスポートしてトンネル経由で接続
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

注意事項：

- `openclaw node run` はトークンまたはパスワード認証をサポートします。
- 環境変数が優先されます：`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`。
- 設定のフォールバックは `gateway.auth.token` / `gateway.auth.password` です。
- ローカルモードでは、ノードホストは意図的に `gateway.remote.token` / `gateway.remote.password` を無視します。
- リモートモードでは、`gateway.remote.token` / `gateway.remote.password` がリモート優先順位ルールに従って適用されます。

### ノードホストの起動（サービス）

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node restart
```

### ペアリング + 命名

ゲートウェイホスト上で：

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

命名オプション：

- `openclaw node run` / `openclaw node install` の `--display-name`（ノードの `~/.openclaw/node.json` に永続化）。
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"`（ゲートウェイのオーバーライド）。

### コマンドの許可リスト登録

Exec 承認は**ノードホストごと**です。ゲートウェイから許可リストエントリを追加します：

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

承認はノードホストの `~/.openclaw/exec-approvals.json` に保存されます。

### exec をノードに向ける

デフォルトの設定（ゲートウェイの設定）：

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.security allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

またはセッションごと：

```
/exec host=node security=allowlist node=<id-or-name>
```

設定後、`host=node` を持つ `exec` 呼び出しはノードホスト上で実行されます（ノードの許可リスト/承認に従います）。

関連：

- [ノードホスト CLI](/cli/node)
- [Exec ツール](/tools/exec)
- [Exec 承認](/tools/exec-approvals)

## コマンドの呼び出し

低レベル（生の RPC）：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

一般的な「エージェントに MEDIA 添付ファイルを渡す」ワークフロー用の高レベルのヘルパーも存在します。

## スクリーンショット（canvas スナップショット）

ノードが Canvas（WebView）を表示している場合、`canvas.snapshot` は `{ format, base64 }` を返します。

CLI ヘルパー（一時ファイルに書き込み `MEDIA:<path>` を出力）：

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### Canvas コントロール

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

注意事項：

- `canvas present` は URL またはローカルファイルパス（`--target`）、および位置指定用のオプションの `--x/--y/--width/--height` を受け入れます。
- `canvas eval` はインライン JS（`--js`）または位置引数を受け入れます。

### A2UI（Canvas）

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

注意事項：

- A2UI v0.8 JSONL のみサポートされています（v0.9/createSurface は拒否されます）。

## 写真 + 動画（ノードカメラ）

写真（`jpg`）：

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # デフォルト：両方の向き（2 つの MEDIA ライン）
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
```

動画クリップ（`mp4`）：

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

注意事項：

- `canvas.*` と `camera.*` はノードが**フォアグラウンド**にある必要があります（バックグラウンド呼び出しは `NODE_BACKGROUND_UNAVAILABLE` を返します）。
- クリップの長さはクランプされます（現在は `<= 60s`）、過大な base64 ペイロードを避けるためです。
- Android では可能な場合 `CAMERA`/`RECORD_AUDIO` のパーミッションを要求します。拒否されたパーミッションは `*_PERMISSION_REQUIRED` で失敗します。

## 画面録画（ノード）

サポートされているノードは `screen.record`（mp4）を公開します。例：

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

注意事項：

- `screen.record` の利用可能性はノードプラットフォームによって異なります。
- 画面録画は `<= 60s` にクランプされます。
- `--no-audio` はサポートされているプラットフォームでマイクキャプチャを無効にします。
- 複数の画面がある場合、`--screen <index>` でディスプレイを選択します。

## 位置情報（ノード）

設定で位置情報が有効になっている場合、ノードは `location.get` を公開します。

CLI ヘルパー：

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

注意事項：

- 位置情報は**デフォルトでオフ**です。
- 「常に」はシステムパーミッションが必要です。バックグラウンドフェッチはベストエフォートです。
- レスポンスには緯度/経度、精度（メートル）、タイムスタンプが含まれます。

## SMS（Android ノード）

ユーザーが **SMS** パーミッションを付与し、デバイスがテレフォニーをサポートしている場合、Android ノードは `sms.send` を公開できます。

低レベルの呼び出し：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

注意事項：

- 機能が通知される前に、Android デバイスでパーミッションプロンプトを受け入れる必要があります。
- テレフォニーなしの Wi-Fi のみのデバイスは `sms.send` を通知しません。

## Android デバイス + 個人データコマンド

Android ノードは、対応する機能が有効になっている場合、追加のコマンドファミリーを通知できます。

利用可能なファミリー：

- `device.status`、`device.info`、`device.permissions`、`device.health`
- `notifications.list`、`notifications.actions`
- `photos.latest`
- `contacts.search`、`contacts.add`
- `calendar.events`、`calendar.add`
- `motion.activity`、`motion.pedometer`

呼び出しの例：

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

注意事項：

- モーションコマンドは利用可能なセンサーによって機能がゲートされています。

## システムコマンド（ノードホスト / Mac ノード）

macOS ノードは `system.run`、`system.notify`、`system.execApprovals.get/set` を公開します。
ヘッドレスノードホストは `system.run`、`system.which`、`system.execApprovals.get/set` を公開します。

例：

```bash
openclaw nodes run --node <idOrNameOrIp> -- echo "Hello from mac node"
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway ready"
```

注意事項：

- `system.run` はペイロードに stdout/stderr/終了コードを返します。
- `system.notify` は macOS アプリの通知パーミッション状態を尊重します。
- 不明なノード `platform` / `deviceFamily` メタデータは、`system.run` と `system.which` を除外する保守的なデフォルト許可リストを使用します。不明なプラットフォームでこれらのコマンドが必要な場合は、`gateway.nodes.allowCommands` で明示的に追加してください。
- `system.run` は `--cwd`、`--env KEY=VAL`、`--command-timeout`、`--needs-screen-recording` をサポートします。
- シェルラッパー（`bash|sh|zsh ... -c/-lc`）の場合、リクエストスコープの `--env` 値は明示的な許可リスト（`TERM`、`LANG`、`LC_*`、`COLORTERM`、`NO_COLOR`、`FORCE_COLOR`）に減らされます。
- `system.notify` は `--priority <passive|active|timeSensitive>` と `--delivery <system|overlay|auto>` をサポートします。
- ノードホストは `PATH` のオーバーライドを無視し、危険なスタートアップ/シェルキー（`DYLD_*`、`LD_*`、`NODE_OPTIONS`、`PYTHON*`、`PERL*`、`RUBYOPT`、`SHELLOPTS`、`PS4`）を除去します。
- macOS ノードモードでは、`system.run` は macOS アプリの exec 承認によってゲートされます（設定 → Exec 承認）。Ask/allowlist/full はヘッドレスノードホストと同じように動作します。拒否されたプロンプトは `SYSTEM_RUN_DENIED` を返します。
- ヘッドレスノードホストでは、`system.run` は exec 承認（`~/.openclaw/exec-approvals.json`）によってゲートされます。

## Exec ノードバインディング

複数のノードが利用可能な場合、exec を特定のノードにバインドできます。これは `exec host=node` のデフォルトノードを設定します（エージェントごとにオーバーライド可能）。

グローバルデフォルト：

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

エージェントごとのオーバーライド：

```bash
openclaw config get agents.list
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
```

任意のノードを許可するために設定を解除：

```bash
openclaw config unset tools.exec.node
openclaw config unset agents.list[0].tools.exec.node
```

## パーミッションマップ

ノードは `node.list` / `node.describe` にパーミッション名（例：`screenRecording`、`accessibility`）をキーとし、ブール値（`true` = 付与済み）を持つ `permissions` マップを含む場合があります。

## ヘッドレスノードホスト（クロスプラットフォーム）

OpenClaw はゲートウェイ WebSocket に接続して `system.run` / `system.which` を公開する**ヘッドレスノードホスト**（UI なし）を実行できます。これは Linux/Windows やサーバーの隣に最小限のノードを実行するのに便利です。

起動方法：

```bash
openclaw node run --host <gateway-host> --port 18789
```

注意事項：

- ペアリングはまだ必要です（ゲートウェイはデバイスペアリングプロンプトを表示します）。
- ノードホストはノード ID、トークン、表示名、ゲートウェイ接続情報を `~/.openclaw/node.json` に保存します。
- Exec 承認は `~/.openclaw/exec-approvals.json` 経由でローカルに実施されます（[Exec 承認](/tools/exec-approvals) を参照）。
- macOS では、ヘッドレスノードホストはデフォルトで `system.run` をローカルで実行します。`OPENCLAW_NODE_EXEC_HOST=app` を設定してコンパニオンアプリの exec ホスト経由で `system.run` をルーティングします。`OPENCLAW_NODE_EXEC_FALLBACK=0` を追加してアプリホストを必須にし、利用不可の場合はクローズドで失敗します。
- ゲートウェイ WS が TLS を使用している場合は `--tls` / `--tls-fingerprint` を追加します。

## Mac ノードモード

- macOS メニューバーアプリはノードとしてゲートウェイ WS サーバーに接続します（これにより `openclaw nodes …` がこの Mac に対して機能します）。
- リモートモードでは、アプリはゲートウェイポートの SSH トンネルを開き、`localhost` に接続します。
