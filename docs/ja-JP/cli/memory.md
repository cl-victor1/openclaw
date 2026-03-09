---
summary: "`openclaw memory`（status/index/search）の CLI リファレンス"
read_when:
  - セマンティックメモリのインデックスや検索をしたい場合
  - メモリの利用可能性やインデックスのデバッグ時
title: "memory"
---

# `openclaw memory`

セマンティックメモリのインデックスと検索を管理します。
アクティブなメモリプラグインにより提供されます（デフォルト: `memory-core`。無効にするには `plugins.slots.memory = "none"` を設定）。

関連:

- メモリの概念: [Memory](/concepts/memory)
- プラグイン: [Plugins](/tools/plugin)

## 例

```bash
openclaw memory status
openclaw memory status --deep
openclaw memory status --deep --index
openclaw memory status --deep --index --verbose
openclaw memory index
openclaw memory index --verbose
openclaw memory search "release checklist"
openclaw memory search --query "release checklist"
openclaw memory status --agent main
openclaw memory index --agent main --verbose
```

## オプション

共通:

- `--agent <id>`: 単一のエージェントにスコープを限定します（デフォルト: 設定済みの全エージェント）。
- `--verbose`: プローブおよびインデックス中の詳細ログを出力します。

`memory search`:

- クエリ入力: 位置引数 `[query]` または `--query <text>` のいずれかを渡します。
- 両方が指定された場合、`--query` が優先されます。
- どちらも指定されていない場合、コマンドはエラーで終了します。

備考:

- `memory status --deep` はベクトル + エンベディングの利用可能性をプローブします。
- `memory status --deep --index` はストアがダーティな場合に再インデックスを実行します。
- `memory index --verbose` はフェーズごとの詳細を表示します（プロバイダー、モデル、ソース、バッチアクティビティ）。
- `memory status` には `memorySearch.extraPaths` で設定された追加パスも含まれます。
- 実質的にアクティブなメモリリモート API キーフィールドが SecretRef として設定されている場合、コマンドはアクティブな gateway スナップショットからそれらの値を解決します。gateway が利用できない場合、コマンドはすぐに失敗します。
- Gateway バージョンスキューに関する注意: このコマンドパスには `secrets.resolve` をサポートする gateway が必要です。古い gateway は unknown-method エラーを返します。
