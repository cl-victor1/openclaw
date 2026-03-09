---
summary: "signal-cli（JSON-RPC + SSE）経由のSignalサポート、セットアップパス、番号モデル"
read_when:
  - Signalサポートのセットアップ
  - Signalの送受信のデバッグ
title: "Signal"
---

# Signal (signal-cli)

ステータス：外部CLI統合。GatewayはHTTP JSON-RPC + SSE経由で `signal-cli` と通信します。

## 前提条件

- サーバーにOpenClawがインストール済み（以下のLinuxフローはUbuntu 24でテスト済み）。
- Gatewayが動作するホストで `signal-cli` が利用可能。
- 1つの確認SMSを受信できる電話番号（SMS登録パスの場合）。
- 登録時のSignalキャプチャ（`signalcaptchas.org`）用のブラウザアクセス。

## クイックセットアップ（初心者向け）

1. ボット用に**別のSignal番号**を使用する（推奨）。
2. `signal-cli` をインストールする（JVMビルドを使用する場合はJavaが必要）。
3. セットアップパスを選択：
   - **パスA（QRリンク）：** `signal-cli link -n "OpenClaw"` を実行してSignalでスキャン。
   - **パスB（SMS登録）：** キャプチャ + SMS確認で専用番号を登録。
4. OpenClawを設定してGatewayを再起動。
5. 最初のDMを送信してペアリングを承認（`openclaw pairing approve signal <CODE>`）。

最小設定：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

フィールドリファレンス：

| フィールド  | 説明                                              |
| ----------- | ------------------------------------------------- |
| `account`   | E.164形式のボット電話番号（`+15551234567`）       |
| `cliPath`   | `signal-cli` へのパス（`PATH` にある場合は `signal-cli`）|
| `dmPolicy`  | DMアクセスポリシー（`pairing` 推奨）              |
| `allowFrom` | DMを許可する電話番号または `uuid:<id>` 値         |

## 概要

- `signal-cli` 経由のSignalチャンネル（libsignalの埋め込みではない）。
- 決定論的ルーティング：返答は常にSignalに返されます。
- DMはエージェントのメインセッションを共有；グループは分離されます（`agent:<agentId>:signal:group:<groupId>`）。

## 設定書き込み

デフォルトでは、SignalはDMからの `/config set|unset` でトリガーされる設定更新の書き込みが許可されています（`commands.config: true` が必要）。

無効化：

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## 番号モデル（重要）

- GatewayはSignalの**デバイス**（`signal-cli` アカウント）に接続します。
- **個人のSignalアカウント**でボットを実行すると、自分自身のメッセージは無視されます（ループ保護）。
- 「テキストを送るとボットが返答する」用途には、**別のボット番号**を使用してください。

## セットアップパスA：既存のSignalアカウントをリンク（QR）

1. `signal-cli`（JVMまたはネイティブビルド）をインストール。
2. ボットアカウントをリンク：
   - `signal-cli link -n "OpenClaw"` を実行して、SignalでQRをスキャン。
3. Signalを設定してGatewayを起動。

例：

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      cliPath: "signal-cli",
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

マルチアカウントサポート：`channels.signal.accounts` にアカウントごとの設定とオプションの `name` を使用します。共通パターンは [`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts)を参照。

## セットアップパスB：専用ボット番号の登録（SMS、Linux）

既存のSignalアプリアカウントをリンクする代わりに専用ボット番号が必要な場合に使用します。

1. SMSを受信できる番号を取得（固定電話の場合は音声確認）。
   - アカウント/セッションの競合を避けるため専用ボット番号を使用。
2. Gatewayホストに `signal-cli` をインストール：

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

JVMビルド（`signal-cli-${VERSION}.tar.gz`）を使用する場合は、先にJRE 25+をインストールしてください。
`signal-cli` は最新版を維持してください；アップストリームはSignalサーバーAPIの変更で古いリリースが動作しなくなることがあると注記しています。

3. 番号を登録して確認：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

キャプチャが必要な場合：

1. `https://signalcaptchas.org/registration/generate.html` を開く。
2. キャプチャを完了して「Open Signal」の `signalcaptcha://...` リンクターゲットをコピー。
3. 可能であればブラウザセッションと同じ外部IPから実行。
4. すぐに登録を再実行（キャプチャトークンは急速に期限切れ）：

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. OpenClawを設定してGatewayを再起動、チャンネルを確認：

```bash
# ユーザーsystemdサービスとしてGatewayを実行している場合：
systemctl --user restart openclaw-gateway

# 確認：
openclaw doctor
openclaw channels status --probe
```

5. DMの送信者とペアリング：
   - ボット番号にメッセージを送信。
   - サーバーでコードを承認：`openclaw pairing approve signal <PAIRING_CODE>`。
   - 「Unknown contact」を避けるため、スマートフォンでボット番号を連絡先として保存。

重要：`signal-cli` で電話番号アカウントを登録すると、その番号のメインSignalアプリセッションの認証が解除される可能性があります。専用ボット番号を優先するか、既存のスマートフォンアプリのセットアップを維持する必要がある場合はQRリンクモードを使用してください。

アップストリームリファレンス：

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- キャプチャフロー: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- リンクフロー: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## 外部デーモンモード（httpUrl）

`signal-cli` を自分で管理したい場合（JVMの起動が遅い、コンテナ初期化、共有CPU）は、デーモンを別途実行してOpenClawを向けます：

```json5
{
  channels: {
    signal: {
      httpUrl: "http://127.0.0.1:8080",
      autoStart: false,
    },
  },
}
```

これにより、OpenClaw内部での自動スポーンと起動待機がスキップされます。自動スポーン時の起動が遅い場合は `channels.signal.startupTimeoutMs` を設定します。

## アクセス制御（DM + グループ）

DM：

- デフォルト：`channels.signal.dmPolicy = "pairing"`。
- 不明な送信者はペアリングコードを受け取り、承認されるまでメッセージは無視されます（コードは1時間で期限切れ）。
- 承認方法：
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal <CODE>`
- ペアリングはSignal DMのデフォルトトークン交換です。詳細：[ペアリング](/channels/pairing)
- UUIDのみの送信者（`sourceUuid` から）は `channels.signal.allowFrom` に `uuid:<id>` として保存されます。

グループ：

- `channels.signal.groupPolicy = open | allowlist | disabled`。
- `channels.signal.groupAllowFrom` は `allowlist` が設定された場合のグループでのトリガー権限を制御します。
- ランタイム注記：`channels.signal` が完全に欠落している場合、ランタイムはグループチェックに `groupPolicy="allowlist"` にフォールバックします（`channels.defaults.groupPolicy` が設定されていても）。

## 動作の仕組み

- `signal-cli` はデーモンとして動作；GatewayはSSE経由でイベントを読み取ります。
- 受信メッセージは共有チャンネルエンベロープに正規化されます。
- 返答は常に同じ番号またはグループにルーティングされます。

## メディア + 制限

- 送信テキストは `channels.signal.textChunkLimit`（デフォルト4000）でチャンクされます。
- オプションの改行チャンク：`channels.signal.chunkMode="newline"` で長さチャンク前に空白行（段落境界）で分割。
- 添付ファイルサポート（`signal-cli` からbase64で取得）。
- デフォルトメディア上限：`channels.signal.mediaMaxMb`（デフォルト8）。
- `channels.signal.ignoreAttachments` でメディアのダウンロードをスキップ。
- グループ履歴コンテキストは `channels.signal.historyLimit`（または `channels.signal.accounts.*.historyLimit`）を使用し、`messages.groupChat.historyLimit` にフォールバックします。`0` で無効化（デフォルト50）。

## タイピング + 既読確認

- **タイピングインジケーター**：OpenClawは `signal-cli sendTyping` でタイピング信号を送信し、返答中にリフレッシュします。
- **既読確認**：`channels.signal.sendReadReceipts` がtrueの場合、OpenClawは許可されたDMの既読確認を転送します。
- signal-cliはグループの既読確認を公開しません。

## リアクション（messageツール）

- `message action=react` に `channel=signal` を使用。
- ターゲット：送信者のE.164またはUUID（ペアリング出力の `uuid:<id>` を使用；裸のUUIDも可）。
- `messageId` はリアクション対象メッセージのSignalタイムスタンプ。
- グループリアクションには `targetAuthor` または `targetAuthorUuid` が必要。

例：

```
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

設定：

- `channels.signal.actions.reactions`：リアクションアクションの有効/無効（デフォルトtrue）。
- `channels.signal.reactionLevel`：`off | ack | minimal | extensive`。
  - `off`/`ack` はエージェントリアクションを無効化（messageツールの `react` はエラーになります）。
  - `minimal`/`extensive` はエージェントリアクションを有効化してガイダンスレベルを設定。
- アカウントごとのオーバーライド：`channels.signal.accounts.<id>.actions.reactions`、`channels.signal.accounts.<id>.reactionLevel`。

## 配信ターゲット（CLI/cron）

- DM：`signal:+15551234567`（またはプレーンE.164）。
- UUID DM：`uuid:<id>`（または裸のUUID）。
- グループ：`signal:group:<groupId>`。
- ユーザー名：`username:<name>`（Signalアカウントでサポートされている場合）。

## トラブルシューティング

まずこの手順を実行：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

必要に応じてDMペアリング状態を確認：

```bash
openclaw pairing list signal
```

よくある障害：

- デーモンは到達可能だが返答なし：アカウント/デーモン設定（`httpUrl`、`account`）と受信モードを確認。
- DMが無視される：送信者がペアリング承認待ち。
- グループメッセージが無視される：グループ送信者/メンションゲーティングが配信をブロック。
- 編集後の設定検証エラー：`openclaw doctor --fix` を実行。
- 診断でSignalが欠落：`channels.signal.enabled: true` を確認。

追加確認：

```bash
openclaw pairing list signal
pgrep -af signal-cli
grep -i "signal" "/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log" | tail -20
```

トリアージフロー：[/channels/troubleshooting](/channels/troubleshooting)。

## セキュリティ注記

- `signal-cli` はアカウントキーをローカルに保存します（通常 `~/.local/share/signal-cli/data/`）。
- サーバー移行や再構築の前にSignalアカウントの状態をバックアップ。
- 広範なDMアクセスを明示的に希望しない限り `channels.signal.dmPolicy: "pairing"` を維持。
- SMS確認は登録または回復フローでのみ必要ですが、番号/アカウントの制御を失うと再登録が複雑になる可能性があります。

## 設定リファレンス（Signal）

完全な設定：[設定](/gateway/configuration)

プロバイダーオプション：

- `channels.signal.enabled`：チャンネル起動の有効/無効。
- `channels.signal.account`：ボットアカウントのE.164。
- `channels.signal.cliPath`：`signal-cli` へのパス。
- `channels.signal.httpUrl`：完全なデーモンURL（host/portを上書き）。
- `channels.signal.httpHost`、`channels.signal.httpPort`：デーモンバインド（デフォルト127.0.0.1:8080）。
- `channels.signal.autoStart`：デーモンの自動スポーン（`httpUrl` が未設定の場合はデフォルトtrue）。
- `channels.signal.startupTimeoutMs`：起動待機タイムアウト（ms、上限120000）。
- `channels.signal.receiveMode`：`on-start | manual`。
- `channels.signal.ignoreAttachments`：添付ファイルのダウンロードをスキップ。
- `channels.signal.ignoreStories`：デーモンからのストーリーを無視。
- `channels.signal.sendReadReceipts`：既読確認を転送。
- `channels.signal.dmPolicy`：`pairing | allowlist | open | disabled`（デフォルト：pairing）。
- `channels.signal.allowFrom`：DM許可リスト（E.164または `uuid:<id>`）。`open` には `"*"` が必要。SignalにはユーザーネームがないのでPhone/UUID IDを使用。
- `channels.signal.groupPolicy`：`open | allowlist | disabled`（デフォルト：allowlist）。
- `channels.signal.groupAllowFrom`：グループ送信者許可リスト。
- `channels.signal.historyLimit`：コンテキストに含める最大グループメッセージ数（0で無効）。
- `channels.signal.dmHistoryLimit`：ユーザーターン数でのDM履歴制限。ユーザーごとのオーバーライド：`channels.signal.dms["<phone_or_uuid>"].historyLimit`。
- `channels.signal.textChunkLimit`：送信チャンクサイズ（文字数）。
- `channels.signal.chunkMode`：`length`（デフォルト）または `newline` で長さチャンク前に空白行（段落境界）で分割。
- `channels.signal.mediaMaxMb`：受信/送信メディア上限（MB）。

関連グローバルオプション：

- `agents.list[].groupChat.mentionPatterns`（Signalはネイティブメンションをサポートしません）。
- `messages.groupChat.mentionPatterns`（グローバルフォールバック）。
- `messages.responsePrefix`。
