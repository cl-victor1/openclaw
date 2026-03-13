---
summary: "`openclaw browser` の CLI リファレンス（プロファイル、タブ、アクション、拡張機能リレー）"
read_when:
  - "`openclaw browser` を使用して一般的なタスクの例を確認したいとき"
  - ノードホスト経由で別のマシンで実行中のブラウザを制御したいとき
  - Chrome 拡張機能リレー（ツールバーボタンでアタッチ/デタッチ）を使用したいとき
title: "browser"
---

# `openclaw browser`

OpenClaw のブラウザコントロールサーバーを管理し、ブラウザアクション（タブ、スナップショット、スクリーンショット、ナビゲーション、クリック、入力）を実行します。

関連ドキュメント:

- ブラウザツール + API: [ブラウザツール](/tools/browser)
- Chrome 拡張機能リレー: [Chrome 拡張機能](/tools/chrome-extension)

## 共通フラグ

- `--url <gatewayWsUrl>`: Gateway WebSocket URL（デフォルトは設定値）。
- `--token <token>`: Gateway トークン（必要な場合）。
- `--timeout <ms>`: リクエストタイムアウト（ミリ秒）。
- `--browser-profile <name>`: ブラウザプロファイルを選択（デフォルトは設定値）。
- `--json`: 機械可読出力（対応している場合）。

## クイックスタート（ローカル）

```bash
openclaw browser --browser-profile chrome tabs
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

## プロファイル

プロファイルは名前付きのブラウザルーティング設定です。実際には:

- `openclaw`: 専用の OpenClaw 管理 Chrome インスタンス（独立したユーザーデータディレクトリ）を起動/アタッチします。
- `chrome`: Chrome 拡張機能リレーを介して既存の Chrome タブを制御します。

```bash
openclaw browser profiles
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser delete-profile --name work
```

特定のプロファイルを使用:

```bash
openclaw browser --browser-profile work tabs
```

## タブ

```bash
openclaw browser tabs
openclaw browser open https://docs.openclaw.ai
openclaw browser focus <targetId>
openclaw browser close <targetId>
```

## スナップショット / スクリーンショット / アクション

スナップショット:

```bash
openclaw browser snapshot
```

スクリーンショット:

```bash
openclaw browser screenshot
```

ナビゲーション/クリック/入力（ref ベースの UI オートメーション）:

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser type <ref> "hello"
```

## Chrome 拡張機能リレー（ツールバーボタンでアタッチ）

このモードでは、手動でアタッチした既存の Chrome タブをエージェントが制御できます（自動アタッチはされません）。

安定したパスにアンパック拡張機能をインストールします:

```bash
openclaw browser extension install
openclaw browser extension path
```

次に Chrome → `chrome://extensions` →「デベロッパーモード」を有効化 →「パッケージ化されていない拡張機能を読み込む」→ 表示されたフォルダを選択。

完全ガイド: [Chrome 拡張機能](/tools/chrome-extension)

## リモートブラウザ制御（ノードホストプロキシ）

Gateway がブラウザとは別のマシンで実行されている場合、Chrome/Brave/Edge/Chromium がインストールされているマシンで**ノードホスト**を実行します。Gateway はブラウザアクションをそのノードにプロキシします（別途ブラウザコントロールサーバーは不要です）。

`gateway.nodes.browser.mode` で自動ルーティングを制御し、複数のノードが接続されている場合は `gateway.nodes.browser.node` で特定のノードを固定します。

セキュリティとリモート設定: [ブラウザツール](/tools/browser)、[リモートアクセス](/gateway/remote)、[Tailscale](/gateway/tailscale)、[セキュリティ](/gateway/security)
