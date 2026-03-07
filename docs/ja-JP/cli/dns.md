---
summary: "`openclaw dns` のCLIリファレンス（広域ディスカバリーヘルパー）"
read_when:
  - Tailscale + CoreDNSを使用した広域ディスカバリー（DNS-SD）を利用したいとき
  - カスタムディスカバリードメイン用のスプリットDNSを設定しているとき（例：openclaw.internal）
title: "dns"
---

# `openclaw dns`

広域ディスカバリー（Tailscale + CoreDNS）用のDNSヘルパー。現在はmacOS + Homebrew CoreDNSに対応しています。

関連情報：

- ゲートウェイディスカバリー：[ディスカバリー](/gateway/discovery)
- 広域ディスカバリーの設定：[設定](/gateway/configuration)

## セットアップ

```bash
openclaw dns setup
openclaw dns setup --apply
```
