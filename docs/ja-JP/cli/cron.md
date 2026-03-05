---
summary: "`openclaw cron`（バックグラウンドジョブのスケジュールと実行）のCLIリファレンス"
read_when:
  - スケジュールジョブやウェイクアップを設定したい場合
  - cronの実行やログをデバッグしている場合
title: "cron"
x-i18n:
  source_path: docs/cli/cron.md
  generated_at: "2026-03-05T10:01:00Z"
  model: claude-opus-4-6
  provider: pi
---

# `openclaw cron`

Gatewayスケジューラのcronジョブを管理します。

関連：

- Cronジョブ：[Cron jobs](/automation/cron-jobs)

ヒント：完全なコマンド一覧を確認するには `openclaw cron --help` を実行してください。

注意：隔離された `cron add` ジョブはデフォルトで `--announce` 配信になります。出力を内部に留めるには `--no-deliver` を使用してください。`--deliver` は `--announce` の非推奨エイリアスとして残っています。

注意：ワンショット（`--at`）ジョブは成功後にデフォルトで削除されます。保持するには `--keep-after-run` を使用してください。

注意：定期ジョブは連続エラー後に指数バックオフリトライを使用するようになりました（30秒 → 1分 → 5分 → 15分 → 60分）。次の成功した実行後に通常のスケジュールに戻ります。

注意：保持/プルーニングは設定で制御されます：

- `cron.sessionRetention`（デフォルト `24h`）は完了した隔離実行セッションをプルーニングします。
- `cron.runLog.maxBytes` + `cron.runLog.keepLines` は `~/.openclaw/cron/runs/<jobId>.jsonl` をプルーニングします。

## よく使う編集

メッセージを変更せずに配信設定を更新する：

```bash
openclaw cron edit <job-id> --announce --channel telegram --to "123456789"
```

隔離ジョブの配信を無効にする：

```bash
openclaw cron edit <job-id> --no-deliver
```

特定のチャネルにアナウンスする：

```bash
openclaw cron edit <job-id> --announce --channel slack --to "channel:C1234567890"
```
