---
summary: "WhatsAppチャンネルのサポート、アクセス制御、デリバリー動作、および運用"
read_when:
  - WhatsApp/Webチャンネルの動作またはインボックスルーティングを扱うとき
title: "WhatsApp"
---

# WhatsApp（Webチャンネル）

ステータス: WhatsApp Web（Baileys）経由でプロダクション対応済み。ゲートウェイがリンクされたセッションを管理します。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    未知の送信者に対するデフォルトのDMポリシーはペアリングです。
  </Card>
  <Card title="チャンネルトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    チャンネル横断の診断と修復プレイブック。
  </Card>
  <Card title="ゲートウェイ設定" icon="settings" href="/gateway/configuration">
    完全なチャンネル設定パターンと例。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="WhatsAppアクセスポリシーを設定する">

```json5
{
  channels: {
    whatsapp: {
      dmPolicy: "pairing",
      allowFrom: ["+15551234567"],
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

  </Step>

  <Step title="WhatsAppをリンクする（QR）">

```bash
openclaw channels login --channel whatsapp
```

    特定のアカウントの場合:

```bash
openclaw channels login --channel whatsapp --account work
```

  </Step>

  <Step title="ゲートウェイを起動する">

```bash
openclaw gateway
```

  </Step>

  <Step title="最初のペアリングリクエストを承認する（ペアリングモードを使用する場合）">

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    ペアリングリクエストは1時間後に期限切れになります。保留リクエストはチャンネルあたり最大3件です。

  </Step>
</Steps>

<Note>
OpenClawはWhatsAppを可能な限り別の番号で実行することを推奨します。（チャンネルのメタデータとオンボーディングフローはそのセットアップに最適化されていますが、個人番号のセットアップもサポートされています。）
</Note>

## デプロイパターン

<AccordionGroup>
  <Accordion title="専用番号（推奨）">
    これが最もクリーンな運用モードです:

    - OpenClaw専用のWhatsAppアイデンティティ
    - より明確なDM allowlistとルーティング境界
    - セルフチャットの混乱が起きにくい

    最小ポリシーパターン:

    ```json5
    {
      channels: {
        whatsapp: {
          dmPolicy: "allowlist",
          allowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="個人番号フォールバック">
    オンボーディングは個人番号モードをサポートし、セルフチャット対応のベースラインを書き込みます:

    - `dmPolicy: "allowlist"`
    - `allowFrom`に個人番号が含まれる
    - `selfChatMode: true`

    ランタイムでは、セルフチャット保護はリンクされたセルフ番号と`allowFrom`に基づいて機能します。

  </Accordion>

  <Accordion title="WhatsApp Webのみのチャンネルスコープ">
    メッセージングプラットフォームチャンネルは、現在のOpenClawチャンネルアーキテクチャではWhatsApp Webベース（`Baileys`）です。

    組み込みチャットチャンネルレジストリには別のTwilio WhatsAppメッセージングチャンネルはありません。

  </Accordion>
</AccordionGroup>

## ランタイムモデル

- ゲートウェイがWhatsAppソケットと再接続ループを管理します。
- 送信にはターゲットアカウントのアクティブなWhatsAppリスナーが必要です。
- ステータスとブロードキャストチャットは無視されます（`@status`、`@broadcast`）。
- ダイレクトチャットはDMセッションルール（`session.dmScope`; デフォルト`main`でDMをエージェントメインセッションに集約）を使用します。
- グループセッションは分離されます（`agent:<agentId>:whatsapp:group:<jid>`）。

## アクセス制御とアクティベーション

<Tabs>
  <Tab title="DMポリシー">
    `channels.whatsapp.dmPolicy`はダイレクトチャットアクセスを制御します:

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`allowFrom`に`"*"`が必要）
    - `disabled`

    `allowFrom`はE.164形式の番号を受け入れます（内部で正規化）。

    マルチアカウントのオーバーライド: `channels.whatsapp.accounts.<id>.dmPolicy`（および`allowFrom`）はそのアカウントのチャンネルレベルのデフォルトより優先されます。

    ランタイム動作の詳細:

    - ペアリングはチャンネルallow-storeに永続化され、設定された`allowFrom`とマージされます
    - allowlistが設定されていない場合、リンクされたセルフ番号はデフォルトで許可されます
    - アウトバウンドの`fromMe` DMは自動ペアリングされません

  </Tab>

  <Tab title="グループポリシーとallowlist">
    グループアクセスには2つのレイヤーがあります:

    1. **グループメンバーシップallowlist**（`channels.whatsapp.groups`）
       - `groups`が省略された場合、すべてのグループが対象となります
       - `groups`が存在する場合、グループallowlistとして機能します（`"*"`を許可）

    2. **グループ送信者ポリシー**（`channels.whatsapp.groupPolicy` + `groupAllowFrom`）
       - `open`: 送信者allowlistをバイパス
       - `allowlist`: 送信者が`groupAllowFrom`（または`*`）と一致する必要があります
       - `disabled`: すべてのグループインバウンドをブロック

    送信者allowlistのフォールバック:

    - `groupAllowFrom`が未設定の場合、利用可能な`allowFrom`にフォールバックします
    - 送信者allowlistはメンション/返信アクティベーションの前に評価されます

    注: `channels.whatsapp`ブロックがまったく存在しない場合、`channels.defaults.groupPolicy`が設定されていても、ランタイムのグループポリシーフォールバックは`allowlist`（警告ログ付き）になります。

  </Tab>

  <Tab title="メンションと/activation">
    グループの返信はデフォルトでメンションが必要です。

    メンション検出には以下が含まれます:

    - ボットアイデンティティの明示的なWhatsAppメンション
    - 設定されたメンション正規表現パターン（`agents.list[].groupChat.mentionPatterns`、フォールバック`messages.groupChat.mentionPatterns`）
    - 暗黙の「ボットへの返信」検出（返信送信者がボットアイデンティティと一致）

    セキュリティノート:

    - 引用/返信はメンションゲーティングを満たすだけで、送信者の認証を**付与しません**
    - `groupPolicy: "allowlist"`では、allowlistされていない送信者はallowlistされたユーザーのメッセージに返信しても引き続きブロックされます

    セッションレベルのアクティベーションコマンド:

    - `/activation mention`
    - `/activation always`

    `activation`はセッション状態を更新します（グローバル設定ではありません）。オーナーゲート機能です。

  </Tab>
</Tabs>

## 個人番号とセルフチャットの動作

リンクされたセルフ番号が`allowFrom`にも存在する場合、WhatsAppセルフチャット保護が有効になります:

- セルフチャットターンの既読確認をスキップ
- 自分自身にpingするメンション-JIDの自動トリガー動作を無視
- `messages.responsePrefix`が未設定の場合、セルフチャットの返信はデフォルトで`[{identity.name}]`または`[openclaw]`になります

## メッセージの正規化とコンテキスト

<AccordionGroup>
  <Accordion title="インバウンドエンベロープ + 返信コンテキスト">
    受信WhatsAppメッセージは共有インバウンドエンベロープにラップされます。

    引用された返信が存在する場合、コンテキストはこの形式で追加されます:

    ```text
    [Replying to <sender> id:<stanzaId>]
    <quoted body or media placeholder>
    [/Replying]
    ```

    返信メタデータフィールドも利用可能な場合は入力されます（`ReplyToId`、`ReplyToBody`、`ReplyToSender`、送信者JID/E.164）。

  </Accordion>

  <Accordion title="メディアプレースホルダーと位置情報/連絡先の抽出">
    メディアのみのインバウンドメッセージは以下のようなプレースホルダーで正規化されます:

    - `<media:image>`
    - `<media:video>`
    - `<media:audio>`
    - `<media:document>`
    - `<media:sticker>`

    位置情報と連絡先のペイロードはルーティング前にテキストコンテキストに正規化されます。

  </Accordion>

  <Accordion title="保留中のグループ履歴インジェクション">
    グループでは、未処理のメッセージがバッファリングされ、ボットが最終的にトリガーされたときにコンテキストとして注入されます。

    - デフォルト制限: `50`
    - 設定: `channels.whatsapp.historyLimit`
    - フォールバック: `messages.groupChat.historyLimit`
    - `0`で無効

    注入マーカー:

    - `[Chat messages since your last reply - for context]`
    - `[Current message - respond to this]`

  </Accordion>

  <Accordion title="既読確認">
    既読確認は受け入れられたインバウンドWhatsAppメッセージに対してデフォルトで有効です。

    グローバルに無効化:

    ```json5
    {
      channels: {
        whatsapp: {
          sendReadReceipts: false,
        },
      },
    }
    ```

    アカウントごとのオーバーライド:

    ```json5
    {
      channels: {
        whatsapp: {
          accounts: {
            work: {
              sendReadReceipts: false,
            },
          },
        },
      },
    }
    ```

    セルフチャットターンはグローバルに有効であっても既読確認をスキップします。

  </Accordion>
</AccordionGroup>

## デリバリー、チャンキング、メディア

<AccordionGroup>
  <Accordion title="テキストチャンキング">
    - デフォルトのチャンク制限: `channels.whatsapp.textChunkLimit = 4000`
    - `channels.whatsapp.chunkMode = "length" | "newline"`
    - `newline`モードは段落境界（空白行）を優先し、その後長さ安全なチャンキングにフォールバックします
  </Accordion>

  <Accordion title="アウトバウンドメディアの動作">
    - 画像、動画、音声（PTTボイスノート）、ドキュメントペイロードをサポート
    - `audio/ogg`はボイスノート互換性のために`audio/ogg; codecs=opus`に書き換えられます
    - アニメーションGIFの再生は動画送信で`gifPlayback: true`によりサポートされます
    - マルチメディア返信ペイロードを送信する際、キャプションは最初のメディアアイテムに適用されます
    - メディアソースはHTTP(S)、`file://`、またはローカルパスが可能です
  </Accordion>

  <Accordion title="メディアサイズ制限とフォールバック動作">
    - インバウンドメディア保存キャップ: `channels.whatsapp.mediaMaxMb`（デフォルト`50`）
    - アウトバウンドメディア送信キャップ: `channels.whatsapp.mediaMaxMb`（デフォルト`50`）
    - アカウントごとのオーバーライドは`channels.whatsapp.accounts.<accountId>.mediaMaxMb`を使用
    - 画像は制限に合わせて自動最適化されます（リサイズ/品質スイープ）
    - メディア送信失敗時、最初のアイテムのフォールバックとしてレスポンスを無音でドロップする代わりにテキスト警告が送信されます
  </Accordion>
</AccordionGroup>

## 確認リアクション

WhatsAppはインバウンド受信時に`channels.whatsapp.ackReaction`で即時のackリアクションをサポートします。

```json5
{
  channels: {
    whatsapp: {
      ackReaction: {
        emoji: "👀",
        direct: true,
        group: "mentions", // always | mentions | never
      },
    },
  },
}
```

動作ノート:

- インバウンドが受け入れられた直後（返信前）に送信されます
- 失敗はログに記録されますが、通常の返信デリバリーをブロックしません
- グループモード`mentions`はメンションでトリガーされたターンにリアクションします; グループアクティベーション`always`はこのチェックのバイパスとして機能します
- WhatsAppは`channels.whatsapp.ackReaction`を使用します（レガシーの`messages.ackReaction`はここでは使用されません）

## マルチアカウントと認証情報

<AccordionGroup>
  <Accordion title="アカウントの選択とデフォルト">
    - アカウントIDは`channels.whatsapp.accounts`から来ます
    - デフォルトのアカウント選択: 存在する場合は`default`、それ以外は最初に設定されたアカウントID（ソート順）
    - アカウントIDは検索のために内部で正規化されます
  </Accordion>

  <Accordion title="認証情報パスとレガシー互換性">
    - 現在の認証パス: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
    - バックアップファイル: `creds.json.bak`
    - `~/.openclaw/credentials/`のレガシーデフォルト認証は、デフォルトアカウントフローでまだ認識/移行されます
  </Accordion>

  <Accordion title="ログアウト動作">
    `openclaw channels logout --channel whatsapp [--account <id>]`はそのアカウントのWhatsApp認証状態をクリアします。

    レガシー認証ディレクトリでは、`oauth.json`は保持されながらBaileysの認証ファイルが削除されます。

  </Accordion>
</AccordionGroup>

## ツール、アクション、設定書き込み

- エージェントツールサポートにはWhatsAppリアクションアクション（`react`）が含まれます。
- アクションゲート:
  - `channels.whatsapp.actions.reactions`
  - `channels.whatsapp.actions.polls`
- チャンネル起点の設定書き込みはデフォルトで有効です（`channels.whatsapp.configWrites=false`で無効化）。

## トラブルシューティング

<AccordionGroup>
  <Accordion title="リンクされていない（QRが必要）">
    症状: チャンネルステータスがリンクされていないと報告する。

    修正:

    ```bash
    openclaw channels login --channel whatsapp
    openclaw channels status
    ```

  </Accordion>

  <Accordion title="リンクされているが切断/再接続ループ">
    症状: リンクされたアカウントで繰り返し切断または再接続の試みが発生する。

    修正:

    ```bash
    openclaw doctor
    openclaw logs --follow
    ```

    必要に応じて`channels login`で再リンクしてください。

  </Accordion>

  <Accordion title="送信時にアクティブなリスナーがない">
    ターゲットアカウントのアクティブなゲートウェイリスナーが存在しない場合、アウトバウンド送信は素早く失敗します。

    ゲートウェイが実行中でアカウントがリンクされていることを確認してください。

  </Accordion>

  <Accordion title="グループメッセージが予期せず無視される">
    この順序で確認する:

    - `groupPolicy`
    - `groupAllowFrom` / `allowFrom`
    - `groups` allowlistエントリ
    - メンションゲーティング（`requireMention` + メンションパターン）
    - `openclaw.json`（JSON5）の重複キー: 後のエントリが前のエントリをオーバーライドするため、スコープごとに単一の`groupPolicy`を維持してください

  </Accordion>

  <Accordion title="Bunランタイムの警告">
    WhatsAppゲートウェイランタイムはNodeを使用する必要があります。BunはWhatsApp/Telegramゲートウェイの安定した動作には非互換として警告されます。
  </Accordion>
</AccordionGroup>

## 設定リファレンスポインター

主要リファレンス:

- [設定リファレンス - WhatsApp](/gateway/configuration-reference#whatsapp)

重要なWhatsAppフィールド:

- アクセス: `dmPolicy`、`allowFrom`、`groupPolicy`、`groupAllowFrom`、`groups`
- デリバリー: `textChunkLimit`、`chunkMode`、`mediaMaxMb`、`sendReadReceipts`、`ackReaction`
- マルチアカウント: `accounts.<id>.enabled`、`accounts.<id>.authDir`、アカウントレベルのオーバーライド
- 運用: `configWrites`、`debounceMs`、`web.enabled`、`web.heartbeatSeconds`、`web.reconnect.*`
- セッション動作: `session.dmScope`、`historyLimit`、`dmHistoryLimit`、`dms.<id>.historyLimit`

## 関連項目

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [マルチエージェントルーティング](/concepts/multi-agent)
- [トラブルシューティング](/channels/troubleshooting)
