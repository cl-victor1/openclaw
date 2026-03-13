---
summary: "チャンネルレベルのトラブルシューティングを素早く行うための、チャンネルごとの障害シグネチャと修正方法"
read_when:
  - チャンネルトランスポートが接続済みと表示されているのに返信が失敗する場合
  - 詳細なプロバイダードキュメントの前にチャンネル固有のチェックが必要な場合
title: "チャンネルトラブルシューティング"
---

# チャンネルトラブルシューティング

チャンネルは接続されているが動作がおかしい場合にこのページを使用してください。

## コマンドラダー

まずこの順番で実行してください：

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

正常なベースライン：

- `Runtime: running`
- `RPC probe: ok`
- チャンネルプローブが接続済み/準備完了を示す

## WhatsApp

### WhatsAppの障害シグネチャ

| 症状                                | 最速チェック方法                                   | 修正方法                                               |
| ----------------------------------- | ------------------------------------------------- | ----------------------------------------------------- |
| 接続済みだがDM返信なし               | `openclaw pairing list whatsapp`                  | 送信者を承認するか、DMポリシー/アローリストを変更します。 |
| グループメッセージが無視される        | 設定の`requireMention`とメンションパターンを確認   | ボットをメンションするか、そのグループのメンションポリシーを緩和します。 |
| ランダムな切断/再ログインのループ     | `openclaw channels status --probe` + ログ         | 再ログインし、認証情報ディレクトリが正常かどうか確認します。 |

完全なトラブルシューティング：[/channels/whatsapp#troubleshooting-quick](/channels/whatsapp#troubleshooting-quick)

## Telegram

### Telegramの障害シグネチャ

| 症状                                | 最速チェック方法                                  | 修正方法                                                                         |
| ----------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `/start`後に使えない返信フロー       | `openclaw pairing list telegram`                 | ペアリングを承認するか、DMポリシーを変更します。                                   |
| ボットはオンラインだがグループが無反応 | メンション要件とボットプライバシーモードを確認   | グループ表示のためプライバシーモードを無効にするか、ボットをメンションします。         |
| ネットワークエラーで送信失敗         | Telegram API呼び出し失敗のログを確認             | `api.telegram.org`へのDNS/IPv6/プロキシルーティングを修正します。                  |
| 起動時に`setMyCommands`が拒否される  | ログで`BOT_COMMANDS_TOO_MUCH`を確認              | プラグイン/スキル/カスタムTelegramコマンドを削減するか、ネイティブメニューを無効にします。 |
| アップグレード後にアローリストでブロック | `openclaw security audit`と設定のアローリストを確認 | `openclaw doctor --fix`を実行するか、`@username`を数値の送信者IDに置き換えます。 |

完全なトラブルシューティング：[/channels/telegram#troubleshooting](/channels/telegram#troubleshooting)

## Discord

### Discordの障害シグネチャ

| 症状                              | 最速チェック方法                        | 修正方法                                                    |
| --------------------------------- | -------------------------------------- | ---------------------------------------------------------- |
| ボットはオンラインだがギルドで返信なし | `openclaw channels status --probe`   | ギルド/チャンネルを許可し、メッセージコンテンツインテントを確認します。 |
| グループメッセージが無視される      | メンションゲーティングのドロップログを確認 | ボットをメンションするか、ギルド/チャンネルの`requireMention: false`を設定します。 |
| DM返信がない                      | `openclaw pairing list discord`        | DMペアリングを承認するか、DMポリシーを調整します。            |

完全なトラブルシューティング：[/channels/discord#troubleshooting](/channels/discord#troubleshooting)

## Slack

### Slackの障害シグネチャ

| 症状                                | 最速チェック方法                           | 修正方法                                             |
| ----------------------------------- | ----------------------------------------- | --------------------------------------------------- |
| ソケットモード接続済みだが返信なし   | `openclaw channels status --probe`        | アプリトークン、ボットトークン、必要なスコープを確認します。 |
| DMがブロックされる                   | `openclaw pairing list slack`             | ペアリングを承認するか、DMポリシーを緩和します。        |
| チャンネルメッセージが無視される     | `groupPolicy`とチャンネルアローリストを確認 | チャンネルを許可するか、ポリシーを`open`に切り替えます。 |

完全なトラブルシューティング：[/channels/slack#troubleshooting](/channels/slack#troubleshooting)

## iMessageとBlueBubbles

### iMessageとBlueBubblesの障害シグネチャ

| 症状                              | 最速チェック方法                                            | 修正方法                                              |
| --------------------------------- | ---------------------------------------------------------- | ---------------------------------------------------- |
| 受信イベントなし                  | Webhookサーバーの到達性とアプリ権限を確認                   | Webhook URLまたはBlueBubblesサーバーの状態を修正します。 |
| macOSで送信はできるが受信できない  | MessagesオートメーションのmacOSプライバシー権限を確認       | TCC権限を再付与し、チャンネルプロセスを再起動します。   |
| DM送信者がブロックされる          | `openclaw pairing list imessage`または`openclaw pairing list bluebubbles` | ペアリングを承認するか、アローリストを更新します。 |

完全なトラブルシューティング：

- [/channels/imessage#troubleshooting-macos-privacy-and-security-tcc](/channels/imessage#troubleshooting-macos-privacy-and-security-tcc)
- [/channels/bluebubbles#troubleshooting](/channels/bluebubbles#troubleshooting)

## Signal

### Signalの障害シグネチャ

| 症状                              | 最速チェック方法                            | 修正方法                                                     |
| --------------------------------- | ------------------------------------------ | ----------------------------------------------------------- |
| デーモンは到達可能だがボットが無反応 | `openclaw channels status --probe`        | `signal-cli`デーモンのURL/アカウントと受信モードを確認します。 |
| DMがブロックされる                 | `openclaw pairing list signal`             | 送信者を承認するか、DMポリシーを調整します。                   |
| グループ返信がトリガーされない      | グループアローリストとメンションパターンを確認 | 送信者/グループを追加するか、ゲーティングを緩和します。        |

完全なトラブルシューティング：[/channels/signal#troubleshooting](/channels/signal#troubleshooting)

## Matrix

### Matrixの障害シグネチャ

| 症状                              | 最速チェック方法                              | 修正方法                                              |
| --------------------------------- | -------------------------------------------- | ---------------------------------------------------- |
| ログイン済みだがルームメッセージを無視 | `openclaw channels status --probe`          | `groupPolicy`とルームアローリストを確認します。        |
| DMが処理されない                   | `openclaw pairing list matrix`               | 送信者を承認するか、DMポリシーを調整します。            |
| 暗号化ルームが失敗する              | 暗号モジュールと暗号化設定を確認             | 暗号化サポートを有効にし、ルームに再参加/同期します。   |

完全なトラブルシューティング：[/channels/matrix#troubleshooting](/channels/matrix#troubleshooting)
