---
summary: "`openclaw devices` のCLIリファレンス（デバイスペアリング・トークンのローテーション/失効）"
read_when:
  - デバイスのペアリングリクエストを承認する場合
  - デバイストークンをローテーションまたは失効させる必要がある場合
title: "devices"
---

# `openclaw devices`

デバイスのペアリングリクエストとデバイススコープのトークンを管理します。

## コマンド

### `openclaw devices list`

保留中のペアリングリクエストとペアリング済みデバイスの一覧を表示します。

```
openclaw devices list
openclaw devices list --json
```

### `openclaw devices remove <deviceId>`

ペアリング済みデバイスのエントリを1つ削除します。

```
openclaw devices remove <deviceId>
openclaw devices remove <deviceId> --json
```

### `openclaw devices clear --yes [--pending]`

ペアリング済みデバイスを一括削除します。

```
openclaw devices clear --yes
openclaw devices clear --yes --pending
openclaw devices clear --yes --pending --json
```

### `openclaw devices approve [requestId] [--latest]`

保留中のデバイスペアリングリクエストを承認します。`requestId` を省略した場合、OpenClaw は最新の保留中リクエストを自動的に承認します。

```
openclaw devices approve
openclaw devices approve <requestId>
openclaw devices approve --latest
```

### `openclaw devices reject <requestId>`

保留中のデバイスペアリングリクエストを拒否します。

```
openclaw devices reject <requestId>
```

### `openclaw devices rotate --device <id> --role <role> [--scope <scope...>]`

特定のロールに対してデバイストークンをローテーションします（オプションでスコープを更新）。

```
openclaw devices rotate --device <deviceId> --role operator --scope operator.read --scope operator.write
```

### `openclaw devices revoke --device <id> --role <role>`

特定のロールのデバイストークンを失効させます。

```
openclaw devices revoke --device <deviceId> --role node
```

## 共通オプション

- `--url <url>`：ゲートウェイのWebSocket URL（設定されている場合は `gateway.remote.url` がデフォルト）。
- `--token <token>`：ゲートウェイトークン（必要な場合）。
- `--password <password>`：ゲートウェイパスワード（パスワード認証）。
- `--timeout <ms>`：RPCタイムアウト。
- `--json`：JSON出力（スクリプト作成に推奨）。

注意：`--url` を設定した場合、CLIは設定ファイルや環境変数の認証情報にフォールバックしません。
`--token` または `--password` を明示的に渡してください。明示的な認証情報が不足している場合はエラーになります。

## 注意事項

- トークンのローテーションは新しいトークン（機密情報）を返します。シークレットと同様に扱ってください。
- これらのコマンドには `operator.pairing`（または `operator.admin`）スコープが必要です。
- `devices clear` は意図的に `--yes` でゲートされています。
- ローカルループバックでペアリングスコープが使用できない場合（かつ明示的な `--url` が渡されない場合）、list/approve はローカルペアリングのフォールバックを使用できます。

## トークンズレの回復チェックリスト

Control UI やその他のクライアントが `AUTH_TOKEN_MISMATCH` または `AUTH_DEVICE_TOKEN_MISMATCH` で失敗し続ける場合に使用してください。

1. 現在のゲートウェイトークンソースを確認する：

```bash
openclaw config get gateway.auth.token
```

2. ペアリング済みデバイスの一覧を表示し、影響を受けるデバイスIDを特定する：

```bash
openclaw devices list
```

3. 影響を受けるデバイスのオペレータートークンをローテーションする：

```bash
openclaw devices rotate --device <deviceId> --role operator
```

4. ローテーションで不十分な場合は、古いペアリングを削除して再承認する：

```bash
openclaw devices remove <deviceId>
openclaw devices list
openclaw devices approve <requestId>
```

5. 現在の共有トークン/パスワードでクライアント接続を再試行する。

関連リンク：

- [ダッシュボード認証トラブルシューティング](/web/dashboard#if-you-see-unauthorized-1008)
- [ゲートウェイトラブルシューティング](/gateway/troubleshooting#dashboard-control-ui-connectivity)
