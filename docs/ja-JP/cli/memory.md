---
summary: "`openclaw memory`（status/index/search）のCLIリファレンス"
read_when:
  - セマンティックメモリのインデックス作成または検索を行いたい場合
  - メモリの可用性やインデックス作成のデバッグを行っている場合
title: "memory"
---

# `openclaw memory`

セマンティックメモリのインデックス作成と検索を管理します。
アクティブなメモリプラグイン（デフォルト：`memory-core`；`plugins.slots.memory = "none"` で無効化）によって提供されます。

関連情報：

- メモリの概念：[Memory](/concepts/memory)
- プラグイン：[Plugins](/tools/plugin)

## 使用例

```bash
openclaw memory status
openclaw memory status --deep
openclaw memory index --force
openclaw memory search "meeting notes"
openclaw memory search --query "deployment" --max-results 20
openclaw memory status --json
openclaw memory status --deep --index
openclaw memory status --deep --index --verbose
openclaw memory status --agent main
openclaw memory index --agent main --verbose
```

## オプション

`memory status` と `memory index`：

- `--agent <id>`：単一エージェントにスコープを絞ります。指定しない場合、これらのコマンドは設定されたすべてのエージェントに対して実行されます。エージェントリストが設定されていない場合は、デフォルトエージェントにフォールバックします。
- `--verbose`：プローブとインデックス作成中に詳細ログを出力します。

`memory status`：

- `--deep`：ベクターとエンベディングの可用性を調査します。
- `--index`：ストアが汚れている場合に再インデックスを実行します（`--deep` を暗示）。
- `--json`：JSON 出力を表示します。

`memory index`：

- `--force`：強制的に完全な再インデックスを実行します。

`memory search`：

- クエリ入力：位置引数 `[query]` または `--query <text>` のいずれかを渡します。
- 両方が指定された場合、`--query` が優先されます。
- どちらも指定されない場合、コマンドはエラーで終了します。
- `--agent <id>`：単一エージェントにスコープを絞ります（デフォルト：デフォルトエージェント）。
- `--max-results <n>`：返される結果の数を制限します。
- `--min-score <n>`：スコアの低い一致結果をフィルタリングします。
- `--json`：JSON 結果を表示します。

注意事項：

- `memory index --verbose` はフェーズごとの詳細を出力します（プロバイダー、モデル、ソース、バッチアクティビティ）。
- `memory status` には、`memorySearch.extraPaths` で設定された追加パスが含まれます。
- 実際にアクティブなメモリリモート API キーフィールドが SecretRef として設定されている場合、コマンドはアクティブなゲートウェイスナップショットからそれらの値を解決します。ゲートウェイが利用できない場合、コマンドはすぐに失敗します。
- ゲートウェイバージョンの乖離に関する注意：このコマンドパスは `secrets.resolve` をサポートするゲートウェイを必要とします。古いゲートウェイは unknown-method エラーを返します。
