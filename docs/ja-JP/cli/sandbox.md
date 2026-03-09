---
title: Sandbox CLI
summary: "サンドボックスコンテナの管理と有効なサンドボックスポリシーの検査"
read_when: "サンドボックスコンテナを管理したり、サンドボックス/ツールポリシーの動作をデバッグしたりするとき。"
status: active
---

# Sandbox CLI

分離されたエージェント実行のための Docker ベースのサンドボックスコンテナを管理します。

## 概要

OpenClaw はセキュリティのためにエージェントを分離された Docker コンテナで実行できます。`sandbox` コマンドは、特にアップデートや設定変更後にこれらのコンテナを管理するのに役立ちます。

## コマンド

### `openclaw sandbox explain`

**有効な**サンドボックスモード/スコープ/ワークスペースアクセス、サンドボックスツールポリシー、および昇格ゲート（修正用の設定キーパス付き）を検査します。

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

### `openclaw sandbox list`

すべてのサンドボックスコンテナのステータスと設定を一覧表示します。

```bash
openclaw sandbox list
openclaw sandbox list --browser  # ブラウザコンテナのみ表示
openclaw sandbox list --json     # JSON 出力
```

**出力内容:**

- コンテナ名とステータス（running/stopped）
- Docker イメージと設定との一致状況
- 経過時間（作成からの時間）
- アイドル時間（最後の使用からの時間）
- 関連するセッション/エージェント

### `openclaw sandbox recreate`

更新されたイメージ/設定でサンドボックスコンテナを削除して再作成を強制します。

```bash
openclaw sandbox recreate --all                # すべてのコンテナを再作成
openclaw sandbox recreate --session main       # 特定のセッション
openclaw sandbox recreate --agent mybot        # 特定のエージェント
openclaw sandbox recreate --browser            # ブラウザコンテナのみ
openclaw sandbox recreate --all --force        # 確認をスキップ
```

**オプション:**

- `--all`: すべてのサンドボックスコンテナを再作成
- `--session <key>`: 特定のセッションのコンテナを再作成
- `--agent <id>`: 特定のエージェントのコンテナを再作成
- `--browser`: ブラウザコンテナのみ再作成
- `--force`: 確認プロンプトをスキップ

**重要:** コンテナはエージェントが次に使用されたときに自動的に再作成されます。

## ユースケース

### Docker イメージ更新後

```bash
# 新しいイメージをプル
docker pull openclaw-sandbox:latest
docker tag openclaw-sandbox:latest openclaw-sandbox:bookworm-slim

# 新しいイメージを使用するように設定を更新
# 設定を編集: agents.defaults.sandbox.docker.image（または agents.list[].sandbox.docker.image）

# コンテナを再作成
openclaw sandbox recreate --all
```

### サンドボックス設定変更後

```bash
# 設定を編集: agents.defaults.sandbox.*（または agents.list[].sandbox.*）

# 新しい設定を適用するために再作成
openclaw sandbox recreate --all
```

### setupCommand 変更後

```bash
openclaw sandbox recreate --all
# または特定のエージェントのみ:
openclaw sandbox recreate --agent family
```

### 特定のエージェントのみ

```bash
# 1つのエージェントのコンテナのみ更新
openclaw sandbox recreate --agent alfred
```

## なぜこれが必要なのか？

**問題:** サンドボックスの Docker イメージまたは設定を更新した場合:

- 既存のコンテナは古い設定で実行し続けます
- コンテナは 24 時間の非アクティブ後にのみプルーニングされます
- 定期的に使用されるエージェントは古いコンテナを無期限に実行し続けます

**解決策:** `openclaw sandbox recreate` を使用して古いコンテナを強制削除します。次に必要になったときに現在の設定で自動的に再作成されます。

ヒント: 手動の `docker rm` よりも `openclaw sandbox recreate` を推奨します。Gateway のコンテナ命名規則を使用し、スコープ/セッションキーが変更された際の不一致を回避します。

## 設定

サンドボックス設定は `~/.openclaw/openclaw.json` の `agents.defaults.sandbox` にあります（エージェントごとのオーバーライドは `agents.list[].sandbox`）:

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off, non-main, all
        "scope": "agent", // session, agent, shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... その他の Docker オプション
        },
        "prune": {
          "idleHours": 24, // 24 時間アイドル後に自動プルーニング
          "maxAgeDays": 7, // 7 日後に自動プルーニング
        },
      },
    },
  },
}
```

## 関連情報

- [サンドボックスドキュメント](/gateway/sandboxing)
- [エージェント設定](/concepts/agent-workspace)
- [Doctor コマンド](/gateway/doctor) - サンドボックスセットアップの確認
