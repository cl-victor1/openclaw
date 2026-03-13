---
summary: "OpenClawのサンドボックスの仕組み: モード、スコープ、ワークスペースアクセス、およびイメージ"
title: サンドボックス
read_when: "サンドボックスの専用説明が必要な場合、またはagents.defaults.sandboxを調整する必要がある場合。"
status: active
---

# サンドボックス

OpenClawは**Dockerコンテナ内でツールを実行**して影響範囲を縮小できます。
これは**オプション**であり、設定（`agents.defaults.sandbox`または
`agents.list[].sandbox`）で制御されます。サンドボックスがオフの場合、ツールはホスト上で実行されます。
Gatewayはホスト上に留まり、ツールの実行は有効時に分離されたサンドボックス
内で行われます。

これは完璧なセキュリティ境界ではありませんが、モデルが不適切な動作をした場合の
ファイルシステムとプロセスアクセスを実質的に制限します。

## サンドボックスの対象

- ツール実行（`exec`、`read`、`write`、`edit`、`apply_patch`、`process`等）。
- オプションのサンドボックスブラウザ（`agents.defaults.sandbox.browser`）。
  - デフォルトでは、ブラウザツールが必要とする場合、サンドボックスブラウザは自動起動します（CDPが到達可能であることを保証）。
    `agents.defaults.sandbox.browser.autoStart`と`agents.defaults.sandbox.browser.autoStartTimeoutMs`で設定します。
  - デフォルトでは、サンドボックスブラウザコンテナはグローバルな`bridge`ネットワークの代わりに専用のDockerネットワーク（`openclaw-sandbox-browser`）を使用します。
    `agents.defaults.sandbox.browser.network`で設定します。
  - オプションの`agents.defaults.sandbox.browser.cdpSourceRange`はCIDR許可リストでコンテナエッジのCDPイングレスを制限します（例: `172.21.0.1/32`）。
  - noVNCオブザーバーアクセスはデフォルトでパスワード保護されています。OpenClawはローカルブートストラップページを提供し、URLフラグメント（クエリ/ヘッダーログではなく）にパスワードを含めてnoVNCを開く短命のトークンURLを発行します。
  - `agents.defaults.sandbox.browser.allowHostControl`はサンドボックスセッションがホストブラウザを明示的にターゲットできるようにします。
  - オプションの許可リストが`target: "custom"`をゲートします: `allowedControlUrls`、`allowedControlHosts`、`allowedControlPorts`。

サンドボックスの対象外:

- Gatewayプロセス自体。
- ホスト上での実行が明示的に許可されているツール（例: `tools.elevated`）。
  - **昇格execはホスト上で実行され、サンドボックスをバイパスします。**
  - サンドボックスがオフの場合、`tools.elevated`は実行を変更しません（すでにホスト上）。[昇格モード](/tools/elevated)を参照してください。

## モード

`agents.defaults.sandbox.mode`はサンドボックスが使用される**タイミング**を制御します:

- `"off"`: サンドボックスなし。
- `"non-main"`: **非メイン**セッションのみサンドボックス化（通常のチャットをホストで行いたい場合のデフォルト）。
- `"all"`: すべてのセッションがサンドボックス内で実行されます。
  注意: `"non-main"`はエージェントIDではなく`session.mainKey`（デフォルト`"main"`）に基づきます。
  グループ/チャンネルセッションは独自のキーを使用するため、非メインとしてカウントされサンドボックス化されます。

## スコープ

`agents.defaults.sandbox.scope`は**作成されるコンテナの数**を制御します:

- `"session"`（デフォルト）: セッションごとに1つのコンテナ。
- `"agent"`: エージェントごとに1つのコンテナ。
- `"shared"`: すべてのサンドボックスセッションで共有される1つのコンテナ。

## ワークスペースアクセス

`agents.defaults.sandbox.workspaceAccess`は**サンドボックスが参照できるもの**を制御します:

- `"none"`（デフォルト）: ツールは`~/.openclaw/sandboxes`の下のサンドボックスワークスペースを参照します。
- `"ro"`: エージェントワークスペースを`/agent`に読み取り専用でマウント（`write`/`edit`/`apply_patch`を無効化）。
- `"rw"`: エージェントワークスペースを`/workspace`に読み書き可能でマウント。

受信メディアはアクティブなサンドボックスワークスペース（`media/inbound/*`）にコピーされます。
スキルに関する注意: `read`ツールはサンドボックスルートです。`workspaceAccess: "none"`の場合、
OpenClawは対象のスキルをサンドボックスワークスペース（`.../skills`）にミラーリングして
読み取れるようにします。`"rw"`の場合、ワークスペースのスキルは
`/workspace/skills`から読み取り可能です。

## カスタムバインドマウント

`agents.defaults.sandbox.docker.binds`は追加のホストディレクトリをコンテナにマウントします。
形式: `host:container:mode`（例: `"/home/user/source:/source:rw"`）。

グローバルとエージェントごとのバインドは**マージ**されます（置換ではありません）。`scope: "shared"`の場合、エージェントごとのバインドは無視されます。

`agents.defaults.sandbox.browser.binds`は追加のホストディレクトリを**サンドボックスブラウザ**コンテナにのみマウントします。

- 設定時（`[]`を含む）、ブラウザコンテナの`agents.defaults.sandbox.docker.binds`を置換します。
- 省略時、ブラウザコンテナは`agents.defaults.sandbox.docker.binds`にフォールバックします（後方互換）。

例（読み取り専用ソース + 追加データディレクトリ）:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

セキュリティに関する注意:

- バインドはサンドボックスのファイルシステムをバイパスします: 設定したモード（`:ro`または`:rw`）でホストパスを公開します。
- OpenClawは危険なバインドソースをブロックします（例: `docker.sock`、`/etc`、`/proc`、`/sys`、`/dev`、およびそれらを公開する親マウント）。
- 機密マウント（シークレット、SSHキー、サービス資格情報）は絶対に必要でない限り`:ro`にすべきです。
- ワークスペースへの読み取りアクセスのみが必要な場合は`workspaceAccess: "ro"`と組み合わせます。バインドモードは独立したままです。
- バインドがツールポリシーおよび昇格execとどのように相互作用するかについては、[サンドボックス vs ツールポリシー vs 昇格](/gateway/sandbox-vs-tool-policy-vs-elevated)を参照してください。

## イメージ + セットアップ

デフォルトイメージ: `openclaw-sandbox:bookworm-slim`

一度ビルドします:

```bash
scripts/sandbox-setup.sh
```

注意: デフォルトイメージにはNodeが**含まれていません**。スキルがNode（または
他のランタイム）を必要とする場合、カスタムイメージをベイクするか、
`sandbox.docker.setupCommand`経由でインストールします（ネットワークエグレス + 書き込み可能なルート +
rootユーザーが必要）。

共通ツール（例えば`curl`、`jq`、`nodejs`、`python3`、`git`）を含むより機能的なサンドボックスイメージが必要な場合は、以下をビルドします:

```bash
scripts/sandbox-common-setup.sh
```

その後、`agents.defaults.sandbox.docker.image`を
`openclaw-sandbox-common:bookworm-slim`に設定します。

サンドボックスブラウザイメージ:

```bash
scripts/sandbox-browser-setup.sh
```

デフォルトでは、サンドボックスコンテナは**ネットワークなし**で実行されます。
`agents.defaults.sandbox.docker.network`でオーバーライドします。

バンドルされたサンドボックスブラウザイメージは、コンテナ化されたワークロード用の
保守的なChromium起動デフォルトも適用します。現在のコンテナデフォルトには以下が含まれます:

- `--remote-debugging-address=127.0.0.1`
- `--remote-debugging-port=<OPENCLAW_BROWSER_CDP_PORTから導出>`
- `--user-data-dir=${HOME}/.chrome`
- `--no-first-run`
- `--no-default-browser-check`
- `--disable-3d-apis`
- `--disable-gpu`
- `--disable-dev-shm-usage`
- `--disable-background-networking`
- `--disable-extensions`
- `--disable-features=TranslateUI`
- `--disable-breakpad`
- `--disable-crash-reporter`
- `--disable-software-rasterizer`
- `--no-zygote`
- `--metrics-recording-only`
- `--renderer-process-limit=2`
- `noSandbox`が有効な場合の`--no-sandbox`と`--disable-setuid-sandbox`。
- 3つのグラフィックスハードニングフラグ（`--disable-3d-apis`、
  `--disable-software-rasterizer`、`--disable-gpu`）はオプションで、
  コンテナがGPUサポートを持たない場合に有用です。ワークロードがWebGLまたは他の3D/ブラウザ機能を必要とする場合は`OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0`を設定してください。
- `--disable-extensions`はデフォルトで有効で、拡張機能に依存するフローでは
  `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0`で無効にできます。
- `--renderer-process-limit=2`は
  `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>`で制御され、`0`はChromiumのデフォルトを維持します。

異なるランタイムプロファイルが必要な場合は、カスタムブラウザイメージを使用し、
独自のエントリーポイントを提供してください。ローカル（非コンテナ）のChromiumプロファイルには、
`browser.extraArgs`を使用して追加の起動フラグを付加します。

セキュリティデフォルト:

- `network: "host"`はブロックされます。
- `network: "container:<id>"`はデフォルトでブロックされます（ネームスペース結合バイパスリスク）。
- 緊急回避オーバーライド: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`。

Dockerインストールとコンテナ化されたGatewayはこちら:
[Docker](/install/docker)

Docker Gatewayデプロイメントでは、`docker-setup.sh`がサンドボックス設定をブートストラップできます。
`OPENCLAW_SANDBOX=1`（または`true`/`yes`/`on`）を設定してそのパスを有効にします。
`OPENCLAW_DOCKER_SOCKET`でソケットの場所をオーバーライドできます。完全なセットアップと環境変数リファレンス: [Docker](/install/docker#enable-agent-sandbox-for-docker-gateway-opt-in)。

## setupCommand（コンテナの初回セットアップ）

`setupCommand`はサンドボックスコンテナ作成後に**1回だけ**実行されます（実行のたびではありません）。
コンテナ内で`sh -lc`経由で実行されます。

パス:

- グローバル: `agents.defaults.sandbox.docker.setupCommand`
- エージェントごと: `agents.list[].sandbox.docker.setupCommand`

よくある落とし穴:

- デフォルトの`docker.network`は`"none"`（エグレスなし）なので、パッケージインストールは失敗します。
- `docker.network: "container:<id>"`は`dangerouslyAllowContainerNamespaceJoin: true`が必要で、緊急回避専用です。
- `readOnlyRoot: true`は書き込みを防止します。`readOnlyRoot: false`に設定するか、カスタムイメージをベイクしてください。
- パッケージインストールには`user`がrootである必要があります（`user`を省略するか`user: "0:0"`に設定）。
- サンドボックスexecはホストの`process.env`を継承**しません**。スキルAPIキーには
  `agents.defaults.sandbox.docker.env`（またはカスタムイメージ）を使用してください。

## ツールポリシー + エスケープハッチ

ツールの許可/拒否ポリシーはサンドボックスルールの前に適用されます。ツールが
グローバルまたはエージェントごとに拒否されている場合、サンドボックスはそれを復活させません。

`tools.elevated`はホスト上で`exec`を実行する明示的なエスケープハッチです。
`/exec`ディレクティブは承認された送信者にのみ適用され、セッションごとに永続化します。`exec`を完全に無効にするには、ツールポリシーのdenyを使用します（[サンドボックス vs ツールポリシー vs 昇格](/gateway/sandbox-vs-tool-policy-vs-elevated)を参照）。

デバッグ:

- `openclaw sandbox explain`を使用して、有効なサンドボックスモード、ツールポリシー、修正用の設定キーを検査します。
- 「なぜこれがブロックされるのか？」のメンタルモデルについては[サンドボックス vs ツールポリシー vs 昇格](/gateway/sandbox-vs-tool-policy-vs-elevated)を参照してください。
  ロックダウンを維持してください。

## マルチエージェントオーバーライド

各エージェントはサンドボックス + ツールをオーバーライドできます:
`agents.list[].sandbox`と`agents.list[].tools`（さらにサンドボックスツールポリシー用の`agents.list[].tools.sandbox.tools`）。
優先順位については[マルチエージェントサンドボックス＆ツール](/tools/multi-agent-sandbox-tools)を参照してください。

## 最小限の有効化例

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## 関連ドキュメント

- [サンドボックス設定](/gateway/configuration#agentsdefaults-sandbox)
- [マルチエージェントサンドボックス＆ツール](/tools/multi-agent-sandbox-tools)
- [セキュリティ](/gateway/security)
