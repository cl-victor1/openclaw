---
summary: "ネイティブ zca-js（QR ログイン）による Zalo 個人アカウントのサポート、機能、および設定"
read_when:
  - OpenClaw 向けに Zalo Personal を設定する場合
  - Zalo Personal のログインまたはメッセージフローのデバッグ時
title: "Zalo Personal"
---

# Zalo Personal（非公式）

ステータス: 実験的。この統合は OpenClaw 内のネイティブ `zca-js` を通じて**個人 Zalo アカウント**を自動化します。

> **警告:** これは非公式の統合であり、アカウントの停止/凍結につながる可能性があります。自己責任でご使用ください。

## プラグインが必要です

Zalo Personal はプラグインとして提供されており、コアインストールには含まれていません。

- CLI 経由でインストール: `openclaw plugins install @openclaw/zalouser`
- またはソースチェックアウトから: `openclaw plugins install ./extensions/zalouser`
- 詳細: [プラグイン](/tools/plugin)

外部の `zca`/`openzca` CLI バイナリは不要です。

## クイックセットアップ（初心者向け）

1. プラグインをインストールします（上記参照）。
2. ログイン（QR、ゲートウェイマシン上で）:
   - `openclaw channels login --channel zalouser`
   - Zalo モバイルアプリで QR コードをスキャンします。
3. チャンネルを有効にします:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

4. ゲートウェイを再起動します（またはオンボーディングを完了します）。
5. DM アクセスはデフォルトでペアリングとなります。初回連絡時にペアリングコードを承認してください。

## 概要

- `zca-js` を通じてプロセス内で完全に実行されます。
- ネイティブイベントリスナーを使用して受信メッセージを受け取ります。
- JS API を通じて直接返信を送信します（テキスト/メディア/リンク）。
- Zalo Bot API が利用できない「個人アカウント」ユースケース向けに設計されています。

## 命名規則

チャンネル ID は `zalouser` で、これが**個人 Zalo ユーザーアカウント**（非公式）を自動化するものであることを明示しています。`zalo` は将来の公式 Zalo API 統合のために予約されています。

## ID の検索（ディレクトリ）

ディレクトリ CLI を使用してピア/グループとその ID を検索します:

```bash
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "work"
```

## 制限事項

- 送信テキストは約 2000 文字にチャンク分割されます（Zalo クライアントの制限）。
- デフォルトではストリーミングはブロックされます。

## アクセス制御（DM）

`channels.zalouser.dmPolicy` のサポート: `pairing | allowlist | open | disabled`（デフォルト: `pairing`）。

`channels.zalouser.allowFrom` はユーザー ID または名前を受け付けます。オンボーディング時に、名前はプラグインのプロセス内連絡先検索を使用して ID に解決されます。

承認方法:

- `openclaw pairing list zalouser`
- `openclaw pairing approve zalouser <code>`

## グループアクセス（オプション）

- デフォルト: `channels.zalouser.groupPolicy = "open"`（グループ許可）。未設定時はデフォルトを上書きするために `channels.defaults.groupPolicy` を使用します。
- 許可リストで制限する場合:
  - `channels.zalouser.groupPolicy = "allowlist"`
  - `channels.zalouser.groups`（キーは安定したグループ ID である必要があります。名前は可能な場合、起動時に ID に解決されます）
  - `channels.zalouser.groupAllowFrom`（許可されたグループ内でボットをトリガーできる送信者を制御します）
- すべてのグループをブロック: `channels.zalouser.groupPolicy = "disabled"`。
- 設定ウィザードでグループ許可リストを求めることができます。
- 起動時に OpenClaw は許可リスト内のグループ/ユーザー名を ID に解決し、マッピングをログに記録します。
- グループ許可リストのマッチングはデフォルトで ID のみです。未解決の名前は `channels.zalouser.dangerouslyAllowNameMatching: true` が有効でない限り認証に無視されます。
- `channels.zalouser.dangerouslyAllowNameMatching: true` は変更可能なグループ名マッチングを再有効化するブレークグラス互換モードです。
- `groupAllowFrom` が未設定の場合、ランタイムはグループ送信者チェックに `allowFrom` にフォールバックします。
- 送信者チェックは通常のグループメッセージとコントロールコマンド（例: `/new`、`/reset`）の両方に適用されます。

例:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["1471383327500481391"],
      groups: {
        "123456789": { allow: true },
        "Work Chat": { allow: true },
      },
    },
  },
}
```

### グループメンションゲーティング

- `channels.zalouser.groups.<group>.requireMention` はグループ返信にメンションが必要かどうかを制御します。
- 解決順序: 正確なグループ ID/名前 -> 正規化されたグループスラッグ -> `*` -> デフォルト（`true`）。
- これは許可リストに登録されたグループとオープングループモードの両方に適用されます。
- 承認されたコントロールコマンド（例: `/new`）はメンションゲーティングをバイパスできます。
- グループメッセージがメンション必須のためスキップされた場合、OpenClaw はそれを保留グループ履歴として保存し、次に処理されるグループメッセージに含めます。
- グループ履歴の上限はデフォルトで `messages.groupChat.historyLimit`（フォールバック `50`）です。`channels.zalouser.historyLimit` でアカウントごとに上書きできます。

例:

```json5
{
  channels: {
    zalouser: {
      groupPolicy: "allowlist",
      groups: {
        "*": { allow: true, requireMention: true },
        "Work Chat": { allow: true, requireMention: false },
      },
    },
  },
}
```

## マルチアカウント

アカウントは OpenClaw の状態の `zalouser` プロファイルにマップされます。例:

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      defaultAccount: "default",
      accounts: {
        work: { enabled: true, profile: "work" },
      },
    },
  },
}
```

## タイピング、リアクション、配信確認

- OpenClaw は返信を送信する前にタイピングイベントを送信します（ベストエフォート）。
- メッセージリアクションアクション `react` は `zalouser` のチャンネルアクションでサポートされています。
  - メッセージから特定のリアクション絵文字を削除するには `remove: true` を使用します。
  - リアクションのセマンティクス: [リアクション](/tools/reactions)
- イベントメタデータを含む受信メッセージについて、OpenClaw は配信済み + 既読確認を送信します（ベストエフォート）。

## トラブルシューティング

**ログインが維持されない場合:**

- `openclaw channels status --probe`
- 再ログイン: `openclaw channels logout --channel zalouser && openclaw channels login --channel zalouser`

**許可リスト/グループ名が解決されない場合:**

- `allowFrom`/`groupAllowFrom`/`groups` には数値 ID か、正確な友達/グループ名を使用してください。

**旧 CLI ベースのセットアップからアップグレードした場合:**

- 古い外部 `zca` プロセスの前提を削除してください。
- チャンネルは外部 CLI バイナリなしに OpenClaw 内で完全に動作するようになりました。
