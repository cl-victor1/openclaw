---
summary: "`openclaw configure` のCLIリファレンス（インタラクティブな設定プロンプト）"
read_when:
  - 認証情報、デバイス、またはエージェントのデフォルト設定をインタラクティブに変更したい場合
title: "configure"
---

# `openclaw configure`

認証情報、デバイス、およびエージェントのデフォルト設定をインタラクティブに設定するためのプロンプトです。

注意：**モデル** セクションには、`agents.defaults.models` 許可リスト（`/model` やモデルピッカーに表示される内容）のマルチセレクトが含まれるようになりました。

ヒント：サブコマンドなしの `openclaw config` でも同じウィザードが開きます。非インタラクティブな編集には `openclaw config get|set|unset` を使用してください。

関連リンク：

- ゲートウェイ設定リファレンス：[Configuration](/gateway/configuration)
- Config CLI：[Config](/cli/config)

注意事項：

- ゲートウェイの実行場所の選択は、常に `gateway.mode` を更新します。それだけが必要な場合は、他のセクションを変更せずに「Continue」を選択できます。
- チャンネル向けサービス（Slack・Discord・Matrix・Microsoft Teams）は、セットアップ中にチャンネル・ルームの許可リストを求めます。名前またはIDを入力できます；ウィザードは可能な場合に名前をIDに解決します。
- デーモンのインストール手順を実行する場合、トークン認証にはトークンが必要で、`gateway.auth.token` は SecretRef で管理されます。configure は SecretRef を検証しますが、解決されたプレーンテキストのトークン値をスーパーバイザーサービスの環境メタデータに保存しません。
- トークン認証が必要で、設定された SecretRef が未解決の場合、configure はアクション可能な修復ガイダンスとともにデーモンのインストールをブロックします。
- `gateway.auth.token` と `gateway.auth.password` の両方が設定されており、`gateway.auth.mode` が未設定の場合、configure はモードが明示的に設定されるまでデーモンのインストールをブロックします。

## 使用例

```bash
openclaw configure
openclaw configure --section model --section channels
```
