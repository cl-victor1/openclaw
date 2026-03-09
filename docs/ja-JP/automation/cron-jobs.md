---
summary: "Gatewayスケジューラーのcronジョブとウェイクアップ"
read_when:
  - バックグラウンドジョブやウェイクアップのスケジューリング
  - ハートビートと連携して実行される自動化の設定
  - スケジュールされたタスクにハートビートとcronのどちらを使うか決める場合
title: "Cronジョブ"
---

# cronジョブ（Gatewayスケジューラー）

> **Cron vs Heartbeat?** どちらを使うべきかは [Cron vs Heartbeat](/automation/cron-vs-heartbeat) を参照してください。

CronはGatewayの組み込みスケジューラーです。ジョブを永続化し、適切なタイミングでエージェントを起動し、オプションで出力をチャットに送り返すことができます。

_「毎朝これを実行したい」_ や _「20分後にエージェントを呼び出したい」_ という場合に、cronが適したメカニズムです。

トラブルシューティング: [/automation/troubleshooting](/automation/troubleshooting)

## TL;DR

- cronは**Gateway内部**で動作します（モデルの中ではありません）。
- ジョブは `~/.openclaw/cron/` に永続化されるため、再起動してもスケジュールが失われません。
- 2つの実行スタイル:
  - **メインセッション**: システムイベントをエンキューし、次のハートビートで実行。
  - **分離**: `cron:<jobId>` で専用のエージェントターンを実行（デフォルトでannounce、またはnone）。
- ウェイクアップはファーストクラス: ジョブは「今すぐウェイク」か「次のハートビート」を指定できます。
- Webhookへの投稿はジョブごとに `delivery.mode = "webhook"` + `delivery.to = "<url>"` で設定します。
- `notify: true` を持つ既存のジョブには `cron.webhook` へのレガシーフォールバックが残っています。webhookデリバリーモードに移行してください。

## クイックスタート（実践的）

ワンショットリマインダーを作成し、存在を確認して、即座に実行:

```bash
openclaw cron add \
  --name "Reminder" \
  --at "2026-02-01T16:00:00Z" \
  --session main \
  --system-event "Reminder: check the cron docs draft" \
  --wake now \
  --delete-after-run

openclaw cron list
openclaw cron run <job-id>
openclaw cron runs --id <job-id>
```

デリバリー付きの繰り返し分離ジョブのスケジュール:

```bash
openclaw cron add \
  --name "Morning brief" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize overnight updates." \
  --announce \
  --channel slack \
  --to "channel:C1234567890"
```

## ツール呼び出しの対応（Gateway cronツール）

正式なJSON形式と例については、[ツール呼び出しのJSONスキーマ](/automation/cron-jobs#json-schema-for-tool-calls) を参照してください。

## cronジョブの保存場所

cronジョブはGatewayホスト上の `~/.openclaw/cron/jobs.json` に永続化されます（デフォルト）。
Gatewayはファイルをメモリに読み込み、変更時に書き戻します。そのため手動編集はGatewayが停止しているときのみ安全です。変更には `openclaw cron add/edit` またはcronツール呼び出しAPIを使用してください。

## 初心者向け概要

cronジョブは次のように考えてください: **いつ**実行するか + **何をする**か。

1. **スケジュールを選ぶ**
   - ワンショットリマインダー → `schedule.kind = "at"`（CLI: `--at`）
   - 繰り返しジョブ → `schedule.kind = "every"` または `schedule.kind = "cron"`
   - ISOタイムスタンプにタイムゾーンを省略した場合、**UTC**として扱われます。

2. **実行場所を選ぶ**
   - `sessionTarget: "main"` → メインコンテキストで次のハートビート時に実行。
   - `sessionTarget: "isolated"` → `cron:<jobId>` で専用のエージェントターンを実行。

3. **ペイロードを選ぶ**
   - メインセッション → `payload.kind = "systemEvent"`
   - 分離セッション → `payload.kind = "agentTurn"`

オプション: ワンショットジョブ（`schedule.kind = "at"`）は成功後にデフォルトで削除されます。保持するには `deleteAfterRun: false` を設定します（成功後に無効化されます）。

## コンセプト

### ジョブ

cronジョブは以下を持つ保存されたレコードです:

- **スケジュール**（いつ実行するか）、
- **ペイロード**（何をするか）、
- オプションの**デリバリーモード**（`announce`、`webhook`、または `none`）。
- オプションの**エージェントバインディング**（`agentId`）: 特定のエージェントの下でジョブを実行します。不明な場合はデフォルトエージェントにフォールバックします。

ジョブは安定した `jobId` で識別されます（CLI/Gateway APIで使用）。
エージェントツール呼び出しでは `jobId` が正式です。互換性のためにレガシーの `id` も受け付けます。
ワンショットジョブは成功後にデフォルトで自動削除されます。保持するには `deleteAfterRun: false` を設定してください。

### スケジュール

Cronは3種類のスケジュールをサポートします:

- `at`: `schedule.at`（ISO 8601）による一回限りのタイムスタンプ。
- `every`: 固定インターバル（ミリ秒）。
- `cron`: オプションのIANAタイムゾーン付き5フィールドcron式（または秒付き6フィールド）。

Cron式は `croner` を使用します。タイムゾーンが省略された場合、Gatewayホストのローカルタイムゾーンが使用されます。

多数のGateway間での毎時トップのロードスパイクを軽減するため、OpenClawは繰り返しの毎時トップ式（例: `0 * * * *`、`0 */2 * * *`）に最大5分の決定論的なジョブごとのスタガーウィンドウを適用します。`0 7 * * *` のような固定時刻の式は正確なまま維持されます。

任意のcronスケジュールに対して、`schedule.staggerMs` で明示的なスタガーウィンドウを設定できます（`0` は正確なタイミングを維持）。CLI省略形:

- `--stagger 30s`（または `1m`、`5m`）で明示的なスタガーウィンドウを設定。
- `--exact` で `staggerMs = 0` を強制。

### メインセッション vs 分離実行

#### メインセッションジョブ（システムイベント）

メインジョブはシステムイベントをエンキューし、オプションでハートビートランナーをウェイクします。
`payload.kind = "systemEvent"` を使用する必要があります。

- `wakeMode: "now"`（デフォルト）: イベントが即時ハートビート実行をトリガー。
- `wakeMode: "next-heartbeat"`: イベントが次のスケジュールされたハートビートを待機。

通常のハートビートプロンプト + メインセッションコンテキストが必要な場合に最適です。
[Heartbeat](/gateway/heartbeat) を参照してください。

#### 分離ジョブ（専用cronセッション）

分離ジョブはセッション `cron:<jobId>` で専用のエージェントターンを実行します。

主な動作:

- プロンプトは追跡可能性のために `[cron:<jobId> <job name>]` でプレフィックスされます。
- 各実行は**新しいセッションID**で開始します（前の会話の持ち越しなし）。
- デフォルト動作: `delivery` が省略された場合、分離ジョブはサマリーをannounceします（`delivery.mode = "announce"`）。
- `delivery.mode` で何が起こるかを選択:
  - `announce`: ターゲットチャンネルにサマリーを配信し、メインセッションに簡単なサマリーを投稿。
  - `webhook`: finishedイベントにサマリーが含まれる場合、`delivery.to` にfinishedイベントペイロードをPOST。
  - `none`: 内部のみ（デリバリーなし、メインセッションサマリーなし）。
- `wakeMode` はメインセッションサマリーがいつ投稿されるかを制御:
  - `now`: 即時ハートビート。
  - `next-heartbeat`: 次のスケジュールされたハートビートを待機。

メインチャット履歴をスパムしてはいけないノイジーで頻繁な「バックグラウンドタスク」に分離ジョブを使用してください。

### ペイロードの形（何が実行されるか）

2種類のペイロードがサポートされています:

- `systemEvent`: メインセッションのみ、ハートビートプロンプトを通じてルーティング。
- `agentTurn`: 分離セッションのみ、専用のエージェントターンを実行。

一般的な `agentTurn` フィールド:

- `message`: 必須のテキストプロンプト。
- `model` / `thinking`: オプションのオーバーライド（以下参照）。
- `timeoutSeconds`: オプションのタイムアウトオーバーライド。
- `lightContext`: ワークスペースブートストラップファイルの注入が不要なジョブ向けのオプションの軽量ブートストラップモード。

デリバリー設定:

- `delivery.mode`: `none` | `announce` | `webhook`。
- `delivery.channel`: `last` または特定のチャンネル。
- `delivery.to`: チャンネル固有のターゲット（announce）またはwebhook URL（webhookモード）。
- `delivery.bestEffort`: announceデリバリーが失敗してもジョブを失敗させない。

Announceデリバリーは実行中のメッセージングツールの送信を抑制します。代わりにチャットをターゲットにするには `delivery.channel`/`delivery.to` を使用してください。`delivery.mode = "none"` の場合、メインセッションにサマリーは投稿されません。

分離ジョブで `delivery` が省略された場合、OpenClawはデフォルトで `announce` になります。

#### Announceデリバリーフロー

`delivery.mode = "announce"` の場合、cronはアウトバウンドチャンネルアダプターを通じて直接デリバリーします。
メインエージェントはメッセージを作成または転送するために起動されません。

動作の詳細:

- コンテンツ: デリバリーは通常のチャンキングとチャンネルフォーマットで、分離実行のアウトバウンドペイロード（テキスト/メディア）を使用します。
- ハートビートのみの応答（実際のコンテンツのない `HEARTBEAT_OK`）は配信されません。
- 分離実行がmessageツールを通じて同じターゲットにすでにメッセージを送信した場合、重複を避けるためデリバリーはスキップされます。
- 存在しないまたは無効なデリバリーターゲットは、`delivery.bestEffort = true` でない限りジョブを失敗させます。
- メインセッションへの短いサマリーは `delivery.mode = "announce"` の場合のみ投稿されます。
- メインセッションサマリーは `wakeMode` を尊重します: `now` は即時ハートビートをトリガーし、`next-heartbeat` は次のスケジュールされたハートビートを待機します。

#### Webhookデリバリーフロー

`delivery.mode = "webhook"` の場合、finishedイベントにサマリーが含まれる場合、cronはfinishedイベントペイロードを `delivery.to` にPOSTします。

動作の詳細:

- エンドポイントは有効なHTTP(S) URLである必要があります。
- webhookモードではチャンネルデリバリーは試みられません。
- webhookモードではメインセッションサマリーは投稿されません。
- `cron.webhookToken` が設定されている場合、authヘッダーは `Authorization: Bearer <cron.webhookToken>` です。
- 非推奨フォールバック: `notify: true` を持つ保存されたレガシージョブは、設定されている場合まだ `cron.webhook` にPOSTします（警告付きで `delivery.mode = "webhook"` に移行できます）。

### モデルとthinkingのオーバーライド

分離ジョブ（`agentTurn`）はモデルとthinkingレベルをオーバーライドできます:

- `model`: プロバイダー/モデル文字列（例: `anthropic/claude-sonnet-4-20250514`）またはエイリアス（例: `opus`）
- `thinking`: Thinkingレベル（`off`、`minimal`、`low`、`medium`、`high`、`xhigh`; GPT-5.2 + Codexモデルのみ）

注: メインセッションジョブにも `model` を設定できますが、共有メインセッションモデルが変更されます。予期しないコンテキストのシフトを避けるため、分離ジョブのみでモデルオーバーライドを使用することをお勧めします。

解決優先度:

1. ジョブペイロードオーバーライド（最高）
2. フック固有のデフォルト（例: `hooks.gmail.model`）
3. エージェント設定のデフォルト

### 軽量ブートストラップコンテキスト

分離ジョブ（`agentTurn`）は `lightContext: true` を設定して軽量ブートストラップコンテキストで実行できます。

- ワークスペースブートストラップファイルの注入が不要なスケジュールされたタスクに使用してください。
- 実際には、組み込みランタイムは `bootstrapContextMode: "lightweight"` で実行され、cronブートストラップコンテキストを意図的に空に保ちます。
- CLI対応: `openclaw cron add --light-context ...` と `openclaw cron edit --light-context`。

### デリバリー（チャンネル + ターゲット）

分離ジョブはトップレベルの `delivery` 設定を通じてチャンネルに出力を配信できます:

- `delivery.mode`: `announce`（チャンネルデリバリー）、`webhook`（HTTP POST）、または `none`。
- `delivery.channel`: `whatsapp` / `telegram` / `discord` / `slack` / `mattermost`（プラグイン）/ `signal` / `imessage` / `last`。
- `delivery.to`: チャンネル固有の受信者ターゲット。

`announce` デリバリーは分離ジョブ（`sessionTarget: "isolated"`）に対してのみ有効です。
`webhook` デリバリーはメインジョブと分離ジョブの両方に対して有効です。

`delivery.channel` または `delivery.to` が省略された場合、cronはメインセッションの「ラストルート」（エージェントが最後に返信した場所）にフォールバックできます。

ターゲット形式のリマインダー:

- Slack/Discord/Mattermost（プラグイン）ターゲットは曖昧さを避けるために明示的なプレフィックスを使用してください（例: `channel:<id>`、`user:<id>`）。
- Telegramトピックは `:topic:` 形式を使用してください（以下参照）。

#### Telegramデリバリーターゲット（トピック/フォームスレッド）

Telegramは `message_thread_id` を通じてフォーラムトピックをサポートします。cronデリバリーでは、トピック/スレッドを `to` フィールドにエンコードできます:

- `-1001234567890`（チャットIDのみ）
- `-1001234567890:topic:123`（推奨: 明示的なトピックマーカー）
- `-1001234567890:123`（省略形: 数値サフィックス）

`telegram:...` / `telegram:group:...` のようなプレフィックス付きターゲットも受け付けます:

- `telegram:group:-1001234567890:topic:123`

## ツール呼び出しのJSONスキーマ

GatewayのCronツール（エージェントツール呼び出しまたはRPC）を直接呼び出す際にはこれらの形式を使用してください。
CLIフラグは `20m` のような人間が読みやすい期間を受け付けますが、ツール呼び出しでは `schedule.at` にISO 8601文字列、`schedule.everyMs` にミリ秒を使用してください。

### cron.addのパラメーター

ワンショット、メインセッションジョブ（システムイベント）:

```json
{
  "name": "Reminder",
  "schedule": { "kind": "at", "at": "2026-02-01T16:00:00Z" },
  "sessionTarget": "main",
  "wakeMode": "now",
  "payload": { "kind": "systemEvent", "text": "Reminder text" },
  "deleteAfterRun": true
}
```

デリバリー付きの繰り返し分離ジョブ:

```json
{
  "name": "Morning brief",
  "schedule": { "kind": "cron", "expr": "0 7 * * *", "tz": "America/Los_Angeles" },
  "sessionTarget": "isolated",
  "wakeMode": "next-heartbeat",
  "payload": {
    "kind": "agentTurn",
    "message": "Summarize overnight updates.",
    "lightContext": true
  },
  "delivery": {
    "mode": "announce",
    "channel": "slack",
    "to": "channel:C1234567890",
    "bestEffort": true
  }
}
```

注:

- `schedule.kind`: `at`（`at`）、`every`（`everyMs`）、または `cron`（`expr`、オプション `tz`）。
- `schedule.at` はISO 8601を受け付けます（タイムゾーンはオプション。省略時はUTCとして扱われます）。
- `everyMs` はミリ秒です。
- `sessionTarget` は `"main"` または `"isolated"` で、`payload.kind` と一致している必要があります。
- オプションフィールド: `agentId`、`description`、`enabled`、`deleteAfterRun`（`at` のデフォルトはtrue）、`delivery`。
- `wakeMode` は省略時のデフォルトは `"now"` です。

### cron.updateのパラメーター

```json
{
  "jobId": "job-123",
  "patch": {
    "enabled": false,
    "schedule": { "kind": "every", "everyMs": 3600000 }
  }
}
```

注:

- `jobId` が正式です。互換性のために `id` も受け付けます。
- パッチで `agentId: null` を使用してエージェントバインディングをクリアします。

### cron.runとcron.removeのパラメーター

```json
{ "jobId": "job-123", "mode": "force" }
```

```json
{ "jobId": "job-123" }
```

## ストレージと履歴

- ジョブストア: `~/.openclaw/cron/jobs.json`（Gateway管理のJSON）。
- 実行履歴: `~/.openclaw/cron/runs/<jobId>.jsonl`（JSONL、サイズと行数で自動プルーニング）。
- `sessions.json` の分離cronセッションは `cron.sessionRetention` によってプルーニングされます（デフォルト `24h`; `false` で無効化）。
- ストアパスのオーバーライド: 設定の `cron.store`。

## リトライポリシー

ジョブが失敗した場合、OpenClawはエラーを**一時的**（リトライ可能）か**永続的**（即時無効化）かに分類します。

### 一時的なエラー（リトライされる）

- レートリミット（429、リクエスト過多、リソース枯渇）
- プロバイダーの過負荷（例: Anthropicの `529 overloaded_error`、過負荷フォールバックサマリー）
- ネットワークエラー（タイムアウト、ECONNRESET、fetch失敗、ソケット）
- サーバーエラー（5xx）
- Cloudflare関連のエラー

### 永続的なエラー（リトライなし）

- 認証失敗（無効なAPIキー、未認証）
- 設定または検証エラー
- その他の非一時的なエラー

### デフォルト動作（設定なし）

**ワンショットジョブ（`schedule.kind: "at"`）:**

- 一時的なエラー時: 指数バックオフで最大3回リトライ（30秒 → 1分 → 5分）。
- 永続的なエラー時: 即時無効化。
- 成功またはスキップ時: 無効化（`deleteAfterRun: true` の場合は削除）。

**繰り返しジョブ（`cron` / `every`）:**

- エラー時: 次のスケジュール実行前に指数バックオフを適用（30秒 → 1分 → 5分 → 15分 → 60分）。
- ジョブは有効のまま; バックオフは次の成功した実行後にリセット。

これらのデフォルトをオーバーライドするには `cron.retry` を設定してください（[設定](/automation/cron-jobs#configuration) 参照）。

## 設定

```json5
{
  cron: {
    enabled: true, // デフォルトtrue
    store: "~/.openclaw/cron/jobs.json",
    maxConcurrentRuns: 1, // デフォルト1
    // オプション: ワンショットジョブのリトライポリシーをオーバーライド
    retry: {
      maxAttempts: 3,
      backoffMs: [60000, 120000, 300000],
      retryOn: ["rate_limit", "overloaded", "network", "server_error"],
    },
    webhook: "https://example.invalid/legacy", // notify:trueの保存済みジョブ向け非推奨フォールバック
    webhookToken: "replace-with-dedicated-webhook-token", // webhookモードのオプションbearerトークン
    sessionRetention: "24h", // 期間文字列またはfalse
    runLog: {
      maxBytes: "2mb", // デフォルト2_000_000バイト
      keepLines: 2000, // デフォルト2000
    },
  },
}
```

実行ログのプルーニング動作:

- `cron.runLog.maxBytes`: プルーニング前の最大実行ログファイルサイズ。
- `cron.runLog.keepLines`: プルーニング時、最新のN行のみ保持。
- 両方とも `cron/runs/<jobId>.jsonl` ファイルに適用されます。

Webhook動作:

- 推奨: ジョブごとに `delivery.mode: "webhook"` と `delivery.to: "https://..."` を設定。
- Webhook URLは有効な `http://` または `https://` URLである必要があります。
- 投稿時、ペイロードはcronのfinishedイベントJSONです。
- `cron.webhookToken` が設定されている場合、authヘッダーは `Authorization: Bearer <cron.webhookToken>` です。
- `cron.webhookToken` が設定されていない場合、`Authorization` ヘッダーは送信されません。
- 非推奨フォールバック: `notify: true` を持つ保存されたレガシージョブは、存在する場合まだ `cron.webhook` を使用します。

cronを完全に無効化:

- `cron.enabled: false`（設定）
- `OPENCLAW_SKIP_CRON=1`（環境変数）

## メンテナンス

Cronには2つの組み込みメンテナンスパスがあります: 分離実行セッションの保持と実行ログのプルーニング。

### デフォルト値

- `cron.sessionRetention`: `24h`（実行セッションプルーニングを無効にするには `false`）
- `cron.runLog.maxBytes`: `2_000_000` バイト
- `cron.runLog.keepLines`: `2000`

### 仕組み

- 分離実行はセッションエントリ（`...:cron:<jobId>:run:<uuid>`）とトランスクリプトファイルを作成します。
- リーパーは `cron.sessionRetention` より古くなった期限切れの実行セッションエントリを削除します。
- 削除された実行セッションでセッションストアに参照されなくなったものについて、OpenClawはトランスクリプトファイルをアーカイブし、同じ保持ウィンドウで古い削除済みアーカイブをパージします。
- 各実行追加後、`cron/runs/<jobId>.jsonl` のサイズがチェックされます:
  - ファイルサイズが `runLog.maxBytes` を超えた場合、最新の `runLog.keepLines` 行にトリミングされます。

### 大量スケジューラーのパフォーマンス上の注意

高頻度のcronセットアップは大きな実行セッションと実行ログのフットプリントを生成する可能性があります。メンテナンスは組み込まれていますが、緩い制限は回避可能なIOとクリーンアップ作業を引き起こす可能性があります。

注意すべき点:

- 多くの分離実行を伴う長い `cron.sessionRetention` ウィンドウ
- 大きな `runLog.maxBytes` と組み合わせた高い `cron.runLog.keepLines`
- 同じ `cron/runs/<jobId>.jsonl` に書き込む多くのノイジーな繰り返しジョブ

対策:

- デバッグ/監査ニーズが許す限り `cron.sessionRetention` を短く保つ
- 適度な `runLog.maxBytes` と `runLog.keepLines` で実行ログを制限する
- ノイジーなバックグラウンドジョブは不要なチャットを避けるデリバリールール付きの分離モードに移行する
- `openclaw cron runs` で定期的に成長を確認し、ログが大きくなる前に保持を調整する

### カスタマイズ例

実行セッションを1週間保持し、より大きな実行ログを許可:

```json5
{
  cron: {
    sessionRetention: "7d",
    runLog: {
      maxBytes: "10mb",
      keepLines: 5000,
    },
  },
}
```

分離実行セッションのプルーニングを無効化し、実行ログのプルーニングを保持:

```json5
{
  cron: {
    sessionRetention: false,
    runLog: {
      maxBytes: "5mb",
      keepLines: 3000,
    },
  },
}
```

大量cron使用のためのチューニング（例）:

```json5
{
  cron: {
    sessionRetention: "12h",
    runLog: {
      maxBytes: "3mb",
      keepLines: 1500,
    },
  },
}
```

## CLIクイックスタート

ワンショットリマインダー（UTC ISO、成功後に自動削除）:

```bash
openclaw cron add \
  --name "Send reminder" \
  --at "2026-01-12T18:00:00Z" \
  --session main \
  --system-event "Reminder: submit expense report." \
  --wake now \
  --delete-after-run
```

ワンショットリマインダー（メインセッション、即時ウェイク）:

```bash
openclaw cron add \
  --name "Calendar check" \
  --at "20m" \
  --session main \
  --system-event "Next heartbeat: check calendar." \
  --wake now
```

繰り返し分離ジョブ（WhatsAppにannounce）:

```bash
openclaw cron add \
  --name "Morning status" \
  --cron "0 7 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize inbox + calendar for today." \
  --announce \
  --channel whatsapp \
  --to "+15551234567"
```

明示的な30秒スタガー付きの繰り返しcronジョブ:

```bash
openclaw cron add \
  --name "Minute watcher" \
  --cron "0 * * * * *" \
  --tz "UTC" \
  --stagger 30s \
  --session isolated \
  --message "Run minute watcher checks." \
  --announce
```

繰り返し分離ジョブ（Telegramトピックに配信）:

```bash
openclaw cron add \
  --name "Nightly summary (topic)" \
  --cron "0 22 * * *" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Summarize today; send to the nightly topic." \
  --announce \
  --channel telegram \
  --to "-1001234567890:topic:123"
```

モデルとthinkingオーバーライド付きの分離ジョブ:

```bash
openclaw cron add \
  --name "Deep analysis" \
  --cron "0 6 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Weekly deep analysis of project progress." \
  --model "opus" \
  --thinking high \
  --announce \
  --channel whatsapp \
  --to "+15551234567"
```

エージェント選択（マルチエージェントセットアップ）:

```bash
# ジョブをエージェント "ops" に固定する（エージェントが見つからない場合はデフォルトにフォールバック）
openclaw cron add --name "Ops sweep" --cron "0 6 * * *" --session isolated --message "Check ops queue" --agent ops

# 既存のジョブのエージェントを切り替えまたはクリア
openclaw cron edit <jobId> --agent ops
openclaw cron edit <jobId> --clear-agent
```

手動実行（デフォルトはforce、`--due` で期限が来た場合のみ実行）:

```bash
openclaw cron run <jobId>
openclaw cron run <jobId> --due
```

`cron.run` はジョブが終了した後ではなく、手動実行がキューに入れられた時点で確認応答するようになりました。成功したキュー応答は `{ ok: true, enqueued: true, runId }` のようになります。ジョブがすでに実行中または `--due` で期限が来たものが見つからない場合、応答は `{ ok: true, ran: false, reason }` のままです。最終的なfinishedエントリを確認するには `openclaw cron runs --id <jobId>` または `cron.runs` Gatewayメソッドを使用してください。

既存のジョブを編集（フィールドをパッチ）:

```bash
openclaw cron edit <jobId> \
  --message "Updated prompt" \
  --model "opus" \
  --thinking low
```

既存のcronジョブをスケジュール通りに正確に実行させる（スタガーなし）:

```bash
openclaw cron edit <jobId> --exact
```

実行履歴:

```bash
openclaw cron runs --id <jobId> --limit 50
```

ジョブを作成せずに即時システムイベント:

```bash
openclaw system event --mode now --text "Next heartbeat: check battery."
```

## Gateway APIサーフェス

- `cron.list`、`cron.status`、`cron.add`、`cron.update`、`cron.remove`
- `cron.run`（forceまたはdue）、`cron.runs`
  ジョブなしの即時システムイベントには [`openclaw system event`](/cli/system) を使用してください。

## トラブルシューティング

### 「何も実行されない」

- cronが有効になっているか確認: `cron.enabled` と `OPENCLAW_SKIP_CRON`。
- Gatewayが継続的に実行されているか確認（cronはGatewayプロセス内で実行されます）。
- `cron` スケジュールの場合: タイムゾーン（`--tz`）とホストタイムゾーンを確認。

### 繰り返しジョブが失敗後に遅延し続ける

- OpenClawは繰り返しジョブが連続してエラーになった後、指数リトライバックオフを適用します: 30秒、1分、5分、15分、その後60分のリトライ間隔。
- バックオフは次の成功した実行後に自動的にリセットされます。
- ワンショット（`at`）ジョブは一時的なエラー（レートリミット、過負荷、ネットワーク、server_error）を最大3回バックオフでリトライし、永続的なエラーは即時無効化します。[リトライポリシー](/automation/cron-jobs#retry-policy) を参照してください。

### Telegramが間違った場所に配信される

- フォーラムトピックには `-100…:topic:<id>` を使用して、明示的で曖昧さのない形式にしてください。
- ログや保存された「ラストルート」ターゲットに `telegram:...` プレフィックスが表示される場合、それは正常です。cronデリバリーはそれらを受け付け、トピックIDを正しく解析します。

### サブエージェントannounceデリバリーのリトライ

- サブエージェント実行が完了すると、Gatewayは結果をリクエスターセッションにannounceします。
- announceフローが `false` を返した場合（例: リクエスターセッションがビジー）、Gatewayは `announceRetryCount` を通じて追跡しながら最大3回リトライします。
- `endedAt` から5分以上経過した古いAnnounceは、古いエントリが無限にループするのを防ぐために強制期限切れにされます。
- ログで繰り返しannounceデリバリーが表示される場合、高い `announceRetryCount` 値を持つエントリのサブエージェントレジストリを確認してください。
