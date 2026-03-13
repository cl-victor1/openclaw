---
summary: "`openclaw agents` のCLIリファレンス（一覧表示・追加・削除・バインディング・ID設定）"
read_when:
  - 複数の独立したエージェント（ワークスペース・ルーティング・認証）を使いたい場合
title: "agents"
---

# `openclaw agents`

独立したエージェント（ワークスペース・認証・ルーティング）を管理します。

関連リンク：

- マルチエージェントルーティング：[Multi-Agent Routing](/concepts/multi-agent)
- エージェントワークスペース：[Agent workspace](/concepts/agent-workspace)

## 使用例

```bash
openclaw agents list
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## ルーティングバインディング

ルーティングバインディングを使用して、受信チャンネルのトラフィックを特定のエージェントに固定します。

バインディングの一覧表示：

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

バインディングの追加：

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

`accountId`（`--bind <channel>`）を省略した場合、OpenClaw はチャンネルのデフォルトと利用可能なプラグインのセットアップフックから解決します。

### バインディングスコープの動作

- `accountId` なしのバインディングは、チャンネルのデフォルトアカウントのみに一致します。
- `accountId: "*"` はチャンネル全体のフォールバック（全アカウント）であり、明示的なアカウントバインディングより優先度が低くなります。
- 同じエージェントがすでに `accountId` なしの一致するチャンネルバインディングを持っており、後から明示的または解決された `accountId` でバインドした場合、OpenClaw は重複を追加せずに既存のバインディングをアップグレードします。

例：

```bash
# 初期のチャンネルのみのバインディング
openclaw agents bind --agent work --bind telegram

# 後でアカウントスコープのバインディングにアップグレード
openclaw agents bind --agent work --bind telegram:ops
```

アップグレード後、そのバインディングのルーティングは `telegram:ops` にスコープされます。デフォルトアカウントのルーティングも必要な場合は、明示的に追加してください（例：`--bind telegram:default`）。

バインディングの削除：

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## IDファイル

各エージェントワークスペースは、ワークスペースのルートに `IDENTITY.md` を含めることができます：

- パスの例：`~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity` はワークスペースのルート（または明示的な `--identity-file`）から読み込みます。

アバターのパスはワークスペースのルートを基準に解決されます。

## IDの設定

`set-identity` は `agents.list[].identity` にフィールドを書き込みます：

- `name`
- `theme`
- `emoji`
- `avatar`（ワークスペース相対パス、http(s) URL、またはデータURI）

`IDENTITY.md` から読み込む：

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

フィールドを明示的に上書きする：

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

設定サンプル：

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```
