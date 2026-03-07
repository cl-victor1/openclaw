---
summary: "`openclaw dashboard` のCLIリファレンス（コントロールUIを開く）"
read_when:
  - 現在のトークンを使用してコントロールUIを開きたいとき
  - ブラウザを起動せずにURLを表示したいとき
title: "dashboard"
---

# `openclaw dashboard`

現在の認証情報を使用してコントロールUIを開きます。

```bash
openclaw dashboard
openclaw dashboard --no-open
```

注意事項：

- `dashboard` は可能な場合、設定済みの `gateway.auth.token` SecretRefを解決します。
- SecretRefで管理されたトークン（解決済みまたは未解決）の場合、`dashboard` はターミナル出力、クリップボード履歴、またはブラウザ起動引数で外部シークレットが露出するのを避けるため、トークンを含まないURLを表示/コピー/開きます。
- `gateway.auth.token` がSecretRefで管理されているがこのコマンドパスで未解決の場合、無効なトークンプレースホルダーを埋め込む代わりに、トークンを含まないURLと明示的な修正ガイダンスを表示します。
