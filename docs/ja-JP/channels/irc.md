---
title: IRC
description: OpenClaw を IRC チャンネルとダイレクトメッセージに接続します。
summary: "IRC プラグインのセットアップ、アクセス制御、およびトラブルシューティング"
read_when:
  - OpenClaw を IRC チャンネルまたは DM に接続したい場合
  - IRC の許可リスト、グループポリシー、またはメンションゲーティングを設定する場合
---

IRC チャンネル（`#room`）とダイレクトメッセージで OpenClaw を使用したい場合に IRC を利用します。
IRC は拡張プラグインとして提供されていますが、メイン設定の `channels.irc` で設定します。

## クイックスタート

1. `~/.openclaw/openclaw.json` で IRC 設定を有効にします。
2. 最低限以下を設定します:

```json
{
  "channels": {
    "irc": {
      "enabled": true,
      "host": "irc.libera.chat",
      "port": 6697,
      "tls": true,
      "nick": "openclaw-bot",
      "channels": ["#openclaw"]
    }
  }
}
```

3. ゲートウェイを起動/再起動します:

```bash
openclaw gateway run
```

## セキュリティのデフォルト

- `channels.irc.dmPolicy` のデフォルトは `"pairing"` です。
- `channels.irc.groupPolicy` のデフォルトは `"allowlist"` です。
- `groupPolicy="allowlist"` の場合、`channels.irc.groups` で許可するチャンネルを定義します。
- 意図的にプレーンテキスト転送を許可しない限り、TLS（`channels.irc.tls=true`）を使用してください。

## アクセス制御

IRC チャンネルには 2 つの別々の「ゲート」があります:

1. **チャンネルアクセス**（`groupPolicy` + `groups`）: ボットがチャンネルからのメッセージを受け付けるかどうか。
2. **送信者アクセス**（`groupAllowFrom` / チャンネルごとの `groups["#channel"].allowFrom`）: そのチャンネル内でボットをトリガーできる人物。

設定キー:

- DM 許可リスト（DM 送信者アクセス）: `channels.irc.allowFrom`
- グループ送信者許可リスト（チャンネル送信者アクセス）: `channels.irc.groupAllowFrom`
- チャンネルごとのコントロール（チャンネル + 送信者 + メンションルール）: `channels.irc.groups["#channel"]`
- `channels.irc.groupPolicy="open"` は未設定のチャンネルを許可します（**デフォルトでも引き続きメンションゲートが有効**）

許可リストのエントリは安定した送信者 ID（`nick!user@host`）を使用してください。
ベアニックのマッチングは変更可能であり、`channels.irc.dangerouslyAllowNameMatching: true` の場合のみ有効になります。

### よくある落とし穴: `allowFrom` は DM 用であり、チャンネル用ではありません

以下のようなログが表示される場合:

- `irc: drop group sender alice!ident@host (policy=allowlist)`

...これは送信者が**グループ/チャンネル**メッセージに許可されていないことを意味します。以下のいずれかで修正します:

- `channels.irc.groupAllowFrom` を設定する（すべてのチャンネルのグローバル設定）、または
- チャンネルごとの送信者許可リストを設定する: `channels.irc.groups["#channel"].allowFrom`

例（`#tuirc-dev` 内の誰でもボットと話せるようにする）:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": { allowFrom: ["*"] },
      },
    },
  },
}
```

## 返信トリガー（メンション）

チャンネルが許可（`groupPolicy` + `groups` 経由）され、送信者が許可されていても、OpenClaw はデフォルトでグループコンテキストで**メンションゲーティング**を使用します。

つまり、メッセージにボットに一致するメンションパターンが含まれていない限り、`drop channel … (missing-mention)` というログが表示される場合があります。

IRC チャンネルでボットが**メンションなしで返信**するようにするには、そのチャンネルのメンションゲーティングを無効にします:

```json5
{
  channels: {
    irc: {
      groupPolicy: "allowlist",
      groups: {
        "#tuirc-dev": {
          requireMention: false,
          allowFrom: ["*"],
        },
      },
    },
  },
}
```

または**すべての** IRC チャンネルを許可し（チャンネルごとの許可リストなし）、メンションなしで返信するようにする場合:

```json5
{
  channels: {
    irc: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: false, allowFrom: ["*"] },
      },
    },
  },
}
```

## セキュリティノート（パブリックチャンネルに推奨）

パブリックチャンネルで `allowFrom: ["*"]` を許可すると、誰でもボットにプロンプトできます。
リスクを軽減するために、そのチャンネルのツールを制限してください。

### チャンネル内の全員に同じツール

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          tools: {
            deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
          },
        },
      },
    },
  },
}
```

### 送信者ごとに異なるツール（オーナーはより多くの権限）

`toolsBySender` を使用して `"*"` に厳しいポリシーを、自分のニックにより緩いポリシーを適用します:

```json5
{
  channels: {
    irc: {
      groups: {
        "#tuirc-dev": {
          allowFrom: ["*"],
          toolsBySender: {
            "*": {
              deny: ["group:runtime", "group:fs", "gateway", "nodes", "cron", "browser"],
            },
            "id:eigen": {
              deny: ["gateway", "nodes", "cron"],
            },
          },
        },
      },
    },
  },
}
```

注意:

- `toolsBySender` のキーは IRC 送信者 ID 値に `id:` プレフィックスを使用してください:
  `id:eigen` またはより強いマッチングには `id:eigen!~eigen@174.127.248.171`。
- レガシーのプレフィックスなしのキーも引き続き受け付けられ、`id:` としてのみマッチします。
- 最初にマッチした送信者ポリシーが優先されます。`"*"` はワイルドカードのフォールバックです。

グループアクセスとメンションゲーティングの詳細（およびそれらの相互作用）については、[/channels/groups](/channels/groups) を参照してください。

## NickServ

接続後に NickServ で認証するには:

```json
{
  "channels": {
    "irc": {
      "nickserv": {
        "enabled": true,
        "service": "NickServ",
        "password": "your-nickserv-password"
      }
    }
  }
}
```

接続時の一度限りの登録（オプション）:

```json
{
  "channels": {
    "irc": {
      "nickserv": {
        "register": true,
        "registerEmail": "bot@example.com"
      }
    }
  }
}
```

ニックが登録されたら `register` を無効にして、繰り返し REGISTER が試みられないようにしてください。

## 環境変数

デフォルトアカウントのサポート:

- `IRC_HOST`
- `IRC_PORT`
- `IRC_TLS`
- `IRC_NICK`
- `IRC_USERNAME`
- `IRC_REALNAME`
- `IRC_PASSWORD`
- `IRC_CHANNELS`（カンマ区切り）
- `IRC_NICKSERV_PASSWORD`
- `IRC_NICKSERV_REGISTER_EMAIL`

## トラブルシューティング

- ボットが接続しているのにチャンネルで返信しない場合は、`channels.irc.groups` を確認し、メンションゲーティングがメッセージをドロップしているかどうか（`missing-mention`）を確認してください。ピングなしで返信させたい場合は、そのチャンネルに `requireMention:false` を設定してください。
- ログインが失敗する場合は、ニックの可用性とサーバーパスワードを確認してください。
- カスタムネットワークで TLS が失敗する場合は、ホスト/ポートと証明書の設定を確認してください。
