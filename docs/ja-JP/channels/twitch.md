---
summary: "Twitchチャットボットの設定とセットアップ"
read_when:
  - OpenClawのTwitchチャット統合のセットアップ時
title: "Twitch"
---

# Twitch（プラグイン）

IRC接続によるTwitchチャットサポート。OpenClawはTwitchユーザー（ボットアカウント）としてチャンネルのメッセージ受信・送信のために接続します。

## プラグインが必要です

TwitchはプラグインとしてShipされており、コアインストールにはバンドルされていません。

CLIからインストール（npmレジストリ）：

```bash
openclaw plugins install @openclaw/twitch
```

ローカルチェックアウト（gitリポジトリから実行する場合）：

```bash
openclaw plugins install ./extensions/twitch
```

詳細：[Plugins](/tools/plugin)

## クイックセットアップ（初心者向け）

1. ボット用のTwitchアカウントを新規作成します（または既存のアカウントを使用）。
2. 認証情報を生成します：[Twitch Token Generator](https://twitchtokengenerator.com/)
   - **Bot Token**を選択します
   - スコープ`chat:read`と`chat:write`が選択されていることを確認します
   - **Client ID**と**Access Token**をコピーします
3. TwitchユーザーIDを確認します：[https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/)
4. トークンを設定します：
   - 環境変数：`OPENCLAW_TWITCH_ACCESS_TOKEN=...`（デフォルトアカウントのみ）
   - または設定：`channels.twitch.accessToken`
   - 両方が設定されている場合、設定が優先されます（環境変数フォールバックはデフォルトアカウントのみ）。
5. ゲートウェイを起動します。

**⚠️ 重要：** 未承認ユーザーがボットをトリガーするのを防ぐため、アクセス制御（`allowFrom`または`allowedRoles`）を追加してください。`requireMention`はデフォルトで`true`です。

最小限の設定：

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // ボットのTwitchアカウント
      accessToken: "oauth:abc123...", // OAuthアクセストークン（またはOPENCLAW_TWITCH_ACCESS_TOKEN環境変数を使用）
      clientId: "xyz789...", // Token GeneratorのクライアントID
      channel: "vevisk", // 参加するTwitchチャンネルのチャット（必須）
      allowFrom: ["123456789"], // （推奨）自分のTwitchユーザーIDのみ - https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/ で取得
    },
  },
}
```

## 概要

- ゲートウェイが所有するTwitchチャンネル。
- 確定的なルーティング：返信は常にTwitchに戻ります。
- 各アカウントは分離されたセッションキー`agent:<agentId>:twitch:<accountName>`にマップされます。
- `username`はボットのアカウント（認証するアカウント）で、`channel`はどのチャットルームに参加するかです。

## セットアップ（詳細）

### 認証情報を生成する

[Twitch Token Generator](https://twitchtokengenerator.com/)を使用します：

- **Bot Token**を選択します
- スコープ`chat:read`と`chat:write`が選択されていることを確認します
- **Client ID**と**Access Token**をコピーします

手動アプリ登録は不要です。トークンは数時間後に期限切れになります。

### ボットを設定する

**環境変数（デフォルトアカウントのみ）：**

```bash
OPENCLAW_TWITCH_ACCESS_TOKEN=oauth:abc123...
```

**または設定ファイル：**

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
    },
  },
}
```

環境変数と設定の両方が設定されている場合、設定が優先されます。

### アクセス制御（推奨）

```json5
{
  channels: {
    twitch: {
      allowFrom: ["123456789"], // （推奨）自分のTwitchユーザーIDのみ
    },
  },
}
```

ハードアローリストには`allowFrom`を使用してください。ロールベースのアクセスが必要な場合は、代わりに`allowedRoles`を使用します。

**利用可能なロール：** `"moderator"`、`"owner"`、`"vip"`、`"subscriber"`、`"all"`。

**なぜユーザーIDを使うの？** ユーザー名は変更できるため、なりすましが可能です。ユーザーIDは永続的です。

TwitchユーザーIDの確認：[https://www.streamweasels.com/tools/convert-twitch-username-%20to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-%20to-user-id/)（TwitchユーザーIDの変換）

## トークンの更新（オプション）

[Twitch Token Generator](https://twitchtokengenerator.com/)のトークンは自動更新できません。期限切れになったら再生成してください。

自動トークン更新のためには、[Twitch Developer Console](https://dev.twitch.tv/console)で独自のTwitchアプリケーションを作成し、設定に追加します：

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

ボットは期限切れ前にトークンを自動的に更新し、更新イベントをログに記録します。

## マルチアカウントサポート

アカウントごとのトークンを含む`channels.twitch.accounts`を使用します。共有パターンについては[`gateway/configuration`](/gateway/configuration)を参照してください。

例（1つのボットアカウントで2つのチャンネルに参加）：

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "vevisk",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

**注意：** 各アカウントは独自のトークンが必要です（チャンネルごとに1つのトークン）。

## アクセス制御

### ロールベースの制限

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator", "vip"],
        },
      },
    },
  },
}
```

### ユーザーIDによるアローリスト（最も安全）

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowFrom: ["123456789", "987654321"],
        },
      },
    },
  },
}
```

### ロールベースのアクセス（代替手段）

`allowFrom`はハードアローリストです。設定すると、それらのユーザーIDのみが許可されます。
ロールベースのアクセスが必要な場合は、`allowFrom`を設定せず、代わりに`allowedRoles`を設定します：

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

### @メンション要件を無効にする

デフォルトでは、`requireMention`は`true`です。すべてのメッセージに返答するには無効にします：

```json5
{
  channels: {
    twitch: {
      accounts: {
        default: {
          requireMention: false,
        },
      },
    },
  },
}
```

## トラブルシューティング

まず診断コマンドを実行します：

```bash
openclaw doctor
openclaw channels status --probe
```

### ボットがメッセージに応答しない

**アクセス制御を確認：** ユーザーIDが`allowFrom`にあることを確認するか、一時的に`allowFrom`を削除して`allowedRoles: ["all"]`を設定してテストします。

**ボットがチャンネルに参加しているか確認：** ボットは`channel`で指定したチャンネルに参加している必要があります。

### トークンの問題

**「接続失敗」または認証エラー：**

- `accessToken`がOAuthアクセストークン値であることを確認します（通常は`oauth:`プレフィックスで始まります）
- トークンに`chat:read`と`chat:write`スコープがあることを確認します
- トークン更新を使用している場合は、`clientSecret`と`refreshToken`が設定されていることを確認します

### トークン更新が機能しない

**ログで更新イベントを確認：**

```
Using env token source for mybot
Access token refreshed for user 123456 (expires in 14400s)
```

「token refresh disabled (no refresh token)」と表示される場合：

- `clientSecret`が提供されていることを確認します
- `refreshToken`が提供されていることを確認します

## 設定

**アカウント設定：**

- `username` - ボットのユーザー名
- `accessToken` - `chat:read`と`chat:write`を持つOAuthアクセストークン
- `clientId` - TwitchクライアントID（Token Generatorまたはアプリから）
- `channel` - 参加するチャンネル（必須）
- `enabled` - このアカウントを有効にする（デフォルト：`true`）
- `clientSecret` - オプション：自動トークン更新のため
- `refreshToken` - オプション：自動トークン更新のため
- `expiresIn` - トークンの有効期限（秒）
- `obtainmentTimestamp` - トークン取得タイムスタンプ
- `allowFrom` - ユーザーIDアローリスト
- `allowedRoles` - ロールベースのアクセス制御（`"moderator" | "owner" | "vip" | "subscriber" | "all"`）
- `requireMention` - @メンションを必須にする（デフォルト：`true`）

**プロバイダーオプション：**

- `channels.twitch.enabled` - チャンネルの起動を有効/無効にする
- `channels.twitch.username` - ボットのユーザー名（シンプルなシングルアカウント設定）
- `channels.twitch.accessToken` - OAuthアクセストークン（シンプルなシングルアカウント設定）
- `channels.twitch.clientId` - TwitchクライアントID（シンプルなシングルアカウント設定）
- `channels.twitch.channel` - 参加するチャンネル（シンプルなシングルアカウント設定）
- `channels.twitch.accounts.<accountName>` - マルチアカウント設定（上記のすべてのアカウントフィールド）

完全な例：

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "vevisk",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      allowedRoles: ["moderator", "vip"],
      accounts: {
        default: {
          username: "mybot",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "your_channel",
          enabled: true,
          clientSecret: "secret123...",
          refreshToken: "refresh456...",
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowFrom: ["123456789", "987654321"],
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## ツールアクション

エージェントは`twitch`をアクションとともに呼び出すことができます：

- `send` - チャンネルにメッセージを送信する

例：

```json5
{
  action: "twitch",
  params: {
    message: "Hello Twitch!",
    to: "#mychannel",
  },
}
```

## 安全性と運用

- **トークンをパスワードのように扱ってください** - トークンをgitにコミットしないでください
- 長期間稼働するボットには**自動トークン更新を使用**してください
- アクセス制御にはユーザー名ではなく**ユーザーIDアローリストを使用**してください
- トークン更新イベントと接続状態の**ログを監視**してください
- **スコープを最小限に留めてください** - `chat:read`と`chat:write`のみをリクエストしてください
- **問題が発生した場合**：他のプロセスがセッションを所有していないことを確認してからゲートウェイを再起動してください

## 制限

- メッセージあたり**500文字**（単語境界で自動チャンク）
- チャンキング前にMarkdownが除去されます
- レート制限なし（Twitchの組み込みレート制限を使用）
