---
summary: "Gatewayサービスのランブック、ライフサイクル、および運用"
read_when:
  - Gatewayプロセスの実行またはデバッグ時
title: "Gatewayランブック"
---

# Gatewayランブック

このページは、Gatewayサービスの初日のスタートアップと2日目以降の運用に使用します。

<CardGroup cols={2}>
  <Card title="詳細なトラブルシューティング" icon="siren" href="/gateway/troubleshooting">
    症状優先の診断と、正確なコマンドラダーおよびログシグネチャ。
  </Card>
  <Card title="設定" icon="sliders" href="/gateway/configuration">
    タスク指向のセットアップガイドと完全な設定リファレンス。
  </Card>
  <Card title="シークレット管理" icon="key-round" href="/gateway/secrets">
    SecretRefコントラクト、ランタイムスナップショットの動作、および移行/リロード操作。
  </Card>
  <Card title="シークレットプランコントラクト" icon="shield-check" href="/gateway/secrets-plan-contract">
    `secrets apply`の正確なターゲット/パスルールとref-only認証プロファイルの動作。
  </Card>
</CardGroup>

## 5分でローカルスタートアップ

<Steps>
  <Step title="Gatewayを起動する">

```bash
openclaw gateway --port 18789
# デバッグ/トレースをstdioにミラーリング
openclaw gateway --port 18789 --verbose
# 選択したポートのリスナーを強制終了してから起動
openclaw gateway --force
```

  </Step>

  <Step title="サービスの健全性を確認する">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

正常なベースライン: `Runtime: running` および `RPC probe: ok`。

  </Step>

  <Step title="チャンネルの準備状態を検証する">

```bash
openclaw channels status --probe
```

  </Step>
</Steps>

<Note>
Gatewayの設定リロードは、アクティブな設定ファイルパス（プロファイル/状態のデフォルトから解決されるか、`OPENCLAW_CONFIG_PATH`が設定されている場合はそれ）を監視します。
デフォルトモードは`gateway.reload.mode="hybrid"`です。
</Note>

## ランタイムモデル

- ルーティング、コントロールプレーン、チャンネル接続のための常時稼働プロセス。
- 以下のための単一の多重化ポート:
  - WebSocketコントロール/RPC
  - HTTP API（OpenAI互換、Responses、ツール呼び出し）
  - コントロールUIとフック
- デフォルトのバインドモード: `loopback`。
- 認証はデフォルトで必須（`gateway.auth.token` / `gateway.auth.password`、または`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`）。

### ポートとバインドの優先順位

| 設定 | 解決順序 |
| ------------ | ------------------------------------------------------------- |
| Gatewayポート | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789` |
| バインドモード | CLI/オーバーライド → `gateway.bind` → `loopback` |

### ホットリロードモード

| `gateway.reload.mode` | 動作 |
| --------------------- | ------------------------------------------ |
| `off` | 設定リロードなし |
| `hot` | ホットセーフな変更のみ適用 |
| `restart` | リロードが必要な変更時に再起動 |
| `hybrid`（デフォルト） | 安全な場合はホット適用、必要な場合は再起動 |

## オペレーターコマンドセット

```bash
openclaw gateway status
openclaw gateway status --deep
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

## リモートアクセス

推奨: Tailscale/VPN。
フォールバック: SSHトンネル。

```bash
ssh -N -L 18789:127.0.0.1:18789 user@host
```

その後、ローカルで`ws://127.0.0.1:18789`にクライアントを接続します。

<Warning>
Gateway認証が設定されている場合、SSHトンネル経由でもクライアントは認証（`token`/`password`）を送信する必要があります。
</Warning>

参照: [リモートGateway](/gateway/remote)、[認証](/gateway/authentication)、[Tailscale](/gateway/tailscale)。

## 監視とサービスライフサイクル

本番環境に近い信頼性のために、監視付き実行を使用します。

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

LaunchAgentのラベルは`ai.openclaw.gateway`（デフォルト）または`ai.openclaw.<profile>`（名前付きプロファイル）です。`openclaw doctor`はサービス設定のドリフトを監査および修復します。

  </Tab>

  <Tab title="Linux (systemdユーザー)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

ログアウト後の永続化には、リンガリングを有効にします:

```bash
sudo loginctl enable-linger <user>
```

  </Tab>

  <Tab title="Linux (システムサービス)">

マルチユーザー/常時稼働ホストにはシステムユニットを使用します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

  </Tab>
</Tabs>

## 1つのホストで複数のGateway

ほとんどのセットアップでは**1つ**のGatewayを実行すべきです。
厳密な分離/冗長性のためにのみ複数を使用します（例えばレスキュープロファイル）。

インスタンスごとのチェックリスト:

- ユニークな`gateway.port`
- ユニークな`OPENCLAW_CONFIG_PATH`
- ユニークな`OPENCLAW_STATE_DIR`
- ユニークな`agents.defaults.workspace`

例:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

参照: [複数のGateway](/gateway/multiple-gateways)。

### 開発プロファイルのクイックパス

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

デフォルトには、分離された状態/設定とベースGatewayポート`19001`が含まれます。

## プロトコルクイックリファレンス（オペレータービュー）

- 最初のクライアントフレームは`connect`でなければなりません。
- Gatewayは`hello-ok`スナップショット（`presence`、`health`、`stateVersion`、`uptimeMs`、制限/ポリシー）を返します。
- リクエスト: `req(method, params)` → `res(ok/payload|error)`。
- 一般的なイベント: `connect.challenge`、`agent`、`chat`、`presence`、`tick`、`health`、`heartbeat`、`shutdown`。

エージェント実行は2段階です:

1. 即時受付確認（`status:"accepted"`）
2. 最終完了レスポンス（`status:"ok"|"error"`）、その間にストリーミングされた`agent`イベント。

完全なプロトコルドキュメントを参照: [Gatewayプロトコル](/gateway/protocol)。

## 運用チェック

### ライブネス

- WSを開いて`connect`を送信。
- スナップショット付きの`hello-ok`レスポンスを期待。

### レディネス

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### ギャップリカバリ

イベントはリプレイされません。シーケンスギャップが発生した場合、続行する前に状態（`health`、`system-presence`）をリフレッシュします。

## 一般的な障害シグネチャ

| シグネチャ | 考えられる問題 |
| -------------------------------------------------------------- | ---------------------------------------- |
| `refusing to bind gateway ... without auth` | 非ループバックバインドでtoken/passwordなし |
| `another gateway instance is already listening` / `EADDRINUSE` | ポート競合 |
| `Gateway start blocked: set gateway.mode=local` | 設定がリモートモードに設定されている |
| `unauthorized` during connect | クライアントとGateway間の認証不一致 |

完全な診断ラダーについては、[Gatewayトラブルシューティング](/gateway/troubleshooting)を使用してください。

## 安全性の保証

- Gatewayプロトコルクライアントは、Gatewayが利用できない場合に高速で失敗します（暗黙的な直接チャンネルフォールバックなし）。
- 無効な/非connectの最初のフレームは拒否されクローズされます。
- グレースフルシャットダウンは、ソケットクローズの前に`shutdown`イベントを発行します。

---

関連:

- [トラブルシューティング](/gateway/troubleshooting)
- [バックグラウンドプロセス](/gateway/background-process)
- [設定](/gateway/configuration)
- [ヘルス](/gateway/health)
- [Doctor](/gateway/doctor)
- [認証](/gateway/authentication)
