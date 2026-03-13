---
summary: "gogcliを通じてOpenClawウェブフックに接続されたGmail Pub/Subプッシュ"
read_when:
  - GmailインボックストリガーをOpenClawに接続する場合
  - エージェント起動のためのPub/Subプッシュを設定する場合
title: "Gmail PubSub"
---

# Gmail Pub/Sub → OpenClaw

目的：Gmail watch → Pub/Subプッシュ → `gog gmail watch serve` → OpenClawウェブフック。

## 前提条件

- `gcloud`がインストールされ、ログイン済み（[インストールガイド](https://docs.cloud.google.com/sdk/docs/install-sdk)）。
- `gog`（gogcli）がインストールされ、Gmailアカウントで認証済み（[gogcli.sh](https://gogcli.sh/)）。
- OpenClawフックが有効（[Webhooks](/automation/webhook)参照）。
- `tailscale`がログイン済み（[tailscale.com](https://tailscale.com/)）。サポートされているセットアップは、公開HTTPSエンドポイントにTailscale Funnelを使用します。
  他のトンネルサービスも使用できますが、DIY/非サポートであり、手動での接続が必要です。
  現在、Tailscaleのみをサポートしています。

フック設定例（Gmailプリセットマッピングを有効化）：

```json5
{
  hooks: {
    enabled: true,
    token: "OPENCLAW_HOOK_TOKEN",
    path: "/hooks",
    presets: ["gmail"],
  },
}
```

GmailサマリーをチャットUIに配信するには、`deliver` + オプションの`channel`/`to`を設定したマッピングでプリセットをオーバーライドします：

```json5
{
  hooks: {
    enabled: true,
    token: "OPENCLAW_HOOK_TOKEN",
    presets: ["gmail"],
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "New email from {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}\n{{messages[0].body}}",
        model: "openai/gpt-5.2-mini",
        deliver: true,
        channel: "last",
        // to: "+15551234567"
      },
    ],
  },
}
```

固定チャンネルが必要な場合は、`channel` + `to`を設定します。それ以外の場合、`channel: "last"`は最後の配信ルートを使用します（WhatsAppにフォールバック）。

Gmailの実行に安価なモデルを強制使用するには、マッピングで`model`を設定します（`provider/model`またはエイリアス）。`agents.defaults.models`を適用している場合は、そこにも含めてください。

GmailフックのデフォルトモデルとThinkingレベルを設定するには、設定に`hooks.gmail.model`/`hooks.gmail.thinking`を追加します：

```json5
{
  hooks: {
    gmail: {
      model: "openrouter/meta-llama/llama-3.3-70b-instruct:free",
      thinking: "off",
    },
  },
}
```

注意事項：

- マッピング内のフックごとの`model`/`thinking`は、これらのデフォルトより優先されます。
- フォールバック順：`hooks.gmail.model` → `agents.defaults.model.fallbacks` → プライマリ（認証/レート制限/タイムアウト）。
- `agents.defaults.models`が設定されている場合、GmailモデルはAllow Listに含まれている必要があります。
- Gmailフックのコンテンツはデフォルトで外部コンテンツ安全境界でラップされます。
  無効にするには（危険）、`hooks.gmail.allowUnsafeExternalContent: true`を設定します。

ペイロード処理をさらにカスタマイズするには、`hooks.mappings`を追加するか、`~/.openclaw/hooks/transforms`の下にJS/TSトランスフォームモジュールを配置します（[Webhooks](/automation/webhook)参照）。

## ウィザード（推奨）

OpenClawヘルパーを使用してすべてを接続します（macOSではbrewを介して依存関係をインストール）：

```bash
openclaw webhooks gmail setup \
  --account openclaw@gmail.com
```

デフォルト：

- 公開プッシュエンドポイントにTailscale Funnelを使用します。
- `openclaw webhooks gmail run`用に`hooks.gmail`設定を書き込みます。
- GmailフックプリセットをEnableします（`hooks.presets: ["gmail"]`）。

パスの注意：`tailscale.mode`が有効な場合、OpenClawは自動的に`hooks.gmail.serve.path`を`/`に設定し、公開パスを`hooks.gmail.tailscale.path`（デフォルト：`/gmail-pubsub`）に保持します。これはTailscaleがプロキシする前にセットパスプレフィックスを削除するためです。
バックエンドがプレフィックス付きパスを受け取る必要がある場合は、`hooks.gmail.tailscale.target`（または`--tailscale-target`）を`http://127.0.0.1:8788/gmail-pubsub`のような完全URLに設定し、`hooks.gmail.serve.path`と一致させてください。

カスタムエンドポイントが必要ですか？`--push-endpoint <url>`または`--tailscale off`を使用してください。

プラットフォームの注意：macOSでは、ウィザードがHomebrewを介して`gcloud`、`gogcli`、`tailscale`をインストールします。Linuxでは最初に手動でインストールしてください。

ゲートウェイの自動起動（推奨）：

- `hooks.enabled=true`かつ`hooks.gmail.account`が設定されている場合、ゲートウェイは起動時に`gog gmail watch serve`を開始し、ウォッチを自動更新します。
- オプトアウトするには`OPENCLAW_SKIP_GMAIL_WATCHER=1`を設定します（デーモンを自分で実行する場合に便利）。
- 手動デーモンを同時に実行しないでください。`listen tcp 127.0.0.1:8788: bind: address already in use`エラーが発生します。

手動デーモン（`gog gmail watch serve` + 自動更新を開始）：

```bash
openclaw webhooks gmail run
```

## 初回セットアップ

1. `gog`で使用するOAuthクライアントが属するGCPプロジェクトを選択します。

```bash
gcloud auth login
gcloud config set project <project-id>
```

注意：Gmail watchはPub/SubトピックがOAuthクライアントと同じプロジェクトにある必要があります。

2. APIを有効にします：

```bash
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

3. トピックを作成します：

```bash
gcloud pubsub topics create gog-gmail-watch
```

4. GmailプッシュがPublishできるようにします：

```bash
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

## ウォッチを開始する

```bash
gog gmail watch start \
  --account openclaw@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

出力から`history_id`を保存します（デバッグ用）。

## プッシュハンドラーを実行する

ローカル例（共有トークン認証）：

```bash
gog gmail watch serve \
  --account openclaw@gmail.com \
  --bind 127.0.0.1 \
  --port 8788 \
  --path /gmail-pubsub \
  --token <shared> \
  --hook-url http://127.0.0.1:18789/hooks/gmail \
  --hook-token OPENCLAW_HOOK_TOKEN \
  --include-body \
  --max-bytes 20000
```

注意事項：

- `--token`はプッシュエンドポイントを保護します（`x-gog-token`または`?token=`）。
- `--hook-url`はOpenClawの`/hooks/gmail`を指します（マップされ、分離実行 + メインへのサマリー）。
- `--include-body`と`--max-bytes`はOpenClawに送信されるボディスニペットを制御します。

推奨：`openclaw webhooks gmail run`は同じフローをラップし、ウォッチを自動更新します。

## ハンドラーの公開（上級、非サポート）

Tailscale以外のトンネルが必要な場合は、手動で接続し、プッシュサブスクリプションの公開URLを使用します（非サポート、ガードレールなし）：

```bash
cloudflared tunnel --url http://127.0.0.1:8788 --no-autoupdate
```

生成されたURLをプッシュエンドポイントとして使用します：

```bash
gcloud pubsub subscriptions create gog-gmail-watch-push \
  --topic gog-gmail-watch \
  --push-endpoint "https://<public-url>/gmail-pubsub?token=<shared>"
```

本番環境：安定したHTTPSエンドポイントを使用し、Pub/Sub OIDC JWTを設定してから実行します：

```bash
gog gmail watch serve --verify-oidc --oidc-email <svc@...>
```

## テスト

監視中のインボックスにメッセージを送信します：

```bash
gog gmail send \
  --account openclaw@gmail.com \
  --to openclaw@gmail.com \
  --subject "watch test" \
  --body "ping"
```

ウォッチ状態と履歴を確認します：

```bash
gog gmail watch status --account openclaw@gmail.com
gog gmail history --account openclaw@gmail.com --since <historyId>
```

## トラブルシューティング

- `Invalid topicName`：プロジェクトの不一致（トピックがOAuthクライアントプロジェクトにない）。
- `User not authorized`：トピックに`roles/pubsub.publisher`が不足。
- 空のメッセージ：GmailプッシュはHistoryIdのみを提供します。`gog gmail history`でフェッチしてください。

## クリーンアップ

```bash
gog gmail watch stop --account openclaw@gmail.com
gcloud pubsub subscriptions delete gog-gmail-watch-push
gcloud pubsub topics delete gog-gmail-watch
```
