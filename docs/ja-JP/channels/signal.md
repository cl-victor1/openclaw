---
summary: "signal-cli（JSON-RPC + SSE）経由のSignalサポート、セットアップ手順、および番号モデル"
read_when:
  - Signalサポートをセットアップするとき
  - Signalの送受信をデバッグするとき
title: "Signal"
---

# Signal (signal-cli)

ステータス: 外部CLI統合。ゲートウェイはHTTP JSON-RPC + SSE経由で`signal-cli`と通信します。

## 前提条件

- サーバーにOpenClawがインストール済み（以下のLinux手順はUbuntu 24でテスト済み）。
- ゲートウェイが動作するホストで`signal-cli`が利用可能。
- SMS検証を受信できる電話番号（SMS登録パスの場合）。
- 登録時のSignalキャプチャ用のブラウザアクセス（`signalcaptchas.org`）。

## クイックセットアップ（初心者向け）

1. ボット専用の**別のSignal番号**を使用する（推奨）。
2. `signal-cli`をインストールする（JVMビルドを使用する場合はJavaが必要）。
3. セットアップパスを選択する:
   - **パスA（QRリンク）:** `signal-cli link -n "OpenClaw"`を実行してSignalでスキャンする。
   - **パスB（SMS登録）:** キャプチャ + SMS検証で専用番号を登録する。
4. OpenClawを設定してゲートウェイを再起動する。
5. 最初のDMを送信してペアリングを承認する（`openclaw pairing approve signal <CODE>`）。

最小設定:

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

フィールドリファレンス:

| フィールド  | 説明                                                |
| ----------- | --------------------------------------------------- |
| `account`   | E.164形式のボット電話番号（`+15551234567`）         |
| `cliPath`   | `signal-cli`へのパス（PATHにある場合は`signal-cli`）|
| `dmPolicy`  | DMアクセスポリシー（`pairing`推奨）                 |
| `allowFrom` | DMを許可する電話番号または`uuid:<id>`値             |

## 概要

- `signal-cli`経由のSignalチャンネル（libsignal組み込みではない）。
- 決定論的ルーティング: 返信は常にSignalに戻ります。
- DMはエージェントのメインセッションを共有し、グループは分離されます（`agent:<agentId>:signal:group:<groupId>`）。

## 設定書き込み

デフォルトでは、Signalは`/config set|unset`でトリガーされる設定更新の書き込みを許可します（`commands.config: true`が必要）。

無効化:

```json5
{
  channels: { signal: { configWrites: false } },
}
```

## 番号モデル（重要）

- ゲートウェイは**Signalデバイス**（`signal-cli`アカウント）に接続します。
- **個人のSignalアカウント**でボットを実行する場合、ループ保護のため自分のメッセージは無視されます。
- 「ボットにテキストを送ると返信が来る」ためには、**別のボット番号**を使用してください。

## セットアップパスA: 既存のSignalアカウントをリンクする（QR）

1. `signal-cli`をインストールする（JVMまたはネイティブビルド）。
2. ボットアカウントをリンクする:
   - `signal-cli link -n "OpenClaw"`を実行してSignalでQRをスキャンする。
3. Signalを設定してゲートウェイを起動する。

例:

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

マルチアカウントサポート: アカウントごとの設定とオプションの`name`を持つ`channels.signal.accounts`を使用します。共有パターンについては[`gateway/configuration`](/gateway/configuration#telegramaccounts--discordaccounts--slackaccounts--signalaccounts--imessageaccounts)を参照してください。

## セットアップパスB: 専用ボット番号を登録する（SMS、Linux）

既存のSignalアプリアカウントをリンクする代わりに専用のボット番号を使用する場合に使います。

1. SMSを受信できる番号を取得する（固定電話の場合は音声確認）。
   - アカウント/セッションの競合を避けるため、専用のボット番号を使用してください。
2. ゲートウェイホストに`signal-cli`をインストールする:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

JVMビルド（`signal-cli-${VERSION}.tar.gz`）を使用する場合は、先にJRE 25+をインストールしてください。
`signal-cli`を最新に保ってください。上流では古いリリースがSignalサーバーAPIの変更で壊れる可能性があると記載されています。

3. 番号を登録して確認する:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

キャプチャが必要な場合:

1. `https://signalcaptchas.org/registration/generate.html`を開く。
2. キャプチャを完了し、「Open Signal」の`signalcaptcha://...`リンクターゲットをコピーする。
3. できればブラウザセッションと同じ外部IPから実行する。
4. すぐに再度登録を実行する（キャプチャトークンはすぐに期限切れになります）:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. OpenClawを設定してゲートウェイを再起動し、チャンネルを確認する:

```bash
# ユーザーsystemdサービスとしてゲートウェイを実行している場合:
systemctl --user restart openclaw-gateway

# 確認:
openclaw doctor
openclaw channels status --probe
```

5. DMの送信者をペアリングする:
   - ボット番号にメッセージを送信する。
   - サーバーでコードを承認する: `openclaw pairing approve signal <PAIRING_CODE>`。
   - 「Unknown contact」を避けるため、ボット番号を電話の連絡先として保存する。

重要: 電話番号アカウントを`signal-cli`で登録すると、その番号のメインSignalアプリセッションが認証解除される場合があります。既存の電話アプリのセットアップを維持する必要がある場合は、専用のボット番号またはQRリンクモードを優先してください。

上流リファレンス:

- `signal-cli` README: `https://github.com/AsamK/signal-cli`
- キャプチャフロー: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- リンクフロー: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## 外部デーモンモード（httpUrl）

`signal-cli`を自分で管理したい場合（低速なJVMコールドスタート、コンテナの初期化、共有CPU）、デーモンを別途実行してOpenClawにポイントします:

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

これにより、OpenClaw内での自動スポーンと起動待機がスキップされます。自動スポーン時の低速な起動には`channels.signal.startupTimeoutMs`を設定してください。

## アクセス制御（DM + グループ）

DM:

- デフォルト: `channels.signal.dmPolicy = "pairing"`。
- 未知の送信者はペアリングコードを受信し、承認されるまでメッセージは無視されます（コードは1時間で期限切れ）。
- 承認方法:
  - `openclaw pairing list signal`
  - `openclaw pairing approve signal <CODE>`
- ペアリングはSignal DMのデフォルトトークン交換です。詳細: [ペアリング](/channels/pairing)
- UUIDのみの送信者（`sourceUuid`から）は`channels.signal.allowFrom`に`uuid:<id>`として保存されます。

グループ:

- `channels.signal.groupPolicy = open | allowlist | disabled`。
- `channels.signal.groupAllowFrom`は`allowlist`が設定されている場合にグループでトリガーできるユーザーを制御します。
- ランタイムノート: `channels.signal`が完全に欠落している場合、`channels.defaults.groupPolicy`が設定されていても、ランタイムはグループチェックのために`groupPolicy="allowlist"`にフォールバックします。

## 動作の仕組み

- `signal-cli`はデーモンとして動作し、ゲートウェイはSSE経由でイベントを読み取ります。
- 受信メッセージは共有チャンネルエンベロープに正規化されます。
- 返信は常に同じ番号またはグループにルーティングされます。

## メディア + 制限

- 送信テキストは`channels.signal.textChunkLimit`（デフォルト4000）にチャンクされます。
- オプションの改行チャンキング: `channels.signal.chunkMode="newline"`を設定すると、長さチャンキングの前に空白行（段落境界）で分割します。
- 添付ファイルをサポート（`signal-cli`からbase64でフェッチ）。
- デフォルトのメディアキャップ: `channels.signal.mediaMaxMb`（デフォルト8）。
- `channels.signal.ignoreAttachments`でメディアのダウンロードをスキップできます。
- グループ履歴コンテキストは`channels.signal.historyLimit`（または`channels.signal.accounts.*.historyLimit`）を使用し、`messages.groupChat.historyLimit`にフォールバックします。`0`で無効（デフォルト50）。

## タイピング + 既読確認

- **タイピングインジケーター**: OpenClawは`signal-cli sendTyping`でタイピングシグナルを送信し、返信中は更新します。
- **既読確認**: `channels.signal.sendReadReceipts`がtrueの場合、OpenClawは許可されたDMの既読確認を転送します。
- signal-cliはグループの既読確認を公開しません。

## リアクション（messageツール）

- `channel=signal`で`message action=react`を使用します。
- ターゲット: 送信者のE.164またはUUID（ペアリング出力から`uuid:<id>`を使用; 裸のUUIDも機能します）。
- `messageId`はリアクション対象のメッセージのSignalタイムスタンプです。
- グループのリアクションには`targetAuthor`または`targetAuthorUuid`が必要です。

例:

```
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

設定:

- `channels.signal.actions.reactions`: リアクションアクションの有効化/無効化（デフォルトtrue）。
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive`。
  - `off`/`ack`はエージェントのリアクションを無効化します（messageツールの`react`はエラーになります）。
  - `minimal`/`extensive`はエージェントのリアクションを有効化してガイダンスレベルを設定します。
- アカウントごとのオーバーライド: `channels.signal.accounts.<id>.actions.reactions`、`channels.signal.accounts.<id>.reactionLevel`。

## デリバリーターゲット（CLI/cron）

- DM: `signal:+15551234567`（またはプレーンE.164）。
- UUID DM: `uuid:<id>`（または裸のUUID）。
- グループ: `signal:group:<groupId>`。
- ユーザー名: `username:<name>`（Signalアカウントがサポートしている場合）。

## トラブルシューティング

まずこのラダーを実行する:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

必要に応じてDMペアリング状態を確認する:

```bash
openclaw pairing list signal
```

よくある失敗:

- デーモンに到達できるが返信なし: アカウント/デーモン設定（`httpUrl`、`account`）と受信モードを確認してください。
- DMが無視される: 送信者がペアリング承認待ちです。
- グループメッセージが無視される: グループ送信者/メンションゲーティングがデリバリーをブロックしています。
- 編集後の設定検証エラー: `openclaw doctor --fix`を実行してください。
- 診断でSignalが欠落: `channels.signal.enabled: true`を確認してください。

追加チェック:

```bash
openclaw pairing list signal
pgrep -af signal-cli
grep -i "signal" "/tmp/openclaw/openclaw-$(date +%Y-%m-%d).log" | tail -20
```

トリアージフロー: [/channels/troubleshooting](/channels/troubleshooting)。

## セキュリティノート

- `signal-cli`はアカウントキーをローカルに保存します（通常`~/.local/share/signal-cli/data/`）。
- サーバーの移行または再構築前にSignalアカウントの状態をバックアップしてください。
- `channels.signal.dmPolicy: "pairing"`を維持してください（より広いDMアクセスを明示的に望む場合を除く）。
- SMS検証は登録または回復フローにのみ必要ですが、番号/アカウントの制御を失うと再登録が複雑になる場合があります。

## 設定リファレンス（Signal）

完全な設定: [設定](/gateway/configuration)

プロバイダーオプション:

- `channels.signal.enabled`: チャンネル起動の有効化/無効化。
- `channels.signal.account`: ボットアカウントのE.164。
- `channels.signal.cliPath`: `signal-cli`へのパス。
- `channels.signal.httpUrl`: 完全なデーモンURL（host/portをオーバーライド）。
- `channels.signal.httpHost`、`channels.signal.httpPort`: デーモンバインド（デフォルト127.0.0.1:8080）。
- `channels.signal.autoStart`: デーモンを自動スポーン（`httpUrl`が未設定の場合デフォルトtrue）。
- `channels.signal.startupTimeoutMs`: 起動待機タイムアウト（ミリ秒、最大120000）。
- `channels.signal.receiveMode`: `on-start | manual`。
- `channels.signal.ignoreAttachments`: 添付ファイルのダウンロードをスキップ。
- `channels.signal.ignoreStories`: デーモンからのストーリーを無視。
- `channels.signal.sendReadReceipts`: 既読確認を転送。
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled`（デフォルト: pairing）。
- `channels.signal.allowFrom`: DMのallowlist（E.164または`uuid:<id>`）。`open`には`"*"`が必要。Signalにはユーザー名がありません; 電話番号/UUID IDを使用してください。
- `channels.signal.groupPolicy`: `open | allowlist | disabled`（デフォルト: allowlist）。
- `channels.signal.groupAllowFrom`: グループ送信者のallowlist。
- `channels.signal.historyLimit`: コンテキストとして含める最大グループメッセージ数（0で無効）。
- `channels.signal.dmHistoryLimit`: ユーザーターン単位のDM履歴制限。ユーザーごとのオーバーライド: `channels.signal.dms["<phone_or_uuid>"].historyLimit`。
- `channels.signal.textChunkLimit`: 送信チャンクサイズ（文字数）。
- `channels.signal.chunkMode`: `length`（デフォルト）または`newline`（長さチャンキングの前に空白行で分割）。
- `channels.signal.mediaMaxMb`: 受信/送信のメディアキャップ（MB）。

関連グローバルオプション:

- `agents.list[].groupChat.mentionPatterns`（Signalはネイティブのメンションをサポートしません）。
- `messages.groupChat.mentionPatterns`（グローバルフォールバック）。
- `messages.responsePrefix`。
