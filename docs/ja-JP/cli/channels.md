---
summary: "`openclaw channels` の CLI リファレンス（アカウント、ステータス、ログイン/ログアウト、ログ）"
read_when:
  - チャンネルアカウント（WhatsApp/Telegram/Discord/Google Chat/Slack/Mattermost（プラグイン）/Signal/iMessage）を追加/削除したいとき
  - チャンネルのステータスを確認したり、チャンネルログを表示したいとき
title: "channels"
---

# `openclaw channels`

Gateway 上のチャットチャンネルアカウントとそのランタイムステータスを管理します。

関連ドキュメント:

- チャンネルガイド: [チャンネル](/channels/index)
- Gateway 設定: [設定](/gateway/configuration)

## 主要コマンド

```bash
openclaw channels list
openclaw channels status
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels logs --channel all
```

## アカウントの追加 / 削除

```bash
openclaw channels add --channel telegram --token <bot-token>
openclaw channels remove --channel telegram --delete
```

ヒント: `openclaw channels add --help` でチャンネルごとのフラグ（トークン、アプリトークン、signal-cli パスなど）を確認できます。

フラグなしで `openclaw channels add` を実行すると、対話型ウィザードが以下を確認します:

- 選択したチャンネルのアカウント ID
- それらのアカウントのオプションの表示名
- `設定済みのチャンネルアカウントを今すぐエージェントにバインドしますか？`

今すぐバインドを確認すると、ウィザードは各設定済みチャンネルアカウントをどのエージェントが所有すべきかを尋ね、アカウントスコープのルーティングバインディングを書き込みます。

同じルーティングルールは後から `openclaw agents bindings`、`openclaw agents bind`、`openclaw agents unbind` でも管理できます（[agents](/cli/agents) を参照）。

まだシングルアカウントのトップレベル設定（`channels.<channel>.accounts` エントリなし）を使用しているチャンネルにデフォルト以外のアカウントを追加すると、OpenClaw はアカウントスコープのシングルアカウントトップレベル値を `channels.<channel>.accounts.default` に移動し、新しいアカウントを書き込みます。これにより、マルチアカウント形式に移行しながら元のアカウント動作が保持されます。

ルーティング動作は一貫しています:

- 既存のチャンネルのみのバインディング（`accountId` なし）は引き続きデフォルトアカウントに一致します。
- `channels add` は非対話モードではバインディングを自動作成または書き換えません。
- 対話型セットアップではオプションでアカウントスコープのバインディングを追加できます。

設定が混在状態（名前付きアカウントが存在し、`default` がなく、トップレベルのシングルアカウント値がまだ設定されている）の場合は、`openclaw doctor --fix` を実行してアカウントスコープの値を `accounts.default` に移動してください。

## ログイン / ログアウト（対話型）

```bash
openclaw channels login --channel whatsapp
openclaw channels logout --channel whatsapp
```

## トラブルシューティング

- 広範な診断には `openclaw status --deep` を実行してください。
- ガイド付き修正には `openclaw doctor` を使用してください。
- `openclaw channels list` が `Claude: HTTP 403 ... user:profile` と表示する場合 → 使用状況スナップショットに `user:profile` スコープが必要です。`--no-usage` を使用するか、claude.ai セッションキー（`CLAUDE_WEB_SESSION_KEY` / `CLAUDE_WEB_COOKIE`）を提供するか、Claude Code CLI で再認証してください。
- `openclaw channels status` は Gateway に到達できない場合、設定のみのサマリーにフォールバックします。サポートされているチャンネルの資格情報が SecretRef 経由で設定されているが現在のコマンドパスで利用できない場合、そのアカウントは未設定ではなく、劣化ノート付きの設定済みとして報告されます。

## 機能プローブ

プロバイダーの機能ヒント（利用可能な場合はインテント/スコープ）と静的機能サポートを取得します:

```bash
openclaw channels capabilities
openclaw channels capabilities --channel discord --target channel:123
```

注意事項:

- `--channel` はオプションです。省略するとすべてのチャンネル（拡張機能を含む）が一覧表示されます。
- `--target` は `channel:<id>` または生の数値チャンネル ID を受け入れ、Discord にのみ適用されます。
- プローブはプロバイダー固有です: Discord インテント + オプションのチャンネル権限、Slack ボット + ユーザースコープ、Telegram ボットフラグ + Webhook、Signal デーモンバージョン、MS Teams アプリトークン + Graph ロール/スコープ（既知の場合は注釈付き）。プローブのないチャンネルは `Probe: unavailable` と報告されます。

## 名前から ID への解決

プロバイダーディレクトリを使用してチャンネル/ユーザー名を ID に解決します:

```bash
openclaw channels resolve --channel slack "#general" "@jane"
openclaw channels resolve --channel discord "My Server/#support" "@someone"
openclaw channels resolve --channel matrix "Project Room"
```

注意事項:

- `--kind user|group|auto` でターゲットタイプを強制できます。
- 同じ名前の複数のエントリがある場合、アクティブな一致が優先されます。
- `channels resolve` は読み取り専用です。選択されたアカウントが SecretRef 経由で設定されているがその資格情報が現在のコマンドパスで利用できない場合、コマンドは実行全体を中止せず、ノート付きの劣化した未解決結果を返します。
