---
summary: "WhatsApp グループメッセージの処理動作と設定（mentionPatterns は複数のサーフェスで共有されます）"
read_when:
  - グループメッセージのルールやメンションの変更時
title: "グループメッセージ"
---

# グループメッセージ（WhatsApp ウェブチャンネル）

目的: Clawd を WhatsApp グループに参加させ、呼ばれたときだけ応答し、そのスレッドをパーソナル DM セッションとは分離して管理することです。

注意: `agents.list[].groupChat.mentionPatterns` は現在 Telegram/Discord/Slack/iMessage でも使用されています。このドキュメントは WhatsApp 固有の動作に焦点を当てています。マルチエージェント構成の場合は、エージェントごとに `agents.list[].groupChat.mentionPatterns` を設定するか（または `messages.groupChat.mentionPatterns` をグローバルフォールバックとして使用してください）。

## 実装済み機能（2025-12-03）

- アクティベーションモード: `mention`（デフォルト）または `always`。`mention` はピング（`mentionedJids` 経由の WhatsApp @-メンション、正規表現パターン、またはテキスト内の E.164 ボット番号）を必要とします。`always` はすべてのメッセージでエージェントを起動しますが、意味のある価値を追加できる場合のみ応答し、それ以外はサイレントトークン `NO_REPLY` を返します。デフォルトは設定（`channels.whatsapp.groups`）で設定でき、`/activation` でグループごとに上書きできます。`channels.whatsapp.groups` を設定すると、グループの許可リストとしても機能します（すべてを許可するには `"*"` を含めてください）。
- グループポリシー: `channels.whatsapp.groupPolicy` はグループメッセージを受け付けるかどうかを制御します（`open|disabled|allowlist`）。`allowlist` は `channels.whatsapp.groupAllowFrom`（フォールバック: 明示的な `channels.whatsapp.allowFrom`）を使用します。デフォルトは `allowlist`（送信者を追加するまでブロック）。
- グループごとのセッション: セッションキーは `agent:<agentId>:whatsapp:group:<jid>` の形式で、`/verbose on` や `/think high`（スタンドアロンメッセージとして送信）などのコマンドはそのグループのスコープになります。パーソナル DM の状態には影響しません。ハートビートはグループスレッドではスキップされます。
- コンテキスト注入: 実行をトリガーしなかった **保留中のみ** のグループメッセージ（デフォルト 50 件）は `[Chat messages since your last reply - for context]` の下にプレフィックスされ、トリガーとなった行は `[Current message - respond to this]` の下に表示されます。すでにセッションにあるメッセージは再注入されません。
- 送信者の表示: すべてのグループバッチは `[from: Sender Name (+E164)]` で終わるため、Pi は誰が話しているかを認識できます。
- エフェメラル/一度見: テキスト/メンションを抽出する前にアンラップするため、その中のピングもトリガーとなります。
- グループシステムプロンプト: グループセッションの最初のターン（および `/activation` でモードが変わるたびに）、システムプロンプトに短いテキストを注入します（例: `You are replying inside the WhatsApp group "<subject>". Group members: Alice (+44...), Bob (+43...), … Activation: trigger-only … Address the specific sender noted in the message context.`）。メタデータが利用できない場合でも、グループチャットであることをエージェントに伝えます。

## 設定例（WhatsApp）

`~/.openclaw/openclaw.json` に `groupChat` ブロックを追加して、WhatsApp がテキスト本文から視覚的な `@` を削除した場合でも表示名によるピングが機能するようにします:

```json5
{
  channels: {
    whatsapp: {
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: {
          historyLimit: 50,
          mentionPatterns: ["@?openclaw", "\\+?15555550123"],
        },
      },
    ],
  },
}
```

注意:

- 正規表現は大文字小文字を区別しません。`@openclaw` のような表示名ピングと、`+` や空白の有無にかかわらず生の番号に対応します。
- WhatsApp は連絡先をタップしたときに `mentionedJids` 経由で正規のメンションを送信するため、番号のフォールバックはほとんど必要ありませんが、有用な安全網として機能します。

### アクティベーションコマンド（オーナーのみ）

グループチャットコマンドを使用します:

- `/activation mention`
- `/activation always`

変更できるのはオーナー番号（`channels.whatsapp.allowFrom` から、または未設定の場合はボット自身の E.164）のみです。グループ内でスタンドアロンメッセージとして `/status` を送信すると、現在のアクティベーションモードを確認できます。

## 使い方

1. WhatsApp アカウント（OpenClaw を実行しているもの）をグループに追加します。
2. `@openclaw …` と送信します（または番号を含めます）。`groupPolicy: "open"` を設定しない限り、許可リストに登録された送信者のみがトリガーできます。
3. エージェントプロンプトには最近のグループコンテキストと末尾の `[from: …]` マーカーが含まれ、適切な人物に対応できます。
4. セッションレベルのディレクティブ（`/verbose on`、`/think high`、`/new` または `/reset`、`/compact`）はそのグループのセッションにのみ適用されます。スタンドアロンメッセージとして送信して登録させてください。パーソナル DM セッションは独立したままです。

## テスト/検証

- 手動スモークテスト:
  - グループで `@openclaw` ピングを送信し、送信者名を参照した返信を確認します。
  - 2 番目のピングを送信し、次のターンで履歴ブロックが含まれ、その後クリアされることを確認します。
- ゲートウェイログ（`--verbose` で実行）で `inbound web message` エントリを確認し、`from: <groupJid>` と `[from: …]` サフィックスが表示されることを確認します。

## 既知の考慮事項

- ハートビートは、うるさいブロードキャストを避けるためにグループでは意図的にスキップされます。
- エコー抑制は結合されたバッチ文字列を使用します。メンションなしに同じテキストを 2 回送信すると、最初のものだけが応答されます。
- セッションストアのエントリはセッションストア（デフォルト: `~/.openclaw/agents/<agentId>/sessions/sessions.json`）に `agent:<agentId>:whatsapp:group:<jid>` として表示されます。エントリがない場合は、グループがまだ実行をトリガーしていないことを意味します。
- グループのタイピングインジケーターは `agents.defaults.typingMode` に従います（デフォルト: メンションなしの場合 `message`）。
