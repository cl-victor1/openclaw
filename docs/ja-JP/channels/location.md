---
summary: "チャンネルの位置情報パース（Telegram・WhatsApp）とコンテキストフィールド"
read_when:
  - チャンネルの位置情報パースを追加または変更する場合
  - エージェントのプロンプトやツールで位置情報コンテキストフィールドを使用する場合
title: "チャンネル位置情報パース"
---

# チャンネル位置情報パース

OpenClaw は、チャットチャンネルから共有された位置情報を以下の形式に正規化します：

- 受信ボディに追記される人間が読めるテキスト
- 自動返信コンテキストペイロード内の構造化フィールド

現在サポートされているチャンネル：

- **Telegram**（位置情報ピン・スポット・ライブ位置情報）
- **WhatsApp**（locationMessage・liveLocationMessage）
- **Matrix**（`geo_uri` を使用した `m.location`）

## テキストフォーマット

位置情報は括弧なしの読みやすい形式でレンダリングされます：

- ピン：
  - `📍 48.858844, 2.294351 ±12m`
- 名前付き場所：
  - `📍 Eiffel Tower — Champ de Mars, Paris (48.858844, 2.294351 ±12m)`
- ライブ共有：
  - `🛰 Live location: 48.858844, 2.294351 ±12m`

チャンネルにキャプション・コメントが含まれている場合、次の行に追記されます：

```
📍 48.858844, 2.294351 ±12m
Meet here
```

## コンテキストフィールド

位置情報が存在する場合、以下のフィールドが `ctx` に追加されます：

- `LocationLat`（数値）
- `LocationLon`（数値）
- `LocationAccuracy`（数値、メートル単位；オプション）
- `LocationName`（文字列；オプション）
- `LocationAddress`（文字列；オプション）
- `LocationSource`（`pin | place | live`）
- `LocationIsLive`（真偽値）

## チャンネル別の注意事項

- **Telegram**：スポットは `LocationName/LocationAddress` にマップされます；ライブ位置情報は `live_period` を使用します。
- **WhatsApp**：`locationMessage.comment` と `liveLocationMessage.caption` はキャプション行として追記されます。
- **Matrix**：`geo_uri` はピン位置情報としてパースされます；高度情報は無視され、`LocationIsLive` は常に false です。
