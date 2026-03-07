---
summary: "`openclaw skills` のCLIリファレンス（list/info/check）およびスキルの適格性"
read_when:
  - 利用可能で実行準備ができているスキルを確認したい場合
  - スキルのバイナリ/環境変数/設定の不足をデバッグしたい場合
title: "skills"
---

# `openclaw skills`

スキル（バンドル + ワークスペース + マネージドオーバーライド）を検査し、適格なものと要件が不足しているものを確認します。

関連：

- スキルシステム: [Skills](/tools/skills)
- スキル設定: [Skills config](/tools/skills-config)
- ClawHub インストール: [ClawHub](/tools/clawhub)

## コマンド

```bash
openclaw skills list
openclaw skills list --eligible
openclaw skills info <name>
openclaw skills check
```
