---
summary: "cronおよびハートビートのスケジューリングと配信のトラブルシューティング"
read_when:
  - cronが実行されなかった場合
  - cronは実行されたがメッセージが配信されなかった場合
  - ハートビートが無音またはスキップされているように見える場合
title: "オートメーション トラブルシューティング"
---

# オートメーション トラブルシューティング

このページは、スケジューラーと配信の問題（`cron` + `heartbeat`）に使用してください。

## コマンドラダー

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

次にオートメーションのチェックを実行します：

```bash
openclaw cron status
openclaw cron list
openclaw system heartbeat last
```

## Cronが実行されない

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw logs --follow
```

正常な出力の例：

- `cron status` が有効で将来の `nextWakeAtMs` を報告している。
- ジョブが有効で有効なスケジュール/タイムゾーンが設定されている。
- `cron runs` が `ok` または明示的なスキップ理由を示している。

よくあるパターン：

- `cron: scheduler disabled; jobs will not run automatically` → cronが設定/環境変数で無効になっている。
- `cron: timer tick failed` → スケジューラーのティックがクラッシュした；周辺のスタック/ログコンテキストを確認する。
- 実行出力に `reason: not-due` → `--force` なしで手動実行が呼び出されたが、ジョブがまだ期限を迎えていない。

## Cronは実行されたが配信されない

```bash
openclaw cron runs --id <jobId> --limit 20
openclaw cron list
openclaw channels status --probe
openclaw logs --follow
```

正常な出力の例：

- 実行ステータスが `ok` である。
- 独立したジョブの配信モード/ターゲットが設定されている。
- チャンネルプローブがターゲットチャンネルの接続を報告している。

よくあるパターン：

- 実行は成功したが配信モードが `none` → 外部メッセージは期待されない。
- 配信ターゲットが不足/無効（`channel`/`to`）→ 実行は内部的に成功するかもしれないが、アウトバウンドをスキップする。
- チャンネル認証エラー（`unauthorized`、`missing_scope`、`Forbidden`）→ チャンネルの認証情報/権限によって配信がブロックされている。

## ハートビートが抑制またはスキップされる

```bash
openclaw system heartbeat last
openclaw logs --follow
openclaw config get agents.defaults.heartbeat
openclaw channels status --probe
```

正常な出力の例：

- ハートビートがゼロ以外のインターバルで有効になっている。
- 最後のハートビート結果が `ran` である（またはスキップ理由が理解できる）。

よくあるパターン：

- `heartbeat skipped` で `reason=quiet-hours` → `activeHours` の外にいる。
- `requests-in-flight` → メインレーンがビジー；ハートビートが延期された。
- `empty-heartbeat-file` → `HEARTBEAT.md` にアクション可能なコンテンツがなく、タグ付きcronイベントもキューにないため、インターバルハートビートがスキップされた。
- `alerts-disabled` → 表示設定によりアウトバウンドのハートビートメッセージが抑制されている。

## タイムゾーンとactiveHoursの注意点

```bash
openclaw config get agents.defaults.heartbeat.activeHours
openclaw config get agents.defaults.heartbeat.activeHours.timezone
openclaw config get agents.defaults.userTimezone || echo "agents.defaults.userTimezone not set"
openclaw cron list
openclaw logs --follow
```

クイックルール：

- `Config path not found: agents.defaults.userTimezone` はキーが未設定であることを意味します；ハートビートはホストのタイムゾーン（または設定されている場合は `activeHours.timezone`）にフォールバックします。
- `--tz` なしのcronはゲートウェイホストのタイムゾーンを使用します。
- ハートビートの `activeHours` は設定されたタイムゾーン解決（`user`、`local`、または明示的なIANA tz）を使用します。
- タイムゾーンなしのISOタイムスタンプは、cronの `at` スケジュールではUTCとして扱われます。

よくあるパターン：

- ホストのタイムゾーン変更後、ジョブが間違った時刻に実行される。
- `activeHours.timezone` が間違っているため、昼間に常にハートビートがスキップされる。

関連リンク：

- [/automation/cron-jobs](/automation/cron-jobs)
- [/gateway/heartbeat](/gateway/heartbeat)
- [/automation/cron-vs-heartbeat](/automation/cron-vs-heartbeat)
- [/concepts/timezone](/concepts/timezone)
