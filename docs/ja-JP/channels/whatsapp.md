---
summary: "WhatsApp チャンネルのサポート、アクセス制御、配信動作、操作"
read_when:
  - WhatsApp/Web チャンネルの動作や受信トレイルーティングの調査
title: "WhatsApp"
---

# WhatsApp（Web チャンネル）

ステータス: WhatsApp Web（Baileys）経由で本番対応済み。ゲートウェイがリンク済みセッションを管理します。

<CardGroup cols={3}>
  <Card title="ペアリング" icon="link" href="/channels/pairing">
    デフォルトの DM ポリシーは未知の送信者に対してペアリングです。
  </Card>
  <Card title="チャンネルのトラブルシューティング" icon="wrench" href="/channels/troubleshooting">
    クロスチャンネルの診断と修復プレイブック。
  </Card>
  <Card title="ゲートウェイ設定" icon="settings" href="/gateway/configuration">
    チャンネル設定のパターンと例の完全版。
  </Card>
</CardGroup>

## クイックセットアップ

<Steps>
  <Step title="WhatsApp アクセスポリシーを設定する">

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

  <Step title="WhatsApp をリンクする（QR）">

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

  <Step title="最初のペアリングリクエストを承認する（ペアリングモードの場合）">

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp <CODE>
```

    ペアリングリクエストは 1 時間後に期限切れになります。チャンネルごとに保留中のリクエストは最大 3 件です。

  </Step>
</Steps>

<Note>
OpenClaw では、可能な限り WhatsApp を別番号で運用することを推奨しています。（チャンネルのメタデータとオンボーディングフローはそのセットアップ向けに最適化されていますが、個人番号でのセットアップもサポートされています。）
</Note>

## デプロイパターン

<AccordionGroup>
  <Accordion title="専用番号（推奨）">
    これが最もクリーンな運用モードです:

    - OpenClaw 専用の WhatsApp アイデンティティ
    - より明確な DM 許可リストとルーティング境界
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
    - `allowFrom` に個人番号を含む
    - `selfChatMode: true`

    実行時、セルフチャット保護はリンクされた自分の番号と `allowFrom` に基づきます。

  </Accordion>

  <Accordion title="WhatsApp Web のみのチャンネルスコープ">
    メッセージングプラットフォームのチャンネルは、現在の OpenClaw チャンネルアーキテクチャでは WhatsApp Web ベース（`Baileys`）です。

    組み込みチャットチャンネルレジストリには Twilio WhatsApp メッセージングチャンネルは存在しません。

  </Accordion>
</AccordionGroup>

## ランタイムモデル

- ゲートウェイが WhatsApp ソケットと再接続ループを管理します
- 送信には対象アカウントのアクティブな WhatsApp リスナーが必要です
- ステータスとブロードキャストチャットは無視されます（`@status`、`@broadcast`）
- ダイレクトチャットは DM セッションルールを使用します（`session.dmScope`; デフォルト `main` で DM をエージェントのメインセッションに集約）
- グループセッションは分離されます（`agent:<agentId>:whatsapp:group:<jid>`）

## アクセス制御とアクティベーション

<Tabs>
  <Tab title="DM ポリシー">
    `channels.whatsapp.dmPolicy` はダイレクトチャットへのアクセスを制御します:

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`（`allowFrom` に `"*"` を含める必要あり）
    - `disabled`

    `allowFrom` は E.164 形式の番号を受け付けます（内部で正規化されます）。

    マルチアカウントオーバーライド: `channels.whatsapp.accounts.<id>.dmPolicy`（と `allowFrom`）は、そのアカウントのチャンネルレベルのデフォルトより優先されます。

  </Tab>

  <Tab title="グループポリシーと許可リスト">
    `channels.whatsapp.groupPolicy` はグループチャットへのアクセスを制御します:

    - `pairing`（デフォルト）
    - `allowlist`
    - `open`
    - `disabled`

    `groupAllowFrom` はグループ送信者のフィルタリングに使用されます。未設定の場合、`allowFrom` にフォールバックします。

  </Tab>
</Tabs>

## 関連情報

- [ペアリング](/channels/pairing)
- [チャンネルルーティング](/channels/channel-routing)
- [チャンネルのトラブルシューティング](/channels/troubleshooting)
