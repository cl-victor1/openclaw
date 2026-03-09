---
summary: "`openclaw config`（get/set/unset/file/validate）の CLI リファレンス"
read_when:
  - 非インタラクティブに設定を読み書きしたい場合
title: "config"
---

# `openclaw config`

設定ヘルパー: パスによる値の get/set/unset/validate とアクティブな設定ファイルの表示。サブコマンドなしで実行すると、設定ウィザードが起動します（`openclaw configure` と同じ）。

## 例

```bash
openclaw config file
openclaw config get browser.executablePath
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set agents.list[0].tools.exec.node "node-id-or-name"
openclaw config unset tools.web.search.apiKey
openclaw config validate
openclaw config validate --json
```

## パス

パスはドット記法またはブラケット記法を使用します:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.list[0].id
```

特定のエージェントを指定するにはエージェントリストのインデックスを使用します:

```bash
openclaw config get agents.list
openclaw config set agents.list[1].tools.exec.node "node-id-or-name"
```

## 値

値は可能な場合 JSON5 としてパースされます。それ以外の場合は文字列として扱われます。
JSON5 パースを必須にするには `--strict-json` を使用します。`--json` はレガシーエイリアスとして引き続きサポートされます。

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

## サブコマンド

- `config file`: アクティブな設定ファイルのパスを表示します（`OPENCLAW_CONFIG_PATH` またはデフォルトの場所から解決）。

編集後は gateway を再起動してください。

## バリデーション

gateway を起動せずに、現在の設定をアクティブなスキーマに対して検証します。

```bash
openclaw config validate
openclaw config validate --json
```
