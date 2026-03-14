---
summary: "OpenClaw で API キーまたは Codex サブスクリプションを使用して OpenAI を利用する"
read_when:
  - OpenClaw で OpenAI モデルを使用したい場合
  - API キーの代わりに Codex サブスクリプション認証を使用したい場合
title: "OpenAI"
---

# OpenAI

OpenAI は GPT モデル向けの開発者 API を提供しています。Codex はサブスクリプションアクセスに対して **ChatGPT サインイン**をサポートし、従量課金アクセスには **API キー**サインインをサポートしています。Codex クラウドには ChatGPT サインインが必要です。
OpenAI は OpenClaw のような外部ツール/ワークフローでのサブスクリプション OAuth 使用を明示的にサポートしています。

## オプション A：OpenAI API キー（OpenAI Platform）

**最適な用途：** 直接 API アクセスと従量課金制。
OpenAI ダッシュボードから API キーを取得してください。

### CLI セットアップ

```bash
openclaw onboard --auth-choice openai-api-key
# または非インタラクティブ
openclaw onboard --openai-api-key "$OPENAI_API_KEY"
```

### 設定スニペット

```json5
{
  env: { OPENAI_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "openai/gpt-5.4" } } },
}
```

OpenAI の現在の API モデルドキュメントには、直接 OpenAI API 使用のために `gpt-5.4` と `gpt-5.4-pro` が掲載されています。OpenClaw は両方を `openai/*` Responses パスを通じて転送します。
OpenClaw は古くなった `openai/gpt-5.3-codex-spark` の行を意図的に抑制します。なぜなら、直接 OpenAI API 呼び出しがライブトラフィックでこれを拒否するためです。

OpenClaw は直接 OpenAI API パスで `openai/gpt-5.3-codex-spark` を公開**しません**。`pi-ai` はそのモデルの組み込み行を引き続き出荷しますが、現在のライブ OpenAI API リクエストはそれを拒否します。Spark は OpenClaw では Codex 専用として扱われます。

## オプション B：OpenAI Code（Codex）サブスクリプション

**最適な用途：** API キーの代わりに ChatGPT/Codex サブスクリプションアクセスを使用する場合。
Codex クラウドには ChatGPT サインインが必要ですが、Codex CLI は ChatGPT または API キーサインインをサポートしています。

### CLI セットアップ（Codex OAuth）

```bash
# ウィザードで Codex OAuth を実行
openclaw onboard --auth-choice openai-codex

# または直接 OAuth を実行
openclaw models auth login --provider openai-codex
```

### 設定スニペット（Codex サブスクリプション）

```json5
{
  agents: { defaults: { model: { primary: "openai-codex/gpt-5.4" } } },
}
```

OpenAI の現在の Codex ドキュメントには、現在の Codex モデルとして `gpt-5.4` が掲載されています。OpenClaw はこれを ChatGPT/Codex OAuth 使用のために `openai-codex/gpt-5.4` にマッピングします。

Codex アカウントが Codex Spark の利用権を持っている場合、OpenClaw は以下もサポートしています：

- `openai-codex/gpt-5.3-codex-spark`

OpenClaw は Codex Spark を Codex 専用として扱います。直接 `openai/gpt-5.3-codex-spark` API キーパスは公開しません。

OpenClaw は `pi-ai` が検出した場合に `openai-codex/gpt-5.3-codex-spark` も保持します。これは利用権依存かつ試験的なものとして扱ってください：Codex Spark は GPT-5.4 の `/fast` とは別物であり、利用可能性はサインインした Codex / ChatGPT アカウントに依存します。

### トランスポートのデフォルト

OpenClaw はモデルのストリーミングに `pi-ai` を使用します。`openai/*` と `openai-codex/*` の両方で、デフォルトのトランスポートは `"auto"`（WebSocket 優先、次に SSE フォールバック）です。

`agents.defaults.models.<provider/model>.params.transport` で設定できます：

- `"sse"`：SSE を強制する
- `"websocket"`：WebSocket を強制する
- `"auto"`：WebSocket を試み、次に SSE にフォールバックする

`openai/*`（Responses API）では、WebSocket トランスポート使用時に OpenClaw はデフォルトで WebSocket ウォームアップを有効化します（`openaiWsWarmup: true`）。

関連する OpenAI ドキュメント：

- [Realtime API with WebSocket](https://platform.openai.com/docs/guides/realtime-websocket)
- [Streaming API responses (SSE)](https://platform.openai.com/docs/guides/streaming-responses)

```json5
{
  agents: {
    defaults: {
      model: { primary: "openai-codex/gpt-5.4" },
      models: {
        "openai-codex/gpt-5.4": {
          params: {
            transport: "auto",
          },
        },
      },
    },
  },
}
```

### OpenAI WebSocket ウォームアップ

OpenAI ドキュメントではウォームアップはオプションとして説明されています。OpenClaw は WebSocket トランスポート使用時の最初のターンのレイテンシを削減するために、`openai/*` のデフォルトでウォームアップを有効化します。

### ウォームアップを無効にする

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: false,
          },
        },
      },
    },
  },
}
```

### ウォームアップを明示的に有効にする

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            openaiWsWarmup: true,
          },
        },
      },
    },
  },
}
```

### OpenAI 優先処理

OpenAI の API は `service_tier=priority` 経由で優先処理を公開しています。
OpenClaw では、`agents.defaults.models["openai/<model>"].params.serviceTier` を設定することで、直接 `openai/*` Responses リクエストにそのフィールドを渡すことができます。

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

サポートされている値は `auto`、`default`、`flex`、`priority` です。

### OpenAI fast モード

OpenClaw は `openai/*` と `openai-codex/*` セッションの両方に対して共有の fast モードトグルを提供しています：

- チャット/UI：`/fast status|on|off`
- 設定：`agents.defaults.models["<provider>/<model>"].params.fastMode`

fast モードが有効な場合、OpenClaw は低レイテンシの OpenAI プロファイルを適用します：

- ペイロードにすでに reasoning が指定されていない場合、`reasoning.effort = "low"`
- ペイロードにすでに verbosity が指定されていない場合、`text.verbosity = "low"`
- `api.openai.com` への直接 `openai/*` Responses 呼び出しに対して `service_tier = "priority"`

例：

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
        "openai-codex/gpt-5.4": {
          params: {
            fastMode: true,
          },
        },
      },
    },
  },
}
```

セッションのオーバーライドは設定より優先されます。Sessions UI でセッションのオーバーライドをクリアすると、セッションは設定されたデフォルトに戻ります。

### OpenAI Responses サーバーサイド圧縮

直接 OpenAI Responses モデル（`api.openai.com` 上の `baseUrl` を持つ `api: "openai-responses"` を使用する `openai/*`）に対して、OpenClaw は OpenAI サーバーサイド圧縮ペイロードヒントを自動有効化するようになりました：

- `store: true` を強制します（モデルの互換性が `supportsStore: false` を設定していない限り）
- `context_management: [{ type: "compaction", compact_threshold: ... }]` を注入します

デフォルトでは、`compact_threshold` はモデルの `contextWindow` の `70%`（利用できない場合は `80000`）です。

### サーバーサイド圧縮を明示的に有効にする

互換性のある Responses モデル（例：Azure OpenAI Responses）で `context_management` の注入を強制したい場合に使用します：

```json5
{
  agents: {
    defaults: {
      models: {
        "azure-openai-responses/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
          },
        },
      },
    },
  },
}
```

### カスタムしきい値で有効にする

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: true,
            responsesCompactThreshold: 120000,
          },
        },
      },
    },
  },
}
```

### サーバーサイド圧縮を無効にする

```json5
{
  agents: {
    defaults: {
      models: {
        "openai/gpt-5.4": {
          params: {
            responsesServerCompaction: false,
          },
        },
      },
    },
  },
}
```

`responsesServerCompaction` は `context_management` の注入のみを制御します。
直接 OpenAI Responses モデルは、互換性が `supportsStore: false` を設定していない限り、引き続き `store: true` を強制します。

## 注意事項

- モデル参照は常に `provider/model` を使用します（[/concepts/models](/concepts/models) を参照）。
- 認証の詳細と再利用ルールは [/concepts/oauth](/concepts/oauth) にあります。
