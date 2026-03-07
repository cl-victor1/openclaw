---
summary: "`openclaw reset` のCLIリファレンス（ローカルの状態・設定をリセット）"
read_when:
  - CLIをインストールしたまま、ローカルの状態を消去したいとき
  - 削除される内容をドライランで確認したいとき
title: "reset"
---

# `openclaw reset`

ローカルの設定・状態をリセットします（CLIはインストールされたまま残ります）。

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config+creds+sessions --yes --non-interactive
```
