---
summary: "設定の概要: 一般的なタスク、クイックセットアップ、および完全なリファレンスへのリンク"
read_when:
  - OpenClawを初めてセットアップする場合
  - 一般的な設定パターンを探している場合
  - 特定の設定セクションに移動する場合
title: "設定"
---

# 設定

OpenClawは`~/.openclaw/openclaw.json`からオプションの<Tooltip tip="JSON5はコメントと末尾カンマをサポートします">**JSON5**</Tooltip>設定を読み込みます。

ファイルが存在しない場合、OpenClawは安全なデフォルト値を使用します。設定を追加する一般的な理由:

- チャンネルを接続し、ボットにメッセージを送信できるユーザーを制御する
- モデル、ツール、サンドボックス、または自動化（cron、フック）を設定する
- セッション、メディア、ネットワーキング、またはUIを調整する

利用可能なすべてのフィールドについては、[完全なリファレンス](/gateway/configuration-reference)を参照してください。

<Tip>
**設定が初めてですか？** 対話型セットアップには`openclaw onboard`から始めるか、コピー＆ペースト可能な完全な設定については[設定例](/gateway/configuration-examples)ガイドをご覧ください。
</Tip>

## 最小限の設定

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## 設定の編集

<Tabs>
  <Tab title="対話型ウィザード">
    ```bash
    openclaw onboard       # フルセットアップウィザード
    openclaw configure     # 設定ウィザード
    ```
  </Tab>
  <Tab title="CLI（ワンライナー）">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset tools.web.search.apiKey
    ```
  </Tab>
  <Tab title="コントロールUI">
    [http://127.0.0.1:18789](http://127.0.0.1:18789)を開き、**Config**タブを使用します。
    コントロールUIは設定スキーマからフォームをレンダリングし、エスケープハッチとして**Raw JSON**エディターを備えています。
  </Tab>
  <Tab title="直接編集">
    `~/.openclaw/openclaw.json`を直接編集します。Gatewayはファイルを監視し、変更を自動的に適用します（[ホットリロード](#config-hot-reload)を参照）。
  </Tab>
</Tabs>

## 厳格なバリデーション

<Warning>
OpenClawはスキーマに完全に一致する設定のみを受け入れます。不明なキー、不正な型、または無効な値があると、Gatewayは**起動を拒否**します。ルートレベルの唯一の例外は`$schema`（文字列）で、エディターがJSON Schemaメタデータを添付できるようにします。
</Warning>

バリデーションが失敗した場合:

- Gatewayは起動しません
- 診断コマンドのみが動作します（`openclaw doctor`、`openclaw logs`、`openclaw health`、`openclaw status`）
- `openclaw doctor`を実行して正確な問題を確認します
- `openclaw doctor --fix`（または`--yes`）を実行して修復を適用します

## 一般的なタスク

<AccordionGroup>
  <Accordion title="チャンネルのセットアップ（WhatsApp、Telegram、Discord等）">
    各チャンネルには`channels.<provider>`の下に独自の設定セクションがあります。セットアップ手順については、専用のチャンネルページを参照してください:

    - [WhatsApp](/channels/whatsapp) — `channels.whatsapp`
    - [Telegram](/channels/telegram) — `channels.telegram`
    - [Discord](/channels/discord) — `channels.discord`
    - [Slack](/channels/slack) — `channels.slack`
    - [Signal](/channels/signal) — `channels.signal`
    - [iMessage](/channels/imessage) — `channels.imessage`
    - [Google Chat](/channels/googlechat) — `channels.googlechat`
    - [Mattermost](/channels/mattermost) — `channels.mattermost`
    - [MS Teams](/channels/msteams) — `channels.msteams`

    すべてのチャンネルは同じDMポリシーパターンを共有します:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // pairing | allowlist | open | disabled
          allowFrom: ["tg:123"], // allowlist/openの場合のみ
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="モデルの選択と設定">
    プライマリモデルとオプションのフォールバックを設定します:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-5",
            fallbacks: ["openai/gpt-5.2"],
          },
          models: {
            "anthropic/claude-sonnet-4-5": { alias: "Sonnet" },
            "openai/gpt-5.2": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models`はモデルカタログを定義し、`/model`の許可リストとして機能します。
    - モデル参照は`provider/model`形式を使用します（例: `anthropic/claude-opus-4-6`）。
    - `agents.defaults.imageMaxDimensionPx`はトランスクリプト/ツール画像のダウンスケールを制御します（デフォルト`1200`）。低い値は通常、スクリーンショットの多い実行でビジョントークンの使用量を削減します。
    - チャットでのモデル切り替えについては[モデルCLI](/concepts/models)を、認証ローテーションとフォールバック動作については[モデルフェイルオーバー](/concepts/model-failover)を参照してください。
    - カスタム/セルフホストプロバイダーについては、リファレンスの[カスタムプロバイダー](/gateway/configuration-reference#custom-providers-and-base-urls)を参照してください。

  </Accordion>

  <Accordion title="ボットにメッセージを送信できるユーザーの制御">
    DMアクセスはチャンネルごとに`dmPolicy`で制御されます:

    - `"pairing"`（デフォルト）: 不明な送信者に承認用のワンタイムペアリングコードが発行されます
    - `"allowlist"`: `allowFrom`（またはペアリング済み許可ストア）のユーザーのみ
    - `"open"`: すべての受信DMを許可（`allowFrom: ["*"]`が必要）
    - `"disabled"`: すべてのDMを無視

    グループについては、`groupPolicy` + `groupAllowFrom`またはチャンネル固有の許可リストを使用します。

    チャンネルごとの詳細については、[完全なリファレンス](/gateway/configuration-reference#dm-and-group-access)を参照してください。

  </Accordion>

  <Accordion title="グループチャットのメンションゲーティングの設定">
    グループメッセージはデフォルトで**メンション必須**です。エージェントごとにパターンを設定します:

    ```json5
    {
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **メタデータメンション**: ネイティブの@メンション（WhatsAppのタップ・トゥ・メンション、Telegramの@bot等）
    - **テキストパターン**: `mentionPatterns`の正規表現パターン
    - チャンネルごとのオーバーライドとセルフチャットモードについては、[完全なリファレンス](/gateway/configuration-reference#group-chat-mention-gating)を参照してください。

  </Accordion>

  <Accordion title="セッションとリセットの設定">
    セッションは会話の継続性と分離を制御します:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // マルチユーザーに推奨
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main`（共有） | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: スレッドバインドセッションルーティングのグローバルデフォルト（Discordは`/focus`、`/unfocus`、`/agents`、`/session idle`、`/session max-age`をサポート）。
    - スコーピング、アイデンティティリンク、送信ポリシーについては[セッション管理](/concepts/session)を参照してください。
    - すべてのフィールドについては[完全なリファレンス](/gateway/configuration-reference#session)を参照してください。

  </Accordion>

  <Accordion title="サンドボックスの有効化">
    エージェントセッションを分離されたDockerコンテナで実行します:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    まずイメージをビルドします: `scripts/sandbox-setup.sh`

    完全なガイドについては[サンドボックス](/gateway/sandboxing)を、すべてのオプションについては[完全なリファレンス](/gateway/configuration-reference#sandbox)を参照してください。

  </Accordion>

  <Accordion title="ハートビート（定期チェックイン）の設定">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: 期間文字列（`30m`、`2h`）。`0m`で無効化。
    - `target`: `last` | `whatsapp` | `telegram` | `discord` | `none`
    - `directPolicy`: `allow`（デフォルト）または`block`（DM形式のハートビートターゲット用）
    - 完全なガイドについては[ハートビート](/gateway/heartbeat)を参照してください。

  </Accordion>

  <Accordion title="cronジョブの設定">
    ```json5
    {
      cron: {
        enabled: true,
        maxConcurrentRuns: 2,
        sessionRetention: "24h",
        runLog: {
          maxBytes: "2mb",
          keepLines: 2000,
        },
      },
    }
    ```

    - `sessionRetention`: `sessions.json`から完了した分離実行セッションをプルーニング（デフォルト`24h`、`false`で無効化）。
    - `runLog`: `cron/runs/<jobId>.jsonl`をサイズと保持行数でプルーニング。
    - 機能の概要とCLIの例については[cronジョブ](/automation/cron-jobs)を参照してください。

  </Accordion>

  <Accordion title="Webhook（フック）の設定">
    GatewayでHTTP Webhookエンドポイントを有効にします:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    セキュリティに関する注意:
    - すべてのフック/Webhookペイロードコンテンツを信頼できない入力として扱います。
    - 厳密にスコープされたデバッグを行う場合を除き、安全でないコンテンツバイパスフラグ（`hooks.gmail.allowUnsafeExternalContent`、`hooks.mappings[].allowUnsafeExternalContent`）を無効のままにします。
    - フック駆動のエージェントには、強力な最新モデルティアと厳格なツールポリシー（例えばメッセージングのみとサンドボックスの組み合わせ）を推奨します。

    すべてのマッピングオプションとGmail統合については[完全なリファレンス](/gateway/configuration-reference#hooks)を参照してください。

  </Accordion>

  <Accordion title="マルチエージェントルーティングの設定">
    個別のワークスペースとセッションを持つ複数の分離されたエージェントを実行します:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    バインディングルールとエージェントごとのアクセスプロファイルについては、[マルチエージェント](/concepts/multi-agent)および[完全なリファレンス](/gateway/configuration-reference#multi-agent-routing)を参照してください。

  </Accordion>

  <Accordion title="設定を複数のファイルに分割する（$include）">
    大きな設定を整理するために`$include`を使用します:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **単一ファイル**: 含むオブジェクトを置換
    - **ファイルの配列**: 順番にディープマージ（後のものが優先）
    - **兄弟キー**: インクルード後にマージ（インクルードされた値をオーバーライド）
    - **ネストされたインクルード**: 最大10レベルの深さまでサポート
    - **相対パス**: インクルード元ファイルからの相対パスで解決
    - **エラーハンドリング**: ファイル不存在、パースエラー、循環インクルードに対する明確なエラー

  </Accordion>
</AccordionGroup>

## 設定のホットリロード

Gatewayは`~/.openclaw/openclaw.json`を監視し、変更を自動的に適用します — ほとんどの設定で手動再起動は不要です。

### リロードモード

| モード | 動作 |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`**（デフォルト） | 安全な変更を即座にホット適用。重要な変更は自動的に再起動。 |
| **`hot`** | 安全な変更のみホット適用。再起動が必要な場合は警告をログに記録 — ユーザーが対応。 |
| **`restart`** | 安全かどうかに関係なく、設定変更時にGatewayを再起動。 |
| **`off`** | ファイル監視を無効化。変更は次の手動再起動で有効。 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### ホット適用と再起動が必要なもの

ほとんどのフィールドはダウンタイムなしでホット適用されます。`hybrid`モードでは、再起動が必要な変更は自動的に処理されます。

| カテゴリ | フィールド | 再起動が必要？ |
| ------------------- | -------------------------------------------------------------------- | --------------- |
| チャンネル | `channels.*`、`web`（WhatsApp） — すべての組み込みおよび拡張チャンネル | いいえ |
| エージェントとモデル | `agent`、`agents`、`models`、`routing` | いいえ |
| 自動化 | `hooks`、`cron`、`agent.heartbeat` | いいえ |
| セッションとメッセージ | `session`、`messages` | いいえ |
| ツールとメディア | `tools`、`browser`、`skills`、`audio`、`talk` | いいえ |
| UIその他 | `ui`、`logging`、`identity`、`bindings` | いいえ |
| Gatewayサーバー | `gateway.*`（ポート、バインド、認証、Tailscale、TLS、HTTP） | **はい** |
| インフラストラクチャ | `discovery`、`canvasHost`、`plugins` | **はい** |

<Note>
`gateway.reload`と`gateway.remote`は例外です — これらを変更しても再起動はトリガーされ**ません**。
</Note>

## 設定RPC（プログラマティック更新）

<Note>
コントロールプレーンの書き込みRPC（`config.apply`、`config.patch`、`update.run`）は、`deviceId+clientIp`あたり**60秒間に3リクエスト**にレート制限されています。制限時、RPCは`retryAfterMs`付きの`UNAVAILABLE`を返します。
</Note>

<AccordionGroup>
  <Accordion title="config.apply（完全置換）">
    設定全体を検証＋書き込みし、Gatewayを1ステップで再起動します。

    <Warning>
    `config.apply`は**設定全体**を置換します。部分的な更新には`config.patch`を、単一キーには`openclaw config set`を使用してください。
    </Warning>

    パラメータ:

    - `raw`（文字列） — 設定全体のJSON5ペイロード
    - `baseHash`（オプション） — `config.get`からの設定ハッシュ（設定が存在する場合は必須）
    - `sessionKey`（オプション） — 再起動後のウェイクアップping用のセッションキー
    - `note`（オプション） — 再起動センチネル用のメモ
    - `restartDelayMs`（オプション） — 再起動前の遅延（デフォルト2000）

    再起動リクエストは保留中/実行中のものがある間は統合され、再起動サイクル間に30秒のクールダウンが適用されます。

    ```bash
    openclaw gateway call config.get --params '{}'  # payload.hashを取得
    openclaw gateway call config.apply --params '{
      "raw": "{ agents: { defaults: { workspace: \"~/.openclaw/workspace\" } } }",
      "baseHash": "<hash>",
      "sessionKey": "agent:main:whatsapp:dm:+15555550123"
    }'
    ```

  </Accordion>

  <Accordion title="config.patch（部分更新）">
    既存の設定に部分的な更新をマージします（JSONマージパッチセマンティクス）:

    - オブジェクトは再帰的にマージ
    - `null`はキーを削除
    - 配列は置換

    パラメータ:

    - `raw`（文字列） — 変更するキーのみを含むJSON5
    - `baseHash`（必須） — `config.get`からの設定ハッシュ
    - `sessionKey`、`note`、`restartDelayMs` — `config.apply`と同じ

    再起動の動作は`config.apply`と一致: 保留中の再起動の統合と再起動サイクル間の30秒クールダウン。

    ```bash
    openclaw gateway call config.patch --params '{
      "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
      "baseHash": "<hash>"
    }'
    ```

  </Accordion>
</AccordionGroup>

## 環境変数

OpenClawは親プロセスからの環境変数に加えて以下を読み込みます:

- カレントワーキングディレクトリの`.env`（存在する場合）
- `~/.openclaw/.env`（グローバルフォールバック）

いずれのファイルも既存の環境変数をオーバーライドしません。設定内でインライン環境変数を設定することもできます:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="シェル環境インポート（オプション）">
  有効にした場合、期待されるキーが設定されていないと、OpenClawはログインシェルを実行し、不足しているキーのみをインポートします:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

環境変数での同等設定: `OPENCLAW_LOAD_SHELL_ENV=1`
</Accordion>

<Accordion title="設定値での環境変数置換">
  `${VAR_NAME}`で任意の設定文字列値内の環境変数を参照できます:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

ルール:

- 大文字の名前のみマッチ: `[A-Z_][A-Z0-9_]*`
- 不足/空の変数はロード時にエラーをスロー
- `$${VAR}`でリテラル出力にエスケープ
- `$include`ファイル内でも動作
- インライン置換: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="シークレット参照（env、file、exec）">
  SecretRefオブジェクトをサポートするフィールドでは、以下を使用できます:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "nano-banana-pro": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/nano-banana-pro/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccountRef: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

SecretRefの詳細（`env`/`file`/`exec`の`secrets.providers`を含む）は[シークレット管理](/gateway/secrets)にあります。
サポートされる資格情報パスは[SecretRef資格情報サーフェス](/reference/secretref-credential-surface)に記載されています。
</Accordion>

完全な優先順位とソースについては[環境](/help/environment)を参照してください。

## 完全なリファレンス

フィールドごとの完全なリファレンスについては、**[設定リファレンス](/gateway/configuration-reference)**を参照してください。

---

_関連: [設定例](/gateway/configuration-examples) · [設定リファレンス](/gateway/configuration-reference) · [Doctor](/gateway/doctor)_
