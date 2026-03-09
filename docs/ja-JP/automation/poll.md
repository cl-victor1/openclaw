---
summary: "Gateway + CLIを通じたポール送信"
read_when:
  - ポールサポートの追加または変更
  - CLIまたはGatewayからのポール送信のデバッグ
title: "Polls"
---

# Polls

## サポートされているチャンネル

- Telegram
- WhatsApp（Webチャンネル）
- Discord
- MS Teams（Adaptive Cards）

## CLI

```bash
# Telegram
openclaw message poll --channel telegram --target 123456789 \
  --poll-question "Ship it?" --poll-option "Yes" --poll-option "No"
openclaw message poll --channel telegram --target -1001234567890:topic:42 \
  --poll-question "Pick a time" --poll-option "10am" --poll-option "2pm" \
  --poll-duration-seconds 300

# WhatsApp
openclaw message poll --target +15555550123 \
  --poll-question "Lunch today?" --poll-option "Yes" --poll-option "No" --poll-option "Maybe"
openclaw message poll --target 123456789@g.us \
  --poll-question "Meeting time?" --poll-option "10am" --poll-option "2pm" --poll-option "4pm" --poll-multi

# Discord
openclaw message poll --channel discord --target channel:123456789 \
  --poll-question "Snack?" --poll-option "Pizza" --poll-option "Sushi"
openclaw message poll --channel discord --target channel:123456789 \
  --poll-question "Plan?" --poll-option "A" --poll-option "B" --poll-duration-hours 48

# MS Teams
openclaw message poll --channel msteams --target conversation:19:abc@thread.tacv2 \
  --poll-question "Lunch?" --poll-option "Pizza" --poll-option "Sushi"
```

オプション:

- `--channel`: `whatsapp`（デフォルト）、`telegram`、`discord`、または `msteams`
- `--poll-multi`: 複数の選択肢を選択できるようにする
- `--poll-duration-hours`: Discordのみ（省略時のデフォルトは24）
- `--poll-duration-seconds`: Telegramのみ（5〜600秒）
- `--poll-anonymous` / `--poll-public`: Telegramのみのポール表示設定

## Gateway RPC

メソッド: `poll`

パラメーター:

- `to`（文字列、必須）
- `question`（文字列、必須）
- `options`（string[]、必須）
- `maxSelections`（数値、オプション）
- `durationHours`（数値、オプション）
- `durationSeconds`（数値、オプション、Telegramのみ）
- `isAnonymous`（ブール値、オプション、Telegramのみ）
- `channel`（文字列、オプション、デフォルト: `whatsapp`）
- `idempotencyKey`（文字列、必須）

## チャンネルの違い

- Telegram: 2〜10の選択肢。`threadId` または `:topic:` ターゲットを通じたフォーラムトピックをサポート。`durationHours` の代わりに `durationSeconds` を使用し、5〜600秒に制限。匿名ポールとパブリックポールをサポート。
- WhatsApp: 2〜12の選択肢、`maxSelections` は選択肢数の範囲内である必要があり、`durationHours` は無視されます。
- Discord: 2〜10の選択肢、`durationHours` は1〜768時間にクランプ（デフォルト24）。`maxSelections > 1` で複数選択が有効になります。Discordは厳密な選択数をサポートしません。
- MS Teams: Adaptive Cardポール（OpenClaw管理）。ネイティブのポールAPIなし。`durationHours` は無視されます。

## エージェントツール（Message）

`message` ツールを `poll` アクション（`to`、`pollQuestion`、`pollOption`、オプション `pollMulti`、`pollDurationHours`、`channel`）で使用します。

Telegramの場合、ツールは `pollDurationSeconds`、`pollAnonymous`、`pollPublic` も受け付けます。

ポール作成には `action: "poll"` を使用してください。`action: "send"` でポールフィールドを渡すと拒否されます。

注: Discordには「正確にN個を選ぶ」モードがありません。`pollMulti` は複数選択にマッピングされます。
Teamsポールは投票を `~/.openclaw/msteams-polls.json` に記録するためにGatewayがオンラインである必要があるAdaptive Cardsとして表示されます。
