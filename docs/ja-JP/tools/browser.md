---
summary: "統合ブラウザコントロールサービス + アクションコマンド"
read_when:
  - エージェント制御のブラウザ自動化を追加するとき
  - openclaw が自分の Chrome に干渉する理由をデバッグするとき
  - macOS アプリでブラウザ設定とライフサイクルを実装するとき
title: "ブラウザ（OpenClaw 管理）"
---

# ブラウザ（openclaw 管理）

OpenClaw はエージェントが制御する**専用の Chrome/Brave/Edge/Chromium プロファイル**を実行できます。
個人のブラウザから隔離されており、Gateway 内の小さなローカルコントロールサービスを通じて管理されます（ループバックのみ）。

初心者向けの説明：

- **エージェント専用の独立したブラウザ**と考えてください。
- `openclaw` プロファイルは個人のブラウザプロファイルに**一切触れません**。
- エージェントは安全なレーンで**タブを開く、ページを読む、クリック、入力**ができます。
- 組み込みの `user` プロファイルは実際のサインイン済み Chrome セッションに接続します；`chrome-relay` は明示的な拡張機能リレープロファイルです。

## 機能

- **openclaw** という名前の独立したブラウザプロファイル（デフォルトはオレンジのアクセント）。
- 確定的なタブ制御（リスト/開く/フォーカス/閉じる）。
- エージェントアクション（クリック/入力/ドラッグ/選択）、スナップショット、スクリーンショット、PDF。
- オプションのマルチプロファイルサポート（`openclaw`、`work`、`remote` など）。

このブラウザは**日常使いのブラウザではありません**。エージェントの自動化と検証のための安全で隔離されたサーフェスです。

## クイックスタート

```bash
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

「Browser disabled」と表示される場合は、設定で有効にして（以下を参照）Gateway を再起動してください。

## プロファイル：`openclaw` vs `user` vs `chrome-relay`

- `openclaw`：管理された隔離ブラウザ（拡張機能不要）。
- `user`：**実際のサインイン済み Chrome** セッション用の組み込み Chrome MCP アタッチプロファイル。
- `chrome-relay`：**システムブラウザ**への拡張機能リレー（タブにアタッチするために OpenClaw 拡張機能が必要）。

エージェントブラウザツール呼び出しの場合：

- デフォルト：隔離された `openclaw` ブラウザを使用。
- 既存のログイン済みセッションが重要で、ユーザーがアタッチプロンプトをクリック/承認できる場合は `profile="user"` を優先。
- ユーザーが明示的に Chrome 拡張機能 / ツールバーボタンのアタッチフローを望む場合のみ `profile="chrome-relay"` を使用。
- `profile` は特定のブラウザモードが必要なときの明示的な上書きです。

管理モードをデフォルトにする場合は `browser.defaultProfile: "openclaw"` を設定してください。

## 設定

ブラウザ設定は `~/.openclaw/openclaw.json` にあります。

```json5
{
  browser: {
    enabled: true, // デフォルト: true
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: true, // デフォルトの信頼ネットワークモード
      // allowPrivateNetwork: true, // レガシーエイリアス
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // レガシーシングルプロファイル上書き
    remoteCdpTimeoutMs: 1500, // リモート CDP HTTP タイムアウト（ms）
    remoteCdpHandshakeTimeoutMs: 3000, // リモート CDP WebSocket ハンドシェイクタイムアウト（ms）
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: { cdpPort: 18801, color: "#0066CC" },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      "chrome-relay": {
        driver: "extension",
        cdpUrl: "http://127.0.0.1:18792",
        color: "#00AA00",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

注意事項：

- ブラウザコントロールサービスは `gateway.port` から派生したポートのループバックにバインドされます（デフォルト：`18791`、つまり gateway + 2）。リレーは次のポート（`18792`）を使用します。
- Gateway ポートを上書きする場合（`gateway.port` または `OPENCLAW_GATEWAY_PORT`）、派生したブラウザポートは同じ「ファミリー」に留まるようにシフトします。
- `cdpUrl` は未設定の場合、リレーポートにデフォルト設定されます。
- `remoteCdpTimeoutMs` はリモート（非ループバック）CDP 到達可能性チェックに適用されます。
- `remoteCdpHandshakeTimeoutMs` はリモート CDP WebSocket 到達可能性チェックに適用されます。
- ブラウザのナビゲーション/タブを開くは、ナビゲーション前に SSRF ガードが行われ、最終 `http(s)` URL でナビゲーション後にベストエフォートで再チェックされます。
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` のデフォルトは `true`（信頼ネットワークモデル）。厳格なパブリックのみのブラウジングのためには `false` に設定してください。
- `browser.ssrfPolicy.allowPrivateNetwork` は互換性のためのレガシーエイリアスとして引き続きサポートされます。
- `attachOnly: true` は「ローカルブラウザを起動しない；すでに実行中の場合のみアタッチ」を意味します。
- `color` + プロファイルごとの `color` でブラウザ UI をティントし、どのプロファイルがアクティブかを確認できます。
- デフォルトプロファイルは `openclaw`（OpenClaw 管理のスタンドアロンブラウザ）。サインイン済みユーザーブラウザを使用するには `defaultProfile: "user"` を、拡張機能リレーには `defaultProfile: "chrome-relay"` を使用します。
- 自動検出の順序：Chromium ベースの場合はシステムデフォルトブラウザ；それ以外は Chrome → Brave → Edge → Chromium → Chrome Canary。
- ローカルの `openclaw` プロファイルは `cdpPort`/`cdpUrl` を自動割り当てします — リモート CDP の場合のみ設定してください。
- `driver: "existing-session"` は生の CDP の代わりに Chrome DevTools MCP を使用します。そのドライバーには `cdpUrl` を設定しないでください。

## Brave（または別の Chromium ベースブラウザ）を使用する

**システムデフォルト**ブラウザが Chromium ベース（Chrome/Brave/Edge など）の場合、OpenClaw は自動的にそれを使用します。自動検出を上書きするには `browser.executablePath` を設定します：

CLI の例：

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
```

```json5
// macOS
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser"
  }
}

// Windows
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe"
  }
}

// Linux
{
  browser: {
    executablePath: "/usr/bin/brave-browser"
  }
}
```

## ローカル vs リモート制御

- **ローカル制御（デフォルト）：** Gateway はループバックコントロールサービスを開始し、ローカルブラウザを起動できます。
- **リモート制御（ノードホスト）：** ブラウザを持つマシン上でノードホストを実行します；Gateway はブラウザアクションをそれにプロキシします。
- **リモート CDP：** `browser.profiles.<name>.cdpUrl`（または `browser.cdpUrl`）をリモートの Chromium ベースブラウザにアタッチするように設定します。この場合、OpenClaw はローカルブラウザを起動しません。

リモート CDP URL には認証を含めることができます：

- クエリトークン（例：`https://provider.example?token=<token>`）
- HTTP Basic 認証（例：`https://user:pass@provider.example`）

OpenClaw は `/json/*` エンドポイントへの呼び出しと CDP WebSocket への接続時に認証を保持します。設定ファイルにコミットする代わりに、トークンには環境変数またはシークレットマネージャーを優先してください。

## ノードブラウザプロキシ（ゼロコンフィグデフォルト）

ブラウザを持つマシン上で**ノードホスト**を実行すると、OpenClaw は追加のブラウザ設定なしにブラウザツール呼び出しをそのノードに自動ルーティングできます。これはリモートゲートウェイのデフォルトパスです。

注意事項：

- ノードホストはローカルブラウザコントロールサーバーを**プロキシコマンド**を通じて公開します。
- プロファイルはノード自身の `browser.profiles` 設定から取得されます（ローカルと同じ）。
- 不要な場合は無効にしてください：
  - ノード上：`nodeHost.browserProxy.enabled=false`
  - Gateway 上：`gateway.nodes.browser.mode="off"`

## Browserless（ホストされたリモート CDP）

[Browserless](https://browserless.io) は HTTPS を通じて CDP エンドポイントを公開するホストされた Chromium サービスです。API キーで認証して OpenClaw ブラウザプロファイルを Browserless リージョンエンドポイントに向けることができます。

例：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    remoteCdpTimeoutMs: 2000,
    remoteCdpHandshakeTimeoutMs: 4000,
    profiles: {
      browserless: {
        cdpUrl: "https://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

注意事項：

- `<BROWSERLESS_API_KEY>` を実際の Browserless トークンに置き換えてください。
- Browserless アカウントに一致するリージョンエンドポイントを選択してください（ドキュメントを参照）。

## ダイレクト WebSocket CDP プロバイダー

一部のホストされたブラウザサービスは、標準の HTTP ベースの CDP ディスカバリー（`/json/version`）ではなく**ダイレクト WebSocket**エンドポイントを公開します。OpenClaw は両方をサポートします：

- **HTTP(S) エンドポイント**（例：Browserless）— OpenClaw は `/json/version` を呼び出して WebSocket デバッガー URL を検出し、接続します。
- **WebSocket エンドポイント**（`ws://` / `wss://`）— OpenClaw は直接接続し、`/json/version` をスキップします。[Browserbase](https://www.browserbase.com) や WebSocket URL を提供するプロバイダーにはこれを使用してください。

### Browserbase

[Browserbase](https://www.browserbase.com) は組み込みの CAPTCHA 解決、ステルスモード、住宅プロキシを備えたヘッドレスブラウザ実行のためのクラウドプラットフォームです。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    remoteCdpTimeoutMs: 3000,
    remoteCdpHandshakeTimeoutMs: 5000,
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

注意事項：

- [サインアップ](https://www.browserbase.com/sign-up)して[概要ダッシュボード](https://www.browserbase.com/overview)から**API キー**をコピーしてください。
- `<BROWSERBASE_API_KEY>` を実際の Browserbase API キーに置き換えてください。
- Browserbase は WebSocket 接続時にブラウザセッションを自動作成するため、手動セッション作成ステップは不要です。
- 無料プランでは1つの同時セッションと月1ブラウザ時間が許可されます。有料プランの制限については[料金](https://www.browserbase.com/pricing)を参照してください。
- API リファレンス、SDK ガイド、統合例については[Browserbase ドキュメント](https://docs.browserbase.com)を参照してください。

## セキュリティ

主なポイント：

- ブラウザ制御はループバックのみ；アクセスは Gateway の認証またはノードペアリングを通じて流れます。
- ブラウザ制御が有効でかつ認証が設定されていない場合、OpenClaw は起動時に `gateway.auth.token` を自動生成して設定に保存します。
- Gateway とノードホストをプライベートネットワーク（Tailscale）上に保持し、公開露出を避けてください。
- リモート CDP の URL/トークンをシークレットとして扱い、環境変数またはシークレットマネージャーを優先してください。

リモート CDP のヒント：

- 可能な場合は暗号化されたエンドポイント（HTTPS または WSS）と短命のトークンを優先してください。
- 長命のトークンを設定ファイルに直接埋め込むのを避けてください。

## プロファイル（マルチブラウザ）

OpenClaw は複数の名前付きプロファイル（ルーティング設定）をサポートします。プロファイルは以下のいずれかです：

- **openclaw 管理**：独自のユーザーデータディレクトリ + CDP ポートを持つ専用 Chromium ベースブラウザインスタンス
- **リモート**：明示的な CDP URL（他の場所で実行中の Chromium ベースブラウザ）
- **拡張機能リレー**：ローカルリレー + Chrome 拡張機能を通じた既存の Chrome タブ
- **既存セッション**：Chrome DevTools MCP 自動接続を通じた既存の Chrome プロファイル

デフォルト：

- `openclaw` プロファイルは存在しない場合に自動作成されます。
- `chrome-relay` プロファイルは Chrome 拡張機能リレー用の組み込みプロファイルです（デフォルトで `http://127.0.0.1:18792` を指します）。
- 既存セッションプロファイルはオプトイン；`--driver existing-session` で作成します。
- ローカル CDP ポートはデフォルトで **18800〜18899** から割り当てられます。
- プロファイルを削除すると、そのローカルデータディレクトリがゴミ箱に移動されます。

すべてのコントロールエンドポイントは `?profile=<name>` を受け入れます；CLI は `--browser-profile` を使用します。

## Chrome 拡張機能リレー（既存の Chrome を使用する）

OpenClaw はローカル CDP リレー + Chrome 拡張機能を介して**既存の Chrome タブ**を操作することもできます（「openclaw」Chrome インスタンスは不要）。

完全なガイド：[Chrome 拡張機能](/tools/chrome-extension)

フロー：

- Gateway はローカル（同じマシン）で実行されるか、ノードホストがブラウザマシンで実行されます。
- ローカルの**リレーサーバー**がループバックの `cdpUrl` でリッスンします（デフォルト：`http://127.0.0.1:18792`）。
- タブでエージェントに制御させたい場合、**OpenClaw Browser Relay**拡張機能アイコンをクリックしてアタッチします（自動アタッチしません）。
- エージェントは正しいプロファイルを選択することで通常の `browser` ツールを通じてそのタブを制御します。

Gateway が他の場所で実行されている場合、Gateway がブラウザアクションをプロキシできるように、ブラウザマシン上でノードホストを実行してください。

### サンドボックス化されたセッション

エージェントセッションがサンドボックス化されている場合、`browser` ツールはデフォルトで `target="sandbox"`（サンドボックスブラウザ）になる場合があります。
Chrome 拡張機能リレーのテイクオーバーにはホストブラウザ制御が必要なため、以下のいずれかを行ってください：

- セッションをサンドボックスなしで実行する、または
- `agents.defaults.sandbox.browser.allowHostControl: true` を設定してツール呼び出し時に `target="host"` を使用する。

### セットアップ

1. 拡張機能をロード（dev/unpacked）：

```bash
openclaw browser extension install
```

- Chrome → `chrome://extensions` → 「デベロッパーモード」を有効にする
- 「パッケージ化されていない拡張機能を読み込む」→ `openclaw browser extension path` で表示されたディレクトリを選択
- 拡張機能をピン留めし、制御したいタブでクリックする（バッジが `ON` を表示）。

2. 使用方法：

- CLI：`openclaw browser --browser-profile chrome-relay tabs`
- エージェントツール：`profile="chrome-relay"` を持つ `browser`

オプション：異なる名前やリレーポートが必要な場合、独自のプロファイルを作成します：

```bash
openclaw browser create-profile \
  --name my-chrome \
  --driver extension \
  --cdp-url http://127.0.0.1:18792 \
  --color "#00AA00"
```

注意事項：

- このモードは大部分の操作（スクリーンショット/スナップショット/アクション）に Playwright-on-CDP を使用します。
- 拡張機能アイコンを再度クリックしてデタッチします。
- エージェントの使用：ログイン済みサイトには `profile="user"` を優先。拡張機能フローを明示的に望む場合のみ `profile="chrome-relay"` を使用。ユーザーは拡張機能をクリックしてタブをアタッチするために存在している必要があります。

## MCP を介した Chrome 既存セッション

OpenClaw は公式の Chrome DevTools MCP サーバーを通じて実行中の Chrome プロファイルにアタッチすることもできます。これはその Chrome プロファイルで既に開いているタブとログイン状態を再利用します。

公式の背景とセットアップのリファレンス：

- [Chrome for Developers: Chrome DevTools MCP を使用したブラウザセッションのデバッグ](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

組み込みプロファイル：

- `user`

オプション：異なる名前や色が必要な場合、独自のカスタム既存セッションプロファイルを作成します。

Chrome での設定：

1. `chrome://inspect/#remote-debugging` を開く
2. リモートデバッグを有効にする
3. Chrome を実行し続け、OpenClaw がアタッチする際の接続プロンプトを承認する

ライブアタッチスモークテスト：

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

成功時の状態：

- `status` が `driver: existing-session` を表示
- `status` が `transport: chrome-mcp` を表示
- `status` が `running: true` を表示
- `tabs` が既に開いている Chrome タブをリストする
- `snapshot` が選択されたライブタブからの ref を返す

アタッチが機能しない場合の確認事項：

- Chrome のバージョンが `144+` であること
- `chrome://inspect/#remote-debugging` でリモートデバッグが有効になっていること
- Chrome がアタッチ同意プロンプトを表示し、承認したこと

エージェントの使用：

- ユーザーのログイン済みブラウザ状態が必要な場合は `profile="user"` を使用。
- カスタム既存セッションプロファイルを使用する場合は、その明示的なプロファイル名を渡す。
- ユーザーが明示的に拡張機能 / アタッチタブフローを望まない限り、`profile="chrome-relay"` より `profile="user"` を優先する。
- ユーザーがアタッチプロンプトを承認するためにコンピューターにいる場合のみこのモードを選択する。
- Gateway またはノードホストは `npx chrome-devtools-mcp@latest --autoConnect` をスポーンできます。

注意事項：

- このパスは隔離された `openclaw` プロファイルよりリスクが高く、サインイン済みブラウザセッション内でアクションを実行できます。
- OpenClaw はこのドライバーのために Chrome を起動しません；既存のセッションにのみアタッチします。
- OpenClaw はここでレガシーのデフォルトプロファイルリモートデバッグポートワークフローではなく、公式の Chrome DevTools MCP `--autoConnect` フローを使用します。
- 既存セッションのスクリーンショットはページキャプチャとスナップショットからの `--ref` 要素キャプチャをサポートしますが、CSS `--element` セレクターはサポートしません。
- 既存セッションの `wait --url` は他のブラウザドライバーと同様に完全一致、部分文字列、グロブパターンをサポートします。`wait --load networkidle` はまだサポートされていません。
- PDF エクスポートやダウンロードインターセプトなど一部の機能は、まだ拡張機能リレーまたは管理ブラウザパスが必要です。
- リレーはデフォルトでループバックのみに保持してください。リレーが異なるネットワーク名前空間から到達可能である必要がある場合（例：WSL2 の Gateway、Windows の Chrome）、周囲のネットワークをプライベートかつ認証済みに保ちながら `browser.relayBindHost` を `0.0.0.0` などの明示的なバインドアドレスに設定してください。

WSL2 / クロス名前空間の例：

```json5
{
  browser: {
    enabled: true,
    relayBindHost: "0.0.0.0",
    defaultProfile: "chrome-relay",
  },
}
```

## 隔離の保証

- **専用ユーザーデータディレクトリ**：個人のブラウザプロファイルには一切触れません。
- **専用ポート**：開発ワークフローとの衝突を防ぐために `9222` を避けます。
- **確定的なタブ制御**：「最後のタブ」ではなく `targetId` でタブをターゲットします。

## ブラウザの選択

ローカルで起動する場合、OpenClaw は最初に利用可能なものを選びます：

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

`browser.executablePath` で上書きできます。

プラットフォーム：

- macOS：`/Applications` と `~/Applications` を確認。
- Linux：`google-chrome`、`brave`、`microsoft-edge`、`chromium` などを検索。
- Windows：一般的なインストール場所を確認。

## コントロール API（オプション）

ローカル統合のみのために、Gateway は小さなループバック HTTP API を公開します：

- ステータス/開始/停止：`GET /`、`POST /start`、`POST /stop`
- タブ：`GET /tabs`、`POST /tabs/open`、`POST /tabs/focus`、`DELETE /tabs/:targetId`
- スナップショット/スクリーンショット：`GET /snapshot`、`POST /screenshot`
- アクション：`POST /navigate`、`POST /act`
- フック：`POST /hooks/file-chooser`、`POST /hooks/dialog`
- ダウンロード：`POST /download`、`POST /wait/download`
- デバッグ：`GET /console`、`POST /pdf`
- デバッグ：`GET /errors`、`GET /requests`、`POST /trace/start`、`POST /trace/stop`、`POST /highlight`
- ネットワーク：`POST /response/body`
- 状態：`GET /cookies`、`POST /cookies/set`、`POST /cookies/clear`
- 状態：`GET /storage/:kind`、`POST /storage/:kind/set`、`POST /storage/:kind/clear`
- 設定：`POST /set/offline`、`POST /set/headers`、`POST /set/credentials`、`POST /set/geolocation`、`POST /set/media`、`POST /set/timezone`、`POST /set/locale`、`POST /set/device`

すべてのエンドポイントは `?profile=<name>` を受け入れます。

gateway 認証が設定されている場合、ブラウザ HTTP ルートも認証が必要です：

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` または HTTP Basic 認証

### Playwright の要件

一部の機能（navigate/act/AI スナップショット/ロールスナップショット、要素スクリーンショット、PDF）は Playwright が必要です。Playwright がインストールされていない場合、それらのエンドポイントは明確な 501 エラーを返します。ARIA スナップショットと基本的なスクリーンショットは openclaw 管理の Chrome でも動作します。Chrome 拡張機能リレードライバーの場合、ARIA スナップショットとスクリーンショットには Playwright が必要です。

`Playwright is not available in this gateway build` と表示された場合は、完全な Playwright パッケージ（`playwright-core` ではなく）をインストールしてゲートウェイを再起動するか、ブラウザサポート付きで OpenClaw を再インストールしてください。

#### Docker での Playwright インストール

Gateway が Docker で実行されている場合、`npx playwright` を避けてください（npm オーバーライドの競合）。
代わりにバンドルされた CLI を使用します：

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

ブラウザのダウンロードを保持するには、`PLAYWRIGHT_BROWSERS_PATH`（例：`/home/node/.cache/ms-playwright`）を設定し、`/home/node` が `OPENCLAW_HOME_VOLUME` またはバインドマウントを通じて保持されていることを確認してください。[Docker](/install/docker) を参照してください。

## 仕組み（内部）

高レベルのフロー：

- 小さな**コントロールサーバー**が HTTP リクエストを受け入れます。
- **CDP** を介して Chromium ベースのブラウザ（Chrome/Brave/Edge/Chromium）に接続します。
- 高度なアクション（クリック/入力/スナップショット/PDF）では、CDP の上で **Playwright** を使用します。
- Playwright が欠けている場合、非 Playwright 操作のみが利用可能です。

この設計により、エージェントは安定した確定的なインターフェースを維持しながら、ローカル/リモートブラウザとプロファイルを交換できます。

## CLI クイックリファレンス

すべてのコマンドは `--browser-profile <name>` で特定のプロファイルをターゲットできます。
すべてのコマンドは `--json` でマシン読み取り可能な出力（安定したペイロード）も受け入れます。

基本：

- `openclaw browser status`
- `openclaw browser start`
- `openclaw browser stop`
- `openclaw browser tabs`
- `openclaw browser tab`
- `openclaw browser tab new`
- `openclaw browser tab select 2`
- `openclaw browser tab close 2`
- `openclaw browser open https://example.com`
- `openclaw browser focus abcd1234`
- `openclaw browser close abcd1234`

検査：

- `openclaw browser screenshot`
- `openclaw browser screenshot --full-page`
- `openclaw browser screenshot --ref 12`
- `openclaw browser screenshot --ref e12`
- `openclaw browser snapshot`
- `openclaw browser snapshot --format aria --limit 200`
- `openclaw browser snapshot --interactive --compact --depth 6`
- `openclaw browser snapshot --efficient`
- `openclaw browser snapshot --labels`
- `openclaw browser snapshot --selector "#main" --interactive`
- `openclaw browser snapshot --frame "iframe#main" --interactive`
- `openclaw browser console --level error`
- `openclaw browser errors --clear`
- `openclaw browser requests --filter api --clear`
- `openclaw browser pdf`
- `openclaw browser responsebody "**/api" --max-chars 5000`

アクション：

- `openclaw browser navigate https://example.com`
- `openclaw browser resize 1280 720`
- `openclaw browser click 12 --double`
- `openclaw browser click e12 --double`
- `openclaw browser type 23 "hello" --submit`
- `openclaw browser press Enter`
- `openclaw browser hover 44`
- `openclaw browser scrollintoview e12`
- `openclaw browser drag 10 11`
- `openclaw browser select 9 OptionA OptionB`
- `openclaw browser download e12 report.pdf`
- `openclaw browser waitfordownload report.pdf`
- `openclaw browser upload /tmp/openclaw/uploads/file.pdf`
- `openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'`
- `openclaw browser dialog --accept`
- `openclaw browser wait --text "Done"`
- `openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"`
- `openclaw browser evaluate --fn '(el) => el.textContent' --ref 7`
- `openclaw browser highlight e12`
- `openclaw browser trace start`
- `openclaw browser trace stop`

状態：

- `openclaw browser cookies`
- `openclaw browser cookies set session abc123 --url "https://example.com"`
- `openclaw browser cookies clear`
- `openclaw browser storage local get`
- `openclaw browser storage local set theme dark`
- `openclaw browser storage session clear`
- `openclaw browser set offline on`
- `openclaw browser set headers --headers-json '{"X-Debug":"1"}'`
- `openclaw browser set credentials user pass`
- `openclaw browser set credentials --clear`
- `openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"`
- `openclaw browser set geo --clear`
- `openclaw browser set media dark`
- `openclaw browser set timezone America/New_York`
- `openclaw browser set locale en-US`
- `openclaw browser set device "iPhone 14"`

注意事項：

- `upload` と `dialog` は**準備**呼び出しです；チューザー/ダイアログをトリガーするクリック/プレスの前に実行してください。
- ダウンロードとトレースの出力パスは OpenClaw 一時ルートに制限されます：
  - トレース：`/tmp/openclaw`（フォールバック：`${os.tmpdir()}/openclaw`）
  - ダウンロード：`/tmp/openclaw/downloads`（フォールバック：`${os.tmpdir()}/openclaw/downloads`）
- アップロードパスは OpenClaw 一時アップロードルートに制限されます：
  - アップロード：`/tmp/openclaw/uploads`（フォールバック：`${os.tmpdir()}/openclaw/uploads`）
- `upload` は `--input-ref` または `--element` を通じてファイル入力を直接設定することもできます。
- `snapshot`：
  - `--format ai`（Playwright がインストールされている場合のデフォルト）：数値 ref（`aria-ref="<n>"`）付きの AI スナップショットを返します。
  - `--format aria`：アクセシビリティツリーを返します（ref なし；検査のみ）。
  - `--efficient`（または `--mode efficient`）：コンパクトなロールスナップショットプリセット（インタラクティブ + コンパクト + 深さ + より低い maxChars）。
  - 設定デフォルト（ツール/CLI のみ）：`browser.snapshotDefaults.mode: "efficient"` を設定すると、呼び出し側がモードを渡さない場合に効率的なスナップショットを使用します（[Gateway 設定](/gateway/configuration#browser-openclaw-managed-browser)を参照）。
  - ロールスナップショットオプション（`--interactive`、`--compact`、`--depth`、`--selector`）は `ref=e12` のような ref を持つロールベースのスナップショットを強制します。
  - `--frame "<iframe selector>"` はロールスナップショットを iframe にスコープします（`e12` のようなロール ref と組み合わせます）。
  - `--interactive` はインタラクティブ要素のフラットで選びやすいリストを出力します（アクションの駆動に最適）。
  - `--labels` はオーバーレイされた ref ラベル付きのビューポートのみのスクリーンショットを追加します（`MEDIA:<path>` を出力）。
- `click`/`type` などには `snapshot` からの `ref`（数値 `12` またはロール ref `e12`）が必要です。CSS セレクターはアクションではサポートされていません。

## スナップショットと ref

OpenClaw は2種類の「スナップショット」スタイルをサポートします：

- **AI スナップショット（数値 ref）**：`openclaw browser snapshot`（デフォルト；`--format ai`）
  - 出力：数値 ref を含むテキストスナップショット。
  - アクション：`openclaw browser click 12`、`openclaw browser type 23 "hello"`。
  - 内部的には、ref は Playwright の `aria-ref` を介して解決されます。

- **ロールスナップショット（`e12` のようなロール ref）**：`openclaw browser snapshot --interactive`（または `--compact`、`--depth`、`--selector`、`--frame`）
  - 出力：`[ref=e12]`（およびオプションの `[nth=1]`）を持つロールベースのリスト/ツリー。
  - アクション：`openclaw browser click e12`、`openclaw browser highlight e12`。
  - 内部的には、ref は `getByRole(...)` を介して解決されます（重複には `nth()` を追加）。
  - `--labels` を追加してオーバーレイされた `e12` ラベル付きのビューポートスクリーンショットを含めます。

ref の動作：

- ref は**ナビゲーション間で安定しません**；何かが失敗した場合は `snapshot` を再実行して新しい ref を使用してください。
- ロールスナップショットが `--frame` で取得された場合、ロール ref は次のロールスナップショットまでその iframe にスコープされます。

## Wait のパワーアップ

時間/テキスト以上のことを待つことができます：

- URL を待つ（Playwright でサポートされるグロブ）：
  - `openclaw browser wait --url "**/dash"`
- ロード状態を待つ：
  - `openclaw browser wait --load networkidle`
- JS 述語を待つ：
  - `openclaw browser wait --fn "window.ready===true"`
- セレクターが表示されるのを待つ：
  - `openclaw browser wait "#main"`

これらを組み合わせることができます：

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## デバッグワークフロー

アクションが失敗した場合（例：「not visible」、「strict mode violation」、「covered」）：

1. `openclaw browser snapshot --interactive`
2. `click <ref>` / `type <ref>` を使用（インタラクティブモードではロール ref を優先）
3. まだ失敗する場合：`openclaw browser highlight <ref>` で Playwright がターゲットにしているものを確認
4. ページが奇妙な動作をする場合：
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. 深いデバッグ用：トレースを記録：
   - `openclaw browser trace start`
   - 問題を再現する
   - `openclaw browser trace stop`（`TRACE:<path>` を出力）

## JSON 出力

`--json` はスクリプトや構造化ツールのためのものです。

例：

```bash
openclaw browser status --json
openclaw browser snapshot --interactive --json
openclaw browser requests --filter api --json
openclaw browser cookies --json
```

JSON のロールスナップショットには `refs` と小さな `stats` ブロック（行数/文字数/ref 数/インタラクティブ数）が含まれるため、ツールがペイロードサイズと密度を推論できます。

## 状態と環境の設定

これらは「サイトを X のように動作させる」ワークフローに役立ちます：

- クッキー：`cookies`、`cookies set`、`cookies clear`
- ストレージ：`storage local|session get|set|clear`
- オフライン：`set offline on|off`
- ヘッダー：`set headers --headers-json '{"X-Debug":"1"}'`（レガシー `set headers --json '{"X-Debug":"1"}'` も引き続きサポート）
- HTTP Basic 認証：`set credentials user pass`（または `--clear`）
- ジオロケーション：`set geo <lat> <lon> --origin "https://example.com"`（または `--clear`）
- メディア：`set media dark|light|no-preference|none`
- タイムゾーン / ロケール：`set timezone ...`、`set locale ...`
- デバイス / ビューポート：
  - `set device "iPhone 14"`（Playwright デバイスプリセット）
  - `set viewport 1280 720`

## セキュリティとプライバシー

- openclaw ブラウザプロファイルにはログイン済みセッションが含まれる場合があります；機密として扱ってください。
- `browser act kind=evaluate` / `openclaw browser evaluate` と `wait --fn` はページコンテキストで任意の JavaScript を実行します。プロンプトインジェクションでこれを操作できます。不要な場合は `browser.evaluateEnabled=false` で無効にしてください。
- ログインとアンチボットの注意事項（X/Twitter など）については、[ブラウザログイン + X/Twitter 投稿](/tools/browser-login) を参照してください。
- Gateway/ノードホストをプライベートに保ってください（ループバックまたは tailnet のみ）。
- リモート CDP エンドポイントは強力です；トンネルを使用して保護してください。

厳格モードの例（デフォルトでプライベート/内部の宛先をブロック）：

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // オプションの完全一致許可
    },
  },
}
```

## トラブルシューティング

Linux 固有の問題（特に snap Chromium）については、[ブラウザトラブルシューティング](/tools/browser-linux-troubleshooting) を参照してください。

WSL2 Gateway + Windows Chrome のスプリットホストセットアップについては、[WSL2 + Windows + リモート Chrome CDP トラブルシューティング](/tools/browser-wsl2-windows-remote-cdp-troubleshooting) を参照してください。

## エージェントツール + 制御の仕組み

エージェントはブラウザ自動化のための**1つのツール**を取得します：

- `browser` — status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

マッピング：

- `browser snapshot` は安定した UI ツリー（AI または ARIA）を返します。
- `browser act` はスナップショットの `ref` ID を使用してクリック/入力/ドラッグ/選択を行います。
- `browser screenshot` はピクセルをキャプチャします（フルページまたは要素）。
- `browser` は以下を受け入れます：
  - `profile`：名前付きブラウザプロファイルを選択（openclaw、chrome、またはリモート CDP）。
  - `target`（`sandbox` | `host` | `node`）：ブラウザが存在する場所を選択。
  - サンドボックス化されたセッションでは、`target: "host"` には `agents.defaults.sandbox.browser.allowHostControl=true` が必要。
  - `target` が省略された場合：サンドボックス化されたセッションはデフォルトで `sandbox`、非サンドボックスセッションはデフォルトで `host`。
  - ブラウザ対応ノードが接続されている場合、`target="host"` または `target="node"` をピン留めしない限り、ツールは自動的にそれにルーティングする場合があります。

これによりエージェントは確定的に保たれ、壊れやすいセレクターを避けることができます。
