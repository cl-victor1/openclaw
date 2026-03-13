---
summary: "ZaloボットのサポートステータスおよびBot API機能と設定"
read_when:
  - Zalo機能やウェブフックの開発時
title: "Zalo"
---

# Zalo（Bot API）

ステータス：実験的。DMはサポートされています。グループ処理は明示的なグループポリシー制御で利用可能です。

## プラグインが必要です

ZaloはプラグインとしてShipされており、コアインストールにはバンドルされていません。

- CLIからインストール：`openclaw plugins install @openclaw/zalo`
- またはオンボーディング中に**Zalo**を選択し、インストールプロンプトを確認します
- 詳細：[Plugins](/tools/plugin)

## クイックセットアップ（初心者向け）

1. Zaloプラグインをインストールします：
   - ソースチェックアウトから：`openclaw plugins install ./extensions/zalo`
   - npm（公開済みの場合）から：`openclaw plugins install @openclaw/zalo`
   - またはオンボーディングで**Zalo**を選択し、インストールプロンプトを確認します
2. トークンを設定します：
   - 環境変数：`ZALO_BOT_TOKEN=...`
   - または設定：`channels.zalo.botToken: "..."`
3. ゲートウェイを再起動します（またはオンボーディングを完了します）。
4. DMアクセスはデフォルトでペアリングです。初回接触時にペアリングコードを承認します。

最小限の設定：

```json5
{
  channels: {
    zalo: {
      enabled: true,
      botToken: "12345689:abc-xyz",
      dmPolicy: "pairing",
    },
  },
}
```

## 概要

Zaloはベトナム向けのメッセージングアプリです。そのBot APIにより、ゲートウェイが1対1の会話用ボットを実行できます。
Zaloへのルーティングが確定的に必要なサポートや通知に適しています。

- ゲートウェイが所有するZalo Bot APIチャンネル。
- 確定的なルーティング：返信はZaloに戻ります。モデルがチャンネルを選択することはありません。
- DMはエージェントのメインセッションを共有します。
- グループはポリシー制御（`groupPolicy` + `groupAllowFrom`）でサポートされており、デフォルトはフェイルクローズのアローリスト動作です。

## セットアップ（高速パス）

### 1）ボットトークンを作成する（Zalo Bot Platform）

1. [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com)にアクセスしてサインインします。
2. 新しいボットを作成し、設定を構成します。
3. ボットトークンをコピーします（形式：`12345689:abc-xyz`）。

### 2）トークンを設定する（環境変数または設定ファイル）

例：

```json5
{
  channels: {
    zalo: {
      enabled: true,
      botToken: "12345689:abc-xyz",
      dmPolicy: "pairing",
    },
  },
}
```

環境変数オプション：`ZALO_BOT_TOKEN=...`（デフォルトアカウントのみ対応）。

マルチアカウントサポート：アカウントごとのトークンとオプションの`name`を含む`channels.zalo.accounts`を使用します。

3. ゲートウェイを再起動します。Zaloはトークンが解決されると（環境変数または設定ファイル）起動します。
4. DMアクセスはデフォルトでペアリングです。ボットが最初に接触されたときにコードを承認します。

## 動作の仕組み

- 受信メッセージは、メディアプレースホルダーを含む共有チャンネルエンベロープに正規化されます。
- 返信は常に同じZaloチャットにルーティングされます。
- デフォルトはロングポーリング。ウェブフックモードは`channels.zalo.webhookUrl`で利用可能です。

## 制限

- アウトバウンドテキストは2000文字にチャンク分割されます（Zalo APIの制限）。
- メディアのダウンロード/アップロードは`channels.zalo.mediaMaxMb`（デフォルト5）で制限されます。
- ストリーミングは2000文字の制限によりデフォルトでブロックされます。

## アクセス制御（DM）

### DMアクセス

- デフォルト：`channels.zalo.dmPolicy = "pairing"`。未知の送信者はペアリングコードを受け取り、承認されるまでメッセージは無視されます（コードは1時間後に失効）。
- 承認方法：
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
- ペアリングはデフォルトのトークン交換です。詳細：[Pairing](/channels/pairing)
- `channels.zalo.allowFrom`は数値ユーザーIDを受け付けます（ユーザー名の検索は利用不可）。

## アクセス制御（グループ）

- `channels.zalo.groupPolicy`はグループの受信処理を制御します：`open | allowlist | disabled`。
- デフォルト動作はフェイルクローズ：`allowlist`。
- `channels.zalo.groupAllowFrom`はグループでボットをトリガーできる送信者IDを制限します。
- `groupAllowFrom`が未設定の場合、Zaloは送信者チェックに`allowFrom`にフォールバックします。
- `groupPolicy: "disabled"`はすべてのグループメッセージをブロックします。
- `groupPolicy: "open"`は任意のグループメンバーを許可します（メンション必須）。
- ランタイムの注意：`channels.zalo`が完全に欠落している場合、ランタイムは安全のため`groupPolicy="allowlist"`にフォールバックします。

## ロングポーリングとウェブフック

- デフォルト：ロングポーリング（公開URLは不要）。
- ウェブフックモード：`channels.zalo.webhookUrl`と`channels.zalo.webhookSecret`を設定します。
  - ウェブフックシークレットは8〜256文字である必要があります。
  - ウェブフックURLはHTTPSを使用する必要があります。
  - Zaloは検証のために`X-Bot-Api-Secret-Token`ヘッダーでイベントを送信します。
  - ゲートウェイHTTPは`channels.zalo.webhookPath`でウェブフックリクエストを処理します（デフォルトはウェブフックURLパス）。
  - リクエストは`Content-Type: application/json`（または`+json`メディアタイプ）を使用する必要があります。
  - 重複イベント（`event_name + message_id`）は短いリプレイウィンドウ内で無視されます。
  - バーストトラフィックはパス/ソースごとにレート制限され、HTTP 429を返すことがあります。

**注意：** getUpdates（ポーリング）とウェブフックはZalo APIドキュメントに従って相互排他的です。

## サポートされているメッセージタイプ

- **テキストメッセージ**：2000文字チャンキングで完全サポート。
- **画像メッセージ**：インバウンド画像のダウンロードと処理。`sendPhoto`で画像を送信。
- **スタンプ**：ログされますが完全には処理されません（エージェント応答なし）。
- **非サポートタイプ**：ログされます（例：保護されたユーザーからのメッセージ）。

## 機能一覧

| 機能             | ステータス                                    |
| --------------- | -------------------------------------------- |
| ダイレクトメッセージ | ✅ サポート済み                              |
| グループ          | ⚠️ ポリシー制御でサポート（デフォルトはアローリスト） |
| メディア（画像）   | ✅ サポート済み                              |
| リアクション       | ❌ 非サポート                                |
| スレッド          | ❌ 非サポート                                |
| ポーリング        | ❌ 非サポート                                |
| ネイティブコマンド  | ❌ 非サポート                                |
| ストリーミング     | ⚠️ ブロック済み（2000文字制限）              |

## 配信ターゲット（CLI/cron）

- チャットIDをターゲットとして使用します。
- 例：`openclaw message send --channel zalo --target 123456789 --message "hi"`。

## トラブルシューティング

**ボットが応答しない：**

- トークンが有効かどうかを確認します：`openclaw channels status --probe`
- 送信者が承認済みかどうかを確認します（ペアリングまたはallowFrom）
- ゲートウェイログを確認します：`openclaw logs --follow`

**ウェブフックがイベントを受信しない：**

- ウェブフックURLがHTTPSを使用していることを確認します
- シークレットトークンが8〜256文字であることを確認します
- 設定されたパスでゲートウェイHTTPエンドポイントに到達できることを確認します
- getUpdatesポーリングが実行されていないことを確認します（相互排他的）

## 設定リファレンス（Zalo）

完全な設定：[Configuration](/gateway/configuration)

プロバイダーオプション：

- `channels.zalo.enabled`：チャンネルの起動を有効/無効にします。
- `channels.zalo.botToken`：Zalo Bot Platformのボットトークン。
- `channels.zalo.tokenFile`：通常のファイルパスからトークンを読み取ります。シンボリックリンクは拒否されます。
- `channels.zalo.dmPolicy`：`pairing | allowlist | open | disabled`（デフォルト：pairing）。
- `channels.zalo.allowFrom`：DMアローリスト（ユーザーID）。`open`には`"*"`が必要です。ウィザードは数値IDを尋ねます。
- `channels.zalo.groupPolicy`：`open | allowlist | disabled`（デフォルト：allowlist）。
- `channels.zalo.groupAllowFrom`：グループ送信者アローリスト（ユーザーID）。未設定の場合は`allowFrom`にフォールバック。
- `channels.zalo.mediaMaxMb`：インバウンド/アウトバウンドメディア上限（MB、デフォルト5）。
- `channels.zalo.webhookUrl`：ウェブフックモードを有効にします（HTTPSが必要）。
- `channels.zalo.webhookSecret`：ウェブフックシークレット（8〜256文字）。
- `channels.zalo.webhookPath`：ゲートウェイHTTPサーバーのウェブフックパス。
- `channels.zalo.proxy`：APIリクエスト用のプロキシURL。

マルチアカウントオプション：

- `channels.zalo.accounts.<id>.botToken`：アカウントごとのトークン。
- `channels.zalo.accounts.<id>.tokenFile`：アカウントごとの通常トークンファイル。シンボリックリンクは拒否されます。
- `channels.zalo.accounts.<id>.name`：表示名。
- `channels.zalo.accounts.<id>.enabled`：アカウントの有効/無効。
- `channels.zalo.accounts.<id>.dmPolicy`：アカウントごとのDMポリシー。
- `channels.zalo.accounts.<id>.allowFrom`：アカウントごとのアローリスト。
- `channels.zalo.accounts.<id>.groupPolicy`：アカウントごとのグループポリシー。
- `channels.zalo.accounts.<id>.groupAllowFrom`：アカウントごとのグループ送信者アローリスト。
- `channels.zalo.accounts.<id>.webhookUrl`：アカウントごとのウェブフックURL。
- `channels.zalo.accounts.<id>.webhookSecret`：アカウントごとのウェブフックシークレット。
- `channels.zalo.accounts.<id>.webhookPath`：アカウントごとのウェブフックパス。
- `channels.zalo.accounts.<id>.proxy`：アカウントごとのプロキシURL。
