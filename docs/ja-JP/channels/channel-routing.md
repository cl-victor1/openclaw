---
summary: "チャンネルごとのルーティングルール（WhatsApp、Telegram、Discord、Slack）と共有コンテキスト"
read_when:
  - チャンネルのルーティングや受信ボックスの動作を変更する場合
title: "チャンネルルーティング"
---

# チャンネルとルーティング

OpenClaw はメッセージが届いたチャンネルに**返信をルーティング**します。
モデルがチャンネルを選択することはなく、ルーティングは決定論的でホスト設定により制御されます。

## 主要な用語

- **チャンネル**: `whatsapp`、`telegram`、`discord`、`slack`、`signal`、`imessage`、`webchat`。
- **AccountId**: チャンネルごとのアカウントインスタンス（サポートされている場合）。
- オプションのチャンネルデフォルトアカウント: `channels.<channel>.defaultAccount` は、送信パスに `accountId` が指定されていない場合に使用するアカウントを選択します。
  - マルチアカウント設定では、2 つ以上のアカウントが設定されている場合に明示的なデフォルト（`defaultAccount` または `accounts.default`）を設定してください。設定しない場合、フォールバックルーティングが最初の正規化されたアカウント ID を選択することがあります。
- **AgentId**: 独立したワークスペース + セッションストア（「ブレイン」）。
- **SessionKey**: コンテキストの保存と同時実行制御に使用されるバケットキー。

## セッションキーの形式（例）

ダイレクトメッセージはエージェントの**メイン**セッションに集約されます:

- `agent:<agentId>:<mainKey>`（デフォルト: `agent:main:main`）

グループとチャンネルはチャンネルごとに分離されます:

- グループ: `agent:<agentId>:<channel>:group:<id>`
- チャンネル/ルーム: `agent:<agentId>:<channel>:channel:<id>`

スレッド:

- Slack/Discord のスレッドはベースキーに `:thread:<threadId>` を追加します。
- Telegram フォーラムトピックはグループキーに `:topic:<topicId>` を埋め込みます。

例:

- `agent:main:telegram:group:-1001234567890:topic:42`
- `agent:main:discord:channel:123456:thread:987654`

## メイン DM ルートのピン固定

`session.dmScope` が `main` の場合、ダイレクトメッセージは 1 つのメインセッションを共有することがあります。
オーナー以外の DM によってセッションの `lastRoute` が上書きされないよう、以下のすべてが真の場合、OpenClaw は `allowFrom` からピン固定されたオーナーを推定します:

- `allowFrom` にワイルドカードでないエントリが 1 件のみある。
- そのエントリがそのチャンネルの具体的な送信者 ID に正規化できる。
- 受信 DM の送信者がそのピン固定されたオーナーと一致しない。

この不一致ケースでは、OpenClaw は受信セッションのメタデータを記録しますが、メインセッションの `lastRoute` の更新はスキップします。

## ルーティングルール（エージェントの選択方法）

ルーティングは受信メッセージごとに**1 つのエージェント**を選択します:

1. **完全なピア一致**（`bindings` で `peer.kind` + `peer.id` を指定）。
2. **親ピア一致**（スレッドの継承）。
3. **ギルド + ロール一致**（Discord）`guildId` + `roles` による。
4. **ギルド一致**（Discord）`guildId` による。
5. **チーム一致**（Slack）`teamId` による。
6. **アカウント一致**（チャンネルの `accountId`）。
7. **チャンネル一致**（そのチャンネルの任意のアカウント、`accountId: "*"`）。
8. **デフォルトエージェント**（`agents.list[].default`、なければ最初のリストエントリ、フォールバックは `main`）。

バインディングに複数のマッチフィールド（`peer`、`guildId`、`teamId`、`roles`）が含まれる場合、そのバインディングが適用されるには**すべての指定フィールドが一致する必要があります**。

マッチしたエージェントがどのワークスペースとセッションストアを使用するかを決定します。

## ブロードキャストグループ（複数エージェントの実行）

ブロードキャストグループを使用すると、OpenClaw が通常返信する場合（例: WhatsApp グループ内のメンション/アクティベーションゲーティング後）に、同じピアに対して**複数のエージェント**を実行できます。

設定:

```json5
{
  broadcast: {
    strategy: "parallel",
    "120363403215116621@g.us": ["alfred", "baerbel"],
    "+15555550123": ["support", "logger"],
  },
}
```

参照: [ブロードキャストグループ](/channels/broadcast-groups)。

## 設定の概要

- `agents.list`: 名前付きエージェント定義（ワークスペース、モデルなど）。
- `bindings`: 受信チャンネル/アカウント/ピアをエージェントにマッピング。

例:

```json5
{
  agents: {
    list: [{ id: "support", name: "Support", workspace: "~/.openclaw/workspace-support" }],
  },
  bindings: [
    { match: { channel: "slack", teamId: "T123" }, agentId: "support" },
    { match: { channel: "telegram", peer: { kind: "group", id: "-100123" } }, agentId: "support" },
  ],
}
```

## セッションストレージ

セッションストアは状態ディレクトリ（デフォルト `~/.openclaw`）以下に保存されます:

- `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- JSONL トランスクリプトはストアと同じ場所に保存されます

ストアパスは `session.store` と `{agentId}` テンプレートでオーバーライドできます。

ゲートウェイと ACP のセッション検出は、デフォルト `agents/` ルートおよびテンプレート化された `session.store` ルート以下のディスクバックアップされたエージェントストアもスキャンします。検出されたストアはその解決済みエージェントルート内に存在し、通常の `sessions.json` ファイルを使用する必要があります。シンボリックリンクとルート外パスは無視されます。

## WebChat の動作

WebChat は**選択されたエージェント**にアタッチし、エージェントのメインセッションをデフォルトとして使用します。このため、WebChat を使用すると、そのエージェントのクロスチャンネルコンテキストを 1 か所で確認できます。

## 返信コンテキスト

受信返信には以下が含まれます:

- 利用可能な場合、`ReplyToId`、`ReplyToBody`、および `ReplyToSender`。
- 引用されたコンテキストは `[Replying to ...]` ブロックとして `Body` に追加されます。

これはチャンネル間で一貫しています。
