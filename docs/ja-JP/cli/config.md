---
summary: "`openclaw config`（get/set/unset/file/validate）のCLIリファレンス"
read_when:
  - You want to read or edit config non-interactively
title: "config"
---

# `openclaw config`

設定ヘルパー: パスによる値のget/set/unset/validate、アクティブな設定ファイルの表示。
サブコマンドなしで実行すると設定ウィザードを開きます（`openclaw configure` と同じ）。

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

パスはドットまたはブラケット記法を使用します:

```bash
openclaw config get agents.defaults.workspace
openclaw config get agents.list[0].id
```

特定のエージェントを対象にするにはエージェントリストのインデックスを使用します:

```bash
openclaw config get agents.list
openclaw config set agents.list[1].tools.exec.node "node-id-or-name"
```

## 値

値は可能な場合にJSON5として解析されます；それ以外の場合は文字列として扱われます。
JSON5解析を強制するには `--strict-json` を使用します。`--json` はレガシーエイリアスとしてサポートされています。

```bash
openclaw config set agents.defaults.heartbeat.every "0m"
openclaw config set gateway.port 19001 --strict-json
openclaw config set channels.whatsapp.groups '["*"]' --strict-json
```

## サブコマンド

- `config file`: アクティブな設定ファイルパスを出力します（`OPENCLAW_CONFIG_PATH` またはデフォルトの場所から解決）。

編集後はgatewayを再起動してください。

## 検証

gatewayを起動せずに現在の設定をアクティブなスキーマに対して検証します。

```bash
openclaw config validate
openclaw config validate --json
```
