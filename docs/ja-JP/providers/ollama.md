---
summary: "OpenClaw を Ollama（クラウドおよびローカルモデル）で実行する"
read_when:
  - Ollama を使用してクラウドまたはローカルモデルで OpenClaw を実行したい場合
  - Ollama のセットアップと設定のガイダンスが必要な場合
title: "Ollama"
---

# Ollama

Ollama はローカル LLM ランタイムで、マシン上でオープンソースモデルを簡単に実行できます。OpenClaw は Ollama のネイティブ API（`/api/chat`）と統合し、ストリーミングとツール呼び出しをサポートしています。`OLLAMA_API_KEY`（または認証プロファイル）を設定し、`models.providers.ollama` エントリを明示的に定義しない場合、ローカル Ollama モデルを自動検出できます。

<Warning>
**リモート Ollama ユーザーへ**: OpenClaw では `/v1` OpenAI 互換 URL（`http://host:11434/v1`）を使用しないでください。これによりツール呼び出しが壊れ、モデルが生のツール JSON をプレーンテキストとして出力する場合があります。代わりにネイティブ Ollama API URL を使用してください：`baseUrl: "http://host:11434"`（`/v1` なし）。
</Warning>

## クイックスタート

### オンボーディングウィザード（推奨）

Ollama のセットアップの最も速い方法は、オンボーディングウィザードを使用することです：

```bash
openclaw onboard
```

プロバイダーリストから **Ollama** を選択します。ウィザードは以下を行います：

1. インスタンスにアクセスできる Ollama ベース URL を尋ねます（デフォルト：`http://127.0.0.1:11434`）。
2. **Cloud + Local**（クラウドモデルとローカルモデル）または **Local**（ローカルモデルのみ）を選択できます。
3. **Cloud + Local** を選択し、ollama.com にサインインしていない場合、ブラウザのサインインフローを開きます。
4. 利用可能なモデルを検出してデフォルトを提案します。
5. 選択したモデルがローカルで利用できない場合、自動的にプルします。

非インタラクティブモードもサポートされています：

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --accept-risk
```

カスタムベース URL やモデルを指定することもできます：

```bash
openclaw onboard --non-interactive \
  --auth-choice ollama \
  --custom-base-url "http://ollama-host:11434" \
  --custom-model-id "qwen3.5:27b" \
  --accept-risk
```

### 手動セットアップ

1. Ollama をインストール：[https://ollama.com/download](https://ollama.com/download)

2. ローカル推論が必要な場合、ローカルモデルをプルします：

```bash
ollama pull glm-4.7-flash
# または
ollama pull gpt-oss:20b
# または
ollama pull llama3.3
```

3. クラウドモデルも使用したい場合はサインインします：

```bash
ollama signin
```

4. オンボーディングを実行して `Ollama` を選択します：

```bash
openclaw onboard
```

- `Local`：ローカルモデルのみ
- `Cloud + Local`：ローカルモデルとクラウドモデル
- `kimi-k2.5:cloud`、`minimax-m2.5:cloud`、`glm-5:cloud` などのクラウドモデルはローカルの `ollama pull` は**不要**です

OpenClaw が現在提案するモデル：

- ローカルデフォルト：`glm-4.7-flash`
- クラウドデフォルト：`kimi-k2.5:cloud`、`minimax-m2.5:cloud`、`glm-5:cloud`

5. 手動セットアップを希望する場合、OpenClaw 用に Ollama を直接有効化します（任意の値が使用可能です。Ollama は実際のキーを必要としません）：

```bash
# 環境変数を設定
export OLLAMA_API_KEY="ollama-local"

# または設定ファイルで設定
openclaw config set models.providers.ollama.apiKey "ollama-local"
```

6. モデルの確認または切り替え：

```bash
openclaw models list
openclaw models set ollama/glm-4.7-flash
```

7. または設定でデフォルトを設定します：

```json5
{
  agents: {
    defaults: {
      model: { primary: "ollama/glm-4.7-flash" },
    },
  },
}
```

## モデル検出（暗黙的プロバイダー）

`OLLAMA_API_KEY`（または認証プロファイル）を設定し、`models.providers.ollama` を定義**しない**場合、OpenClaw は `http://127.0.0.1:11434` のローカル Ollama インスタンスからモデルを検出します：

- `/api/tags` をクエリします
- ベストエフォートで `/api/show` ルックアップを使用して、利用可能な場合に `contextWindow` を読み取ります
- モデル名のヒューリスティック（`r1`、`reasoning`、`think`）で `reasoning` をマークします
- `maxTokens` を OpenClaw が使用するデフォルトの Ollama 最大トークン上限に設定します
- すべてのコストを `0` に設定します

これにより、ローカル Ollama インスタンスとカタログを同期させながら、手動のモデルエントリを避けることができます。

利用可能なモデルを確認するには：

```bash
ollama list
openclaw models list
```

新しいモデルを追加するには、Ollama でプルするだけです：

```bash
ollama pull mistral
```

新しいモデルは自動的に検出され、使用可能になります。

`models.providers.ollama` を明示的に設定した場合、自動検出はスキップされ、モデルを手動で定義する必要があります（以下を参照）。

## 設定

### 基本セットアップ（暗黙的な検出）

Ollama を有効化する最もシンプルな方法は環境変数を使用することです：

```bash
export OLLAMA_API_KEY="ollama-local"
```

### 明示的セットアップ（手動モデル定義）

以下の場合は明示的な設定を使用します：

- Ollama が別のホスト/ポートで動作している場合。
- 特定のコンテキストウィンドウまたはモデルリストを強制したい場合。
- 完全に手動のモデル定義が必要な場合。

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
        apiKey: "ollama-local",
        api: "ollama",
        models: [
          {
            id: "gpt-oss:20b",
            name: "GPT-OSS 20B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 8192 * 10
          }
        ]
      }
    }
  }
}
```

`OLLAMA_API_KEY` が設定されている場合、プロバイダーエントリで `apiKey` を省略でき、OpenClaw は可用性チェックのために自動的に補完します。

### カスタムベース URL（明示的設定）

Ollama が別のホストまたはポートで動作している場合（明示的設定は自動検出を無効にするため、モデルを手動で定義してください）：

```json5
{
  models: {
    providers: {
      ollama: {
        apiKey: "ollama-local",
        baseUrl: "http://ollama-host:11434", // /v1 なし - ネイティブ Ollama API URL を使用
        api: "ollama", // ネイティブツール呼び出しの動作を保証するために明示的に設定
      },
    },
  },
}
```

<Warning>
URL に `/v1` を追加しないでください。`/v1` パスは OpenAI 互換モードを使用し、そこではツール呼び出しが信頼できません。パスサフィックスなしのベース Ollama URL を使用してください。
</Warning>

### モデル選択

設定が完了すると、すべての Ollama モデルが利用可能になります：

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

## クラウドモデル

クラウドモデルを使用すると、ローカルモデルと並行してクラウドホスト型モデル（例：`kimi-k2.5:cloud`、`minimax-m2.5:cloud`、`glm-5:cloud`）を実行できます。

クラウドモデルを使用するには、オンボーディング中に **Cloud + Local** モードを選択します。ウィザードはサインイン済みかどうかを確認し、必要に応じてブラウザのサインインフローを開きます。認証を確認できない場合、ウィザードはローカルモデルのデフォルトにフォールバックします。

[ollama.com/signin](https://ollama.com/signin) から直接サインインすることもできます。

## 高度な設定

### 推論モデル

OpenClaw は `deepseek-r1`、`reasoning`、`think` などの名前を持つモデルをデフォルトで推論対応として扱います：

```bash
ollama pull deepseek-r1:32b
```

### モデルコスト

Ollama は無料でローカルで動作するため、すべてのモデルコストは $0 に設定されています。

### ストリーミング設定

OpenClaw の Ollama 統合はデフォルトで**ネイティブ Ollama API**（`/api/chat`）を使用し、ストリーミングとツール呼び出しを同時に完全サポートしています。特別な設定は不要です。

#### レガシー OpenAI 互換モード

<Warning>
**OpenAI 互換モードではツール呼び出しが信頼できません。** このモードは、OpenAI フォーマットのみをサポートするプロキシが必要で、かつネイティブツール呼び出しの動作に依存しない場合にのみ使用してください。
</Warning>

OpenAI 互換エンドポイントを使用する必要がある場合（例：OpenAI フォーマットのみをサポートするプロキシの背後）、`api: "openai-completions"` を明示的に設定します：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: true, // デフォルト: true
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

このモードはストリーミングとツール呼び出しを同時にサポートしない場合があります。モデル設定の `params: { streaming: false }` でストリーミングを無効にする必要があるかもしれません。

`api: "openai-completions"` を Ollama と一緒に使用する場合、OpenClaw はデフォルトで `options.num_ctx` を注入するため、Ollama が 4096 コンテキストウィンドウにサイレントフォールバックしません。プロキシ/アップストリームが未知の `options` フィールドを拒否する場合、この動作を無効にします：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434/v1",
        api: "openai-completions",
        injectNumCtxForOpenAICompat: false,
        apiKey: "ollama-local",
        models: [...]
      }
    }
  }
}
```

### コンテキストウィンドウ

自動検出されたモデルの場合、OpenClaw は利用可能な場合に Ollama が報告するコンテキストウィンドウを使用し、そうでない場合は OpenClaw が使用するデフォルトの Ollama コンテキストウィンドウにフォールバックします。明示的なプロバイダー設定で `contextWindow` と `maxTokens` を上書きできます。

## トラブルシューティング

### Ollama が検出されない

Ollama が動作していること、`OLLAMA_API_KEY`（または認証プロファイル）を設定したこと、そして明示的な `models.providers.ollama` エントリを定義**していない**ことを確認してください：

```bash
ollama serve
```

API がアクセス可能であることを確認します：

```bash
curl http://localhost:11434/api/tags
```

### 利用可能なモデルがない

モデルがリストに表示されない場合、以下のいずれかです：

- モデルをローカルにプルする、または
- `models.providers.ollama` でモデルを明示的に定義する。

モデルを追加するには：

```bash
ollama list  # インストール済みのモデルを確認
ollama pull glm-4.7-flash
ollama pull gpt-oss:20b
ollama pull llama3.3     # または別のモデル
```

### 接続が拒否される

Ollama が正しいポートで動作していることを確認します：

```bash
# Ollama が動作しているか確認
ps aux | grep ollama

# または Ollama を再起動
ollama serve
```

## 関連情報

- [モデルプロバイダー](/concepts/model-providers) - 全プロバイダーの概要
- [モデル選択](/concepts/models) - モデルの選び方
- [設定](/gateway/configuration) - 完全な設定リファレンス
