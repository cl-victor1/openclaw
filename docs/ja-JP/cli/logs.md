---
summary: "`openclaw logs` のCLIリファレンス（RPC経由でゲートウェイログをテールする）"
read_when:
  - SSHなしでリモートからゲートウェイログをテールしたい場合
  - ツール連携用にJSONログ行が必要な場合
title: "logs"
---

# `openclaw logs`

RPC経由でゲートウェイのファイルログをテールします（リモートモードで動作します）。

関連:

- ロギングの概要: [ロギング](/logging)

## 使用例

```bash
openclaw logs
openclaw logs --follow
openclaw logs --json
openclaw logs --limit 500
openclaw logs --local-time
openclaw logs --follow --local-time
```

`--local-time` を使用すると、タイムスタンプがローカルタイムゾーンで表示されます。
