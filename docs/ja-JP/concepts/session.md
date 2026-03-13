---
summary: "セッション管理のルール、キー、チャットの永続化"
read_when:
  - Modifying session handling or storage
title: "セッション管理"
---

# セッション管理

OpenClawは**エージェントごとに1つのダイレクトチャットセッション**をプライマリとして扱います。ダイレクトチャットは`agent:<agentId>:<mainKey>`（デフォルト`main`）に集約され、グループ/チャネルチャットは独自のキーを取得します。`session.mainKey`は尊重されます。

`session.dmScope`を使用して**ダイレクトメッセージ**のグループ化方法を制御します：

- `main`（デフォルト）：すべてのDMが連続性のためにメインセッションを共有。
- `per-peer`：チャネルをまたいで送信者IDで分離。
- `per-channel-peer`：チャネル＋送信者で分離（マルチユーザーインボックスに推奨）。
- `per-account-channel-peer`：アカウント＋チャネル＋送信者で分離（マルチアカウントインボックスに推奨）。
  `session.identityLinks`を使用して、プロバイダープレフィックス付きのピアIDを正規IDにマッピングし、`per-peer`、`per-channel-peer`、または`per-account-channel-peer`使用時に同じ人物がチャネル間でDMセッションを共有できるようにします。

## セキュアDMモード（マルチユーザー設定に推奨）

> **セキュリティ警告：** エージェントが**複数の人**からDMを受信できる場合、セキュアDMモードの有効化を強く検討してください。これがないと、すべてのユーザーが同じ会話コンテキストを共有し、ユーザー間でプライベート情報が漏洩する可能性があります。

**デフォルト設定での問題の例：**

- Alice（`<SENDER_A>`）がプライベートなトピック（例：医療の予約）についてエージェントにメッセージを送る
- Bob（`<SENDER_B>`）がエージェントに「何について話していましたか？」と質問する
- 両方のDMが同じセッションを共有しているため、モデルがAliceの以前のコンテキストを使ってBobに回答する可能性がある

**修正方法：** `dmScope`を設定してユーザーごとにセッションを分離します：

```json5
// ~/.openclaw/openclaw.json
{
  session: {
    // セキュアDMモード：チャネル＋送信者ごとにDMコンテキストを分離。
    dmScope: "per-channel-peer",
  },
}
```

**有効にすべきタイミング：**

- 複数の送信者に対するペアリング承認がある場合
- 複数のエントリを持つDM許可リストを使用している場合
- `dmPolicy: "open"`を設定している場合
- 複数の電話番号やアカウントがエージェントにメッセージを送れる場合

注意事項：

- デフォルトは連続性のための`dmScope: "main"`（すべてのDMがメインセッションを共有）。シングルユーザー設定では問題ありません。
- ローカルCLIオンボーディングは、未設定の場合にデフォルトで`session.dmScope: "per-channel-peer"`を書き込みます（既存の明示的な値は保持されます）。
- 同じチャネルでのマルチアカウントインボックスには`per-account-channel-peer`を推奨します。
- 同じ人物が複数のチャネルで連絡してくる場合、`session.identityLinks`を使用してDMセッションを1つの正規IDに集約します。
- `openclaw security audit`でDM設定を確認できます（[セキュリティ](/cli/security)を参照）。

## ゲートウェイが信頼の源

すべてのセッション状態は**ゲートウェイ**（「マスター」OpenClaw）が**所有**しています。UIクライアント（macOSアプリ、WebChatなど）は、ローカルファイルを読む代わりにゲートウェイにセッションリストとトークン数を照会する必要があります。

- **リモートモード**では、重要なセッションストアはMacではなくリモートゲートウェイホスト上にあります。
- UIに表示されるトークン数はゲートウェイのストアフィールド（`inputTokens`、`outputTokens`、`totalTokens`、`contextTokens`）から取得されます。クライアントはJSONLトランスクリプトを解析して合計を「修正」しません。

## 状態の保存場所

- **ゲートウェイホスト**上：
  - ストアファイル：`~/.openclaw/agents/<agentId>/sessions/sessions.json`（エージェントごと）。
- トランスクリプト：`~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`（Telegramトピックセッションは`.../<SessionId>-topic-<threadId>.jsonl`を使用）。
- ストアは`sessionKey -> { sessionId, updatedAt, ... }`のマップです。エントリの削除は安全です。オンデマンドで再作成されます。
- グループエントリには、UIでセッションにラベルを付けるための`displayName`、`channel`、`subject`、`room`、`space`が含まれる場合があります。
- セッションエントリには`origin`メタデータ（ラベル＋ルーティングヒント）が含まれ、UIがセッションの出所を説明できるようにします。
- OpenClawはレガシーのPi/Tauセッションフォルダを**読み取りません**。

## メンテナンス

OpenClawは`sessions.json`とトランスクリプトのアーティファクトを時間とともに制限するためにセッションストアメンテナンスを適用します。

### デフォルト

- `session.maintenance.mode`：`warn`
- `session.maintenance.pruneAfter`：`30d`
- `session.maintenance.maxEntries`：`500`
- `session.maintenance.rotateBytes`：`10mb`
- `session.maintenance.resetArchiveRetention`：デフォルトは`pruneAfter`（`30d`）
- `session.maintenance.maxDiskBytes`：未設定（無効）
- `session.maintenance.highWaterBytes`：バジェットが有効な場合、`maxDiskBytes`の`80%`がデフォルト

### 仕組み

メンテナンスはセッションストアの書き込み時に実行され、`openclaw sessions cleanup`でオンデマンドでトリガーできます。

- `mode: "warn"`：何が削除されるかを報告しますが、エントリ/トランスクリプトを変更しません。
- `mode: "enforce"`：以下の順序でクリーンアップを適用します：
  1. `pruneAfter`より古い陳腐なエントリを削除
  2. エントリ数を`maxEntries`に制限（古いものから）
  3. 参照されなくなった削除済みエントリのトランスクリプトファイルをアーカイブ
  4. リテンションポリシーに基づいて古い`*.deleted.<timestamp>`および`*.reset.<timestamp>`アーカイブを削除
  5. `sessions.json`が`rotateBytes`を超えた場合にローテーション
  6. `maxDiskBytes`が設定されている場合、`highWaterBytes`に向けてディスクバジェットを強制（最も古いアーティファクトから、次に最も古いセッション）

### 大規模ストアのパフォーマンス注意事項

大規模なセッションストアは高トラフィック設定で一般的です。メンテナンス作業は書き込みパスの作業であるため、非常に大きなストアは書き込みレイテンシを増加させる可能性があります。

コストを最も増加させるもの：

- 非常に高い`session.maintenance.maxEntries`の値
- 陳腐なエントリを残す長い`pruneAfter`ウィンドウ
- `~/.openclaw/agents/<agentId>/sessions/`内の多数のトランスクリプト/アーカイブアーティファクト
- 適切なプルーニング/キャップ制限なしでディスクバジェット（`maxDiskBytes`）を有効にする

対処方法：

- 本番環境では`mode: "enforce"`を使用して成長を自動的に制限
- 時間とカウントの両方の制限を設定（`pruneAfter`＋`maxEntries`、片方だけではなく）
- 大規模デプロイメントではハード上限のために`maxDiskBytes`＋`highWaterBytes`を設定
- `highWaterBytes`を`maxDiskBytes`よりも有意に低く保つ（デフォルトは80%）
- 設定変更後に`openclaw sessions cleanup --dry-run --json`を実行して、強制前に予想される影響を確認
- 頻繁なアクティブセッションの場合、手動クリーンアップ実行時に`--active-key`を渡す

### カスタマイズ例

保守的な強制ポリシーの使用：

```json5
{
  session: {
    maintenance: {
      mode: "enforce",
      pruneAfter: "45d",
      maxEntries: 800,
      rotateBytes: "20mb",
      resetArchiveRetention: "14d",
    },
  },
}
```

セッションディレクトリのハードディスクバジェットの有効化：

```json5
{
  session: {
    maintenance: {
      mode: "enforce",
      maxDiskBytes: "1gb",
      highWaterBytes: "800mb",
    },
  },
}
```

大規模インストール用のチューニング（例）：

```json5
{
  session: {
    maintenance: {
      mode: "enforce",
      pruneAfter: "14d",
      maxEntries: 2000,
      rotateBytes: "25mb",
      maxDiskBytes: "2gb",
      highWaterBytes: "1.6gb",
    },
  },
}
```

CLIからのメンテナンスのプレビューまたは強制：

```bash
openclaw sessions cleanup --dry-run
openclaw sessions cleanup --enforce
```

## セッションプルーニング

OpenClawはデフォルトでLLMコール直前にインメモリコンテキストから**古いツール結果**をトリムします。
これはJSONL履歴を書き換え**ません**。[/concepts/session-pruning](/concepts/session-pruning)を参照してください。

## コンパクション前のメモリフラッシュ

セッションが自動コンパクションに近づくと、OpenClawはモデルに永続的なメモをディスクに書き込むよう促す**サイレントなメモリフラッシュ**ターンを実行できます。これはワークスペースが書き込み可能な場合にのみ実行されます。[メモリ](/concepts/memory)と[コンパクション](/concepts/compaction)を参照してください。

## トランスポート → セッションキーのマッピング

- ダイレクトチャットは`session.dmScope`に従います（デフォルト`main`）。
  - `main`：`agent:<agentId>:<mainKey>`（デバイス/チャネル間の連続性）。
    - 複数の電話番号とチャネルが同じエージェントメインキーにマッピングできます。1つの会話へのトランスポートとして機能します。
  - `per-peer`：`agent:<agentId>:dm:<peerId>`。
  - `per-channel-peer`：`agent:<agentId>:<channel>:dm:<peerId>`。
  - `per-account-channel-peer`：`agent:<agentId>:<channel>:<accountId>:dm:<peerId>`（accountIdのデフォルトは`default`）。
  - `session.identityLinks`がプロバイダープレフィックス付きのピアID（例：`telegram:123`）にマッチする場合、正規キーが`<peerId>`を置き換え、同じ人物がチャネル間でセッションを共有します。
- グループチャットは状態を分離：`agent:<agentId>:<channel>:group:<id>`（ルーム/チャネルは`agent:<agentId>:<channel>:channel:<id>`を使用）。
  - Telegramフォーラムトピックは分離のためにグループIDに`:topic:<threadId>`を追加します。
  - レガシーの`group:<id>`キーはマイグレーション用に引き続き認識されます。
- インバウンドコンテキストは引き続き`group:<id>`を使用する場合があります。チャネルは`Provider`から推論され、正規の`agent:<agentId>:<channel>:group:<id>`形式に正規化されます。
- その他のソース：
  - Cronジョブ：`cron:<job.id>`
  - Webhook：`hook:<uuid>`（hookによって明示的に設定されない限り）
  - Nodeの実行：`node-<nodeId>`

## ライフサイクル

- リセットポリシー：セッションは期限切れになるまで再利用され、期限切れは次の受信メッセージで評価されます。
- デイリーリセット：デフォルトは**ゲートウェイホストのローカル時間の午前4:00**です。最後の更新が最新のデイリーリセット時刻より前のセッションは陳腐です。
- アイドルリセット（オプション）：`idleMinutes`がスライディングアイドルウィンドウを追加します。デイリーリセットとアイドルリセットの両方が設定されている場合、**先に期限切れになった方**が新しいセッションを強制します。
- レガシーのアイドルのみ：`session.reset`/`resetByType`設定なしで`session.idleMinutes`を設定した場合、OpenClawは下位互換性のためにアイドルのみモードを維持します。
- タイプごとのオーバーライド（オプション）：`resetByType`で`direct`、`group`、`thread`セッションのポリシーをオーバーライドできます（thread = Slack/Discordスレッド、Telegramトピック、コネクタが提供する場合はMatrixスレッド）。
- チャネルごとのオーバーライド（オプション）：`resetByChannel`はチャネルのリセットポリシーをオーバーライドします（そのチャネルのすべてのセッションタイプに適用され、`reset`/`resetByType`よりも優先されます）。
- リセットトリガー：正確な`/new`または`/reset`（`resetTriggers`の追加分も含む）が新しいセッションIDを開始し、メッセージの残りを通過させます。`/new <model>`はモデルエイリアス、`provider/model`、またはプロバイダー名（ファジーマッチ）を受け付けて新しいセッションモデルを設定します。`/new`または`/reset`が単独で送信された場合、OpenClawはリセットを確認するための短い「こんにちは」グリーティングターンを実行します。
- 手動リセット：ストアから特定のキーを削除するか、JSONLトランスクリプトを削除します。次のメッセージで再作成されます。
- 分離されたCronジョブは常に実行ごとに新しい`sessionId`を生成します（アイドル再利用なし）。

## 送信ポリシー（オプション）

個別のIDをリストすることなく、特定のセッションタイプの配信をブロックします。

```json5
{
  session: {
    sendPolicy: {
      rules: [
        { action: "deny", match: { channel: "discord", chatType: "group" } },
        { action: "deny", match: { keyPrefix: "cron:" } },
        // 生のセッションキー（`agent:<id>:`プレフィックスを含む）にマッチ。
        { action: "deny", match: { rawKeyPrefix: "agent:main:discord:" } },
      ],
      default: "allow",
    },
  },
}
```

ランタイムオーバーライド（オーナーのみ）：

- `/send on` → このセッションで許可
- `/send off` → このセッションで拒否
- `/send inherit` → オーバーライドをクリアし設定ルールを使用
  スタンドアロンメッセージとして送信してください。

## 設定（オプションのリネーム例）

```json5
// ~/.openclaw/openclaw.json
{
  session: {
    scope: "per-sender", // グループキーを分離
    dmScope: "main", // DMの連続性（共有インボックスにはper-channel-peer/per-account-channel-peerを設定）
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      // デフォルト: mode=daily, atHour=4（ゲートウェイホストのローカル時間）。
      // idleMinutesも設定する場合、先に期限切れになった方が優先。
      mode: "daily",
      atHour: 4,
      idleMinutes: 120,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 10080 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    mainKey: "main",
  },
}
```

## 確認方法

- `openclaw status` — ストアパスと最近のセッションを表示。
- `openclaw sessions --json` — すべてのエントリをダンプ（`--active <minutes>`でフィルター）。
- `openclaw gateway call sessions.list --params '{}'` — 実行中のゲートウェイからセッションを取得（リモートゲートウェイアクセスには`--url`/`--token`を使用）。
- チャットでスタンドアロンメッセージとして`/status`を送信すると、エージェントが到達可能かどうか、セッションコンテキストの使用量、現在のthinking/verboseトグル、WhatsApp Web認証の最終更新時刻（再リンクの必要性の発見に役立つ）を確認できます。
- `/context list`または`/context detail`を送信して、システムプロンプトとインジェクションされたワークスペースファイルの内容（および最大のコンテキスト貢献者）を確認できます。
- `/stop`（またはスタンドアロンの中止フレーズ：`stop`、`stop action`、`stop run`、`stop openclaw`）を送信して、現在の実行を中止し、そのセッションのキューに入ったフォローアップをクリアし、それから生成されたサブエージェントの実行を停止します（返信には停止数が含まれます）。
- オプションの指示付きで`/compact`をスタンドアロンメッセージとして送信して、古いコンテキストを要約しウィンドウスペースを解放します。[/concepts/compaction](/concepts/compaction)を参照してください。
- JSONLトランスクリプトを直接開いて完全なターンを確認できます。

## ヒント

- プライマリキーを1:1トラフィック専用にし、グループには独自のキーを保持させてください。
- クリーンアップの自動化時は、他のコンテキストを保持するためにストア全体ではなく個別のキーを削除してください。

## セッションオリジンメタデータ

各セッションエントリは、どこから来たかを（ベストエフォートで）`origin`に記録します：

- `label`：人間可読ラベル（会話ラベル＋グループ件名/チャネルから解決）
- `provider`：正規化されたチャネルID（拡張を含む）
- `from`/`to`：インバウンドエンベロープからの生のルーティングID
- `accountId`：プロバイダーアカウントID（マルチアカウント時）
- `threadId`：チャネルがサポートしている場合のスレッド/トピックID
  オリジンフィールドはダイレクトメッセージ、チャネル、グループに対して設定されます。コネクタが配信ルーティングのみを更新する場合（例：DMメインセッションを最新に保つため）でも、セッションが説明メタデータを保持するようにインバウンドコンテキストを提供する必要があります。拡張はインバウンドコンテキストで`ConversationLabel`、`GroupSubject`、`GroupChannel`、`GroupSpace`、`SenderName`を送信し、`recordSessionMetaFromInbound`を呼び出す（または同じコンテキストを`updateLastRoute`に渡す）ことでこれを行えます。
