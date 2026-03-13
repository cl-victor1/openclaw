---
summary: "ハートビートのポーリングメッセージと通知ルール"
read_when:
  - ハートビートの頻度やメッセージングを調整する場合
  - スケジュールタスクにハートビートとcronのどちらを使うか決める場合
title: "ハートビート"
---

# ハートビート（Gateway）

> **ハートビート vs Cron？** どちらを使うべきかのガイダンスについては[Cron vs ハートビート](/automation/cron-vs-heartbeat)を参照してください。

ハートビートはメインセッションで**定期的なエージェントターン**を実行し、
スパムにならずに注意が必要なことをモデルが通知できるようにします。

トラブルシューティング: [/automation/troubleshooting](/automation/troubleshooting)

## クイックスタート（初心者向け）

1. ハートビートを有効のままにします（デフォルトは`30m`、Anthropic OAuth/setup-tokenの場合は`1h`）、または独自の頻度を設定します。
2. エージェントワークスペースに小さな`HEARTBEAT.md`チェックリストを作成します（オプションですが推奨）。
3. ハートビートメッセージの送信先を決めます（`target: "none"`がデフォルト、`target: "last"`で最後の連絡先にルーティング）。
4. オプション: 透明性のためにハートビートの推論配信を有効にします。
5. オプション: ハートビート実行に`HEARTBEAT.md`のみが必要な場合は軽量ブートストラップコンテキストを使用します。
6. オプション: ハートビートをアクティブ時間帯（ローカル時間）に制限します。

設定例:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先への明示的配信（デフォルトは"none"）
        directPolicy: "allow", // デフォルト: direct/DMターゲットを許可、"block"で抑制
        lightContext: true, // オプション: ブートストラップファイルからHEARTBEAT.mdのみを注入
        // activeHours: { start: "08:00", end: "24:00" },
        // includeReasoning: true, // オプション: 別のReasoning:メッセージも送信
      },
    },
  },
}
```

## デフォルト

- 間隔: `30m`（検出された認証モードがAnthropic OAuth/setup-tokenの場合は`1h`）。`agents.defaults.heartbeat.every`またはエージェントごとの`agents.list[].heartbeat.every`で設定、`0m`で無効化。
- プロンプト本文（`agents.defaults.heartbeat.prompt`で設定可能）:
  `Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
- ハートビートプロンプトはユーザーメッセージとして**そのまま**送信されます。システム
  プロンプトには「Heartbeat」セクションが含まれ、実行は内部的にフラグが付けられます。
- アクティブ時間帯（`heartbeat.activeHours`）は設定されたタイムゾーンで確認されます。
  ウィンドウ外では、ウィンドウ内の次のティックまでハートビートがスキップされます。

## ハートビートプロンプトの目的

デフォルトのプロンプトは意図的に広範です:

- **バックグラウンドタスク**: 「未処理のタスクを検討」はエージェントにフォローアップ
  （受信トレイ、カレンダー、リマインダー、キューイングされた作業）を確認し、緊急のものを通知するよう促します。
- **人間へのチェックイン**: 「日中に人間のチェックインを時々行う」は、
  設定されたローカルタイムゾーンを使用して夜間のスパムを避けながら、
  軽量な「何か必要ですか？」メッセージを促します（[/concepts/timezone](/concepts/timezone)を参照）。

ハートビートに非常に具体的な作業をさせたい場合（例: 「Gmail PubSubの統計を確認」
または「Gatewayのヘルスを検証」）、`agents.defaults.heartbeat.prompt`（または
`agents.list[].heartbeat.prompt`）にカスタム本文を設定します（そのまま送信）。

## レスポンスコントラクト

- 注意が必要なことがない場合、**`HEARTBEAT_OK`**で返信します。
- ハートビート実行中、OpenClawは返信の**先頭または末尾**に`HEARTBEAT_OK`が表示された場合に確認として扱います。トークンは削除され、残りのコンテンツが**≤ `ackMaxChars`**（デフォルト: 300）の場合、返信はドロップされます。
- `HEARTBEAT_OK`が返信の**途中**に表示された場合、特別な扱いはされません。
- アラートの場合、`HEARTBEAT_OK`を含め**ず**、アラートテキストのみを返します。

ハートビート外では、メッセージの先頭/末尾にある偶発的な`HEARTBEAT_OK`は削除されログに記録されます。`HEARTBEAT_OK`のみのメッセージはドロップされます。

## 設定

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // デフォルト: 30m（0mで無効化）
        model: "anthropic/claude-opus-4-6",
        includeReasoning: false, // デフォルト: false（利用可能な場合に別のReasoning:メッセージを配信）
        lightContext: false, // デフォルト: false、trueはワークスペースブートストラップファイルからHEARTBEAT.mdのみを保持
        target: "last", // デフォルト: none | オプション: last | none | <channel id>（コアまたはプラグイン、例: "bluebubbles"）
        to: "+15551234567", // オプションのチャンネル固有オーバーライド
        accountId: "ops-bot", // オプションのマルチアカウントチャンネルID
        prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        ackMaxChars: 300, // HEARTBEAT_OK後に許可される最大文字数
      },
    },
  },
}
```

### スコープと優先順位

- `agents.defaults.heartbeat`はグローバルなハートビート動作を設定します。
- `agents.list[].heartbeat`はその上にマージされます。いずれかのエージェントに`heartbeat`ブロックがある場合、**それらのエージェントのみ**がハートビートを実行します。
- `channels.defaults.heartbeat`はすべてのチャンネルの可視性デフォルトを設定します。
- `channels.<channel>.heartbeat`はチャンネルデフォルトをオーバーライドします。
- `channels.<channel>.accounts.<id>.heartbeat`（マルチアカウントチャンネル）はチャンネルごとの設定をオーバーライドします。

### エージェントごとのハートビート

`agents.list[]`エントリのいずれかに`heartbeat`ブロックが含まれている場合、
**それらのエージェントのみ**がハートビートを実行します。エージェントごとのブロックは
`agents.defaults.heartbeat`の上にマージされます（共通のデフォルトを一度設定し、
エージェントごとにオーバーライドできます）。

例: 2つのエージェント、2番目のエージェントのみがハートビートを実行。

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先への明示的配信（デフォルトは"none"）
      },
    },
    list: [
      { id: "main", default: true },
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "whatsapp",
          to: "+15551234567",
          prompt: "Read HEARTBEAT.md if it exists (workspace context). Follow it strictly. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.",
        },
      },
    ],
  },
}
```

### アクティブ時間帯の例

特定のタイムゾーンでハートビートを業務時間に制限します:

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m",
        target: "last", // 最後の連絡先への明示的配信（デフォルトは"none"）
        activeHours: {
          start: "09:00",
          end: "22:00",
          timezone: "America/New_York", // オプション、設定されている場合はuserTimezoneを使用、それ以外はホストのタイムゾーン
        },
      },
    },
  },
}
```

このウィンドウ外（東部時間の午前9時前または午後10時後）では、ハートビートはスキップされます。ウィンドウ内の次のスケジュールされたティックは通常通り実行されます。

### 24時間365日のセットアップ

ハートビートを終日実行したい場合は、以下のパターンのいずれかを使用します:

- `activeHours`を完全に省略（時間ウィンドウの制限なし、これがデフォルトの動作）。
- 終日ウィンドウを設定: `activeHours: { start: "00:00", end: "24:00" }`。

`start`と`end`に同じ時間を設定しないでください（例: `08:00`から`08:00`）。
ゼロ幅ウィンドウとして扱われ、ハートビートは常にスキップされます。

### マルチアカウントの例

マルチアカウントチャンネル（Telegram等）で特定のアカウントをターゲットにするには`accountId`を使用します:

```json5
{
  agents: {
    list: [
      {
        id: "ops",
        heartbeat: {
          every: "1h",
          target: "telegram",
          to: "12345678:topic:42", // オプション: 特定のトピック/スレッドにルーティング
          accountId: "ops-bot",
        },
      },
    ],
  },
  channels: {
    telegram: {
      accounts: {
        "ops-bot": { botToken: "YOUR_TELEGRAM_BOT_TOKEN" },
      },
    },
  },
}
```

### フィールドノート

- `every`: ハートビート間隔（期間文字列、デフォルト単位 = 分）。
- `model`: ハートビート実行用のオプションのモデルオーバーライド（`provider/model`）。
- `includeReasoning`: 有効時、利用可能な場合に別の`Reasoning:`メッセージも配信（`/reasoning on`と同じ形式）。
- `lightContext`: trueの場合、ハートビート実行は軽量ブートストラップコンテキストを使用し、ワークスペースブートストラップファイルから`HEARTBEAT.md`のみを保持。
- `session`: ハートビート実行用のオプションのセッションキー。
  - `main`（デフォルト）: エージェントのメインセッション。
  - 明示的なセッションキー（`openclaw sessions --json`または[セッションCLI](/cli/sessions)からコピー）。
  - セッションキーの形式: [セッション](/concepts/session)および[グループ](/channels/groups)を参照。
- `target`:
  - `last`: 最後に使用した外部チャンネルに配信。
  - 明示的なチャンネル: `whatsapp` / `telegram` / `discord` / `googlechat` / `slack` / `msteams` / `signal` / `imessage`。
  - `none`（デフォルト）: ハートビートを実行するが外部には配信**しない**。
- `directPolicy`: direct/DM配信動作を制御:
  - `allow`（デフォルト）: direct/DMハートビート配信を許可。
  - `block`: direct/DM配信を抑制（`reason=dm-blocked`）。
- `to`: オプションの受信者オーバーライド（チャンネル固有のID、例: WhatsAppのE.164またはTelegramのチャットID）。Telegramのトピック/スレッドには`<chatId>:topic:<messageThreadId>`を使用。
- `accountId`: マルチアカウントチャンネル用のオプションのアカウントID。`target: "last"`の場合、解決された最後のチャンネルがアカウントをサポートする場合にアカウントIDが適用されます。それ以外の場合は無視されます。アカウントIDが解決されたチャンネルの設定済みアカウントと一致しない場合、配信はスキップされます。
- `prompt`: デフォルトのプロンプト本文をオーバーライド（マージではない）。
- `ackMaxChars`: 配信前に`HEARTBEAT_OK`後に許可される最大文字数。
- `suppressToolErrorWarnings`: trueの場合、ハートビート実行中のツールエラー警告ペイロードを抑制。
- `activeHours`: ハートビート実行を時間ウィンドウに制限。`start`（HH:MM、包含、日の開始には`00:00`を使用）、`end`（HH:MM、排他、日の終わりには`24:00`が許可）、およびオプションの`timezone`を持つオブジェクト。
  - 省略または`"user"`: `agents.defaults.userTimezone`が設定されている場合はそれを使用、それ以外はホストシステムのタイムゾーンにフォールバック。
  - `"local"`: 常にホストシステムのタイムゾーンを使用。
  - 任意のIANA識別子（例: `America/New_York`）: 直接使用、無効な場合は上記の`"user"`動作にフォールバック。
  - `start`と`end`はアクティブウィンドウで等しくてはなりません。等しい値はゼロ幅として扱われます（常にウィンドウ外）。
  - アクティブウィンドウ外では、ウィンドウ内の次のティックまでハートビートがスキップされます。

## 配信動作

- ハートビートはデフォルトでエージェントのメインセッション（`agent:<id>:<mainKey>`）で実行されるか、
  `session.scope = "global"`の場合は`global`で実行されます。特定のチャンネルセッション（Discord/WhatsApp等）にオーバーライドするには`session`を設定します。
- `session`は実行コンテキストにのみ影響します。配信は`target`と`to`で制御されます。
- 特定のチャンネル/受信者に配信するには、`target` + `to`を設定します。`target: "last"`の場合、そのセッションの最後の外部チャンネルを使用して配信します。
- ハートビート配信はデフォルトでdirect/DMターゲットを許可します。`directPolicy: "block"`を設定して、ハートビートターンは実行しつつdirectターゲットへの送信を抑制します。
- メインキューがビジーの場合、ハートビートはスキップされ後で再試行されます。
- `target`が外部の宛先に解決されない場合でも実行は行われますが、
  送信メッセージは送信されません。
- ハートビートのみの返信はセッションをアクティブに保ち**ません**。最後の`updatedAt`が
  復元されるため、アイドル期限切れは通常通り動作します。

## 可視性コントロール

デフォルトでは、`HEARTBEAT_OK`の確認応答は抑制され、アラートコンテンツは
配信されます。チャンネルごとまたはアカウントごとに調整できます:

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false # HEARTBEAT_OKを非表示（デフォルト）
      showAlerts: true # アラートメッセージを表示（デフォルト）
      useIndicator: true # インジケーターイベントを発行（デフォルト）
  telegram:
    heartbeat:
      showOk: true # TelegramでOK確認応答を表示
  whatsapp:
    accounts:
      work:
        heartbeat:
          showAlerts: false # このアカウントのアラート配信を抑制
```

優先順位: アカウントごと → チャンネルごと → チャンネルデフォルト → 組み込みデフォルト。

### 各フラグの機能

- `showOk`: モデルがOKのみの返信を返した場合に`HEARTBEAT_OK`確認応答を送信。
- `showAlerts`: モデルが非OKの返信を返した場合にアラートコンテンツを送信。
- `useIndicator`: UIステータスサーフェス用のインジケーターイベントを発行。

**3つすべて**がfalseの場合、OpenClawはハートビート実行を完全にスキップします（モデル呼び出しなし）。

### チャンネルごと vs アカウントごとの例

```yaml
channels:
  defaults:
    heartbeat:
      showOk: false
      showAlerts: true
      useIndicator: true
  slack:
    heartbeat:
      showOk: true # すべてのSlackアカウント
    accounts:
      ops:
        heartbeat:
          showAlerts: false # opsアカウントのみアラートを抑制
  telegram:
    heartbeat:
      showOk: true
```

### 一般的なパターン

| 目的 | 設定 |
| ---------------------------------------- | ---------------------------------------------------------------------------------------- |
| デフォルト動作（サイレントOK、アラートオン） | _（設定不要）_ |
| 完全サイレント（メッセージなし、インジケーターなし） | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: false }` |
| インジケーターのみ（メッセージなし） | `channels.defaults.heartbeat: { showOk: false, showAlerts: false, useIndicator: true }` |
| 1つのチャンネルのみOK表示 | `channels.telegram.heartbeat: { showOk: true }` |

## HEARTBEAT.md（オプション）

`HEARTBEAT.md`ファイルがワークスペースに存在する場合、デフォルトのプロンプトは
エージェントにそれを読むよう指示します。これを「ハートビートチェックリスト」と考えてください:
小さく、安定しており、30分ごとに含めても安全です。

`HEARTBEAT.md`が存在するが実質的に空の場合（空行とマークダウンヘッダー
（`# Heading`など）のみ）、OpenClawはAPI呼び出しを節約するためにハートビート実行をスキップします。
ファイルが存在しない場合、ハートビートは引き続き実行され、モデルが何をするか決定します。

プロンプトの肥大化を避けるため、小さく保ちます（短いチェックリストやリマインダー）。

`HEARTBEAT.md`の例:

```md
# ハートビートチェックリスト

- クイックスキャン: 受信トレイに緊急のものはありますか？
- 日中であれば、他に保留中のことがなければ軽量なチェックインを行います。
- タスクがブロックされている場合、_何が不足しているか_を書き留め、次回Peterに聞きます。
```

### エージェントはHEARTBEAT.mdを更新できますか？

はい — 依頼すれば可能です。

`HEARTBEAT.md`はエージェントワークスペース内の通常のファイルなので、通常のチャットで
エージェントに以下のように指示できます:

- 「`HEARTBEAT.md`を更新して、毎日のカレンダーチェックを追加して。」
- 「`HEARTBEAT.md`を書き直して、受信トレイのフォローアップに焦点を当てた短いものにして。」

これを能動的に行いたい場合は、ハートビートプロンプトに明示的な行を含めることもできます:
「チェックリストが古くなったら、HEARTBEAT.mdをより良いものに更新して。」

安全に関する注意: `HEARTBEAT.md`にシークレット（APIキー、電話番号、プライベートトークン）を
入れないでください — プロンプトコンテキストの一部になります。

## 手動ウェイク（オンデマンド）

システムイベントをキューに入れ、即時ハートビートをトリガーできます:

```bash
openclaw system event --text "緊急フォローアップを確認" --mode now
```

複数のエージェントに`heartbeat`が設定されている場合、手動ウェイクはそれらの
エージェントハートビートをすべて即座に実行します。

`--mode next-heartbeat`で次のスケジュールされたティックを待ちます。

## 推論配信（オプション）

デフォルトでは、ハートビートは最終的な「回答」ペイロードのみを配信します。

透明性が必要な場合は、以下を有効にします:

- `agents.defaults.heartbeat.includeReasoning: true`

有効時、ハートビートは`Reasoning:`プレフィックス付きの別のメッセージも配信します
（`/reasoning on`と同じ形式）。これはエージェントが複数のセッション/コーデックスを
管理している場合に、なぜpingしたのかを確認するのに便利ですが、
望む以上の内部詳細が漏れる可能性もあります。グループチャットではオフにしておくことを推奨します。

## コスト意識

ハートビートは完全なエージェントターンを実行します。短い間隔ほどトークンを消費します。
`HEARTBEAT.md`を小さく保ち、内部状態の更新のみが必要な場合は
より安価な`model`や`target: "none"`を検討してください。
