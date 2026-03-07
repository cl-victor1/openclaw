---
summary: "`openclaw tui` のCLIリファレンス（Gatewayに接続するターミナルUI）"
read_when:
  - Gateway用のターミナルUIを使用したい場合（リモート対応）
  - スクリプトからURL/トークン/セッションを渡したい場合
title: "tui"
---

# `openclaw tui`

Gatewayに接続するターミナルUIを開きます。

関連：

- TUIガイド: [TUI](/web/tui)

注意事項：

- `tui` は可能な場合、トークン/パスワード認証のために設定済みのGateway認証 SecretRef を解決します（`env`/`file`/`exec` プロバイダー）。

## 使用例

```bash
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
```
