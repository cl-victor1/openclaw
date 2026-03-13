---
summary: "NIP-04 暗号化メッセージを使用した Nostr DM チャンネル"
read_when:
  - Nostr 経由で OpenClaw が DM を受信できるようにしたい場合
  - 分散型メッセージングを設定する場合
title: "Nostr"
---

# Nostr

**ステータス:** オプションのプラグイン（デフォルトは無効）。

Nostr はソーシャルネットワーキング向けの分散型プロトコルです。このチャンネルにより、OpenClaw は NIP-04 経由で暗号化ダイレクトメッセージ（DM）を受信して応答できます。

## インストール（オンデマンド）

### オンボーディング（推奨）

- オンボーディングウィザード（`openclaw onboard`）と `openclaw channels add` にオプションのチャンネルプラグインが一覧表示されます。
- Nostr を選択すると、オンデマンドでプラグインをインストールするよう求められます。

インストールのデフォルト:

- **開発チャンネル + git チェックアウトが利用可能:** ローカルプラグインパスを使用します。
- **安定版/ベータ版:** npm からダウンロードします。

プロンプトで選択を上書きすることもできます。

### 手動インストール

```bash
openclaw plugins install @openclaw/nostr
```

ローカルチェックアウトを使用する場合（開発ワークフロー）:

```bash
openclaw plugins install --link <path-to-openclaw>/extensions/nostr
```

プラグインのインストールまたは有効化後にゲートウェイを再起動してください。

## クイックセットアップ

1. Nostr キーペアを生成します（必要な場合）:

```bash
# nak を使用
nak key generate
```

2. 設定に追加します:

```json
{
  "channels": {
    "nostr": {
      "privateKey": "${NOSTR_PRIVATE_KEY}"
    }
  }
}
```

3. キーをエクスポートします:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. ゲートウェイを再起動します。

## 設定リファレンス

| キー          | 型       | デフォルト                                  | 説明                                |
| ------------- | -------- | ------------------------------------------- | ----------------------------------- |
| `privateKey`  | string   | 必須                                        | `nsec` または 16 進数形式の秘密鍵   |
| `relays`      | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | リレー URL（WebSocket）             |
| `dmPolicy`    | string   | `pairing`                                   | DM アクセスポリシー                 |
| `allowFrom`   | string[] | `[]`                                        | 許可する送信者の公開鍵              |
| `enabled`     | boolean  | `true`                                      | チャンネルの有効/無効               |
| `name`        | string   | -                                           | 表示名                              |
| `profile`     | object   | -                                           | NIP-01 プロファイルメタデータ       |

## プロファイルメタデータ

プロファイルデータは NIP-01 `kind:0` イベントとして公開されます。コントロール UI（チャンネル -> Nostr -> プロファイル）から管理するか、設定に直接記述できます。

例:

```json
{
  "channels": {
    "nostr": {
      "privateKey": "${NOSTR_PRIVATE_KEY}",
      "profile": {
        "name": "openclaw",
        "displayName": "OpenClaw",
        "about": "Personal assistant DM bot",
        "picture": "https://example.com/avatar.png",
        "banner": "https://example.com/banner.png",
        "website": "https://example.com",
        "nip05": "openclaw@example.com",
        "lud16": "openclaw@example.com"
      }
    }
  }
}
```

注意:

- プロファイル URL は `https://` を使用する必要があります。
- リレーからのインポートはフィールドをマージし、ローカルの上書きを保持します。

## アクセス制御

### DM ポリシー

- **pairing**（デフォルト）: 未知の送信者にはペアリングコードが送られます。
- **allowlist**: `allowFrom` に含まれる公開鍵のみが DM できます。
- **open**: パブリックの受信 DM（`allowFrom: ["*"]` が必要）。
- **disabled**: 受信 DM を無視します。

### 許可リストの例

```json
{
  "channels": {
    "nostr": {
      "privateKey": "${NOSTR_PRIVATE_KEY}",
      "dmPolicy": "allowlist",
      "allowFrom": ["npub1abc...", "npub1xyz..."]
    }
  }
}
```

## キー形式

受け付ける形式:

- **秘密鍵:** `nsec...` または 64 文字の 16 進数
- **公開鍵（`allowFrom`）:** `npub...` または 16 進数

## リレー

デフォルト: `relay.damus.io` と `nos.lol`。

```json
{
  "channels": {
    "nostr": {
      "privateKey": "${NOSTR_PRIVATE_KEY}",
      "relays": ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"]
    }
  }
}
```

ヒント:

- 冗長性のために 2〜3 つのリレーを使用してください。
- リレーを多くしすぎないでください（レイテンシ、重複）。
- 有料リレーは信頼性を向上させます。
- テスト用にはローカルリレーも使用できます（`ws://localhost:7777`）。

## プロトコルサポート

| NIP    | ステータス    | 説明                                  |
| ------ | ------------- | ------------------------------------- |
| NIP-01 | サポート中    | 基本イベント形式 + プロファイルメタデータ |
| NIP-04 | サポート中    | 暗号化 DM（`kind:4`）                 |
| NIP-17 | 計画中        | ギフトラップ DM                       |
| NIP-44 | 計画中        | バージョン管理された暗号化            |

## テスト

### ローカルリレー

```bash
# strfry を起動
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json
{
  "channels": {
    "nostr": {
      "privateKey": "${NOSTR_PRIVATE_KEY}",
      "relays": ["ws://localhost:7777"]
    }
  }
}
```

### 手動テスト

1. ログからボットの公開鍵（npub）を確認します。
2. Nostr クライアント（Damus、Amethyst など）を開きます。
3. ボットの公開鍵に DM を送信します。
4. 返信を確認します。

## トラブルシューティング

### メッセージを受信しない場合

- 秘密鍵が有効であることを確認します。
- リレー URL が到達可能で `wss://`（またはローカルの場合 `ws://`）を使用していることを確認します。
- `enabled` が `false` でないことを確認します。
- ゲートウェイログでリレー接続エラーを確認します。

### 返信を送信しない場合

- リレーが書き込みを受け付けているか確認します。
- アウトバウンド接続を確認します。
- リレーのレート制限に注意してください。

### 重複した返信が発生する場合

- 複数のリレーを使用する場合は予想されます。
- メッセージはイベント ID で重複排除されます。最初の配信のみが返信をトリガーします。

## セキュリティ

- 秘密鍵をコミットしないでください。
- キーには環境変数を使用してください。
- 本番ボットには `allowlist` の使用を検討してください。

## 制限事項（MVP）

- ダイレクトメッセージのみ（グループチャットなし）。
- メディア添付なし。
- NIP-04 のみ（NIP-17 ギフトラップは計画中）。
