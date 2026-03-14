---
summary: "OpenClaw のオプションの Docker ベースのセットアップとオンボーディング"
read_when:
  - ローカルインストールの代わりにコンテナ化されたゲートウェイが必要な場合
  - Docker フローを検証している場合
title: "Docker"
---

# Docker（オプション）

Docker は**オプション**です。コンテナ化されたゲートウェイが必要な場合、または Docker フローを検証したい場合にのみ使用してください。

## Docker は私に合っていますか？

- **はい**：隔離された、使い捨てのゲートウェイ環境が必要な場合、またはローカルインストールなしのホストで OpenClaw を実行したい場合。
- **いいえ**：自分のマシンで実行していて、最速の開発ループが必要な場合。代わりに通常のインストールフローを使用してください。
- **サンドボックスの注意**：エージェントのサンドボックス化も Docker を使用しますが、完全なゲートウェイを Docker で実行する**必要はありません**。[サンドボックス化](/gateway/sandboxing) を参照してください。

このガイドで説明する内容：

- コンテナ化ゲートウェイ（Docker 内の完全な OpenClaw）
- セッションごとのエージェントサンドボックス（ホストゲートウェイ + Docker 隔離エージェントツール）

サンドボックス化の詳細：[サンドボックス化](/gateway/sandboxing)

## 要件

- Docker Desktop（または Docker Engine）+ Docker Compose v2
- イメージビルド用に少なくとも 2 GB の RAM（`pnpm install` は 1 GB ホストで exit 137 による OOM-kill が発生する可能性があります）
- イメージ + ログ用の十分なディスク容量
- VPS/パブリックホストで実行する場合は、[ネットワーク公開のセキュリティ強化](/gateway/security#04-network-exposure-bind--port--firewall)（特に Docker の `DOCKER-USER` ファイアウォールポリシー）を確認してください。

## コンテナ化ゲートウェイ（Docker Compose）

### クイックスタート（推奨）

<Note>
ここでの Docker デフォルトはバインドモード（`lan`/`loopback`）を想定しており、ホストエイリアスではありません。`gateway.bind` にはバインドモードの値（例：`lan` または `loopback`）を使用し、`0.0.0.0` や `localhost` などのホストエイリアスは使用しないでください。
</Note>

リポジトリのルートから：

```bash
./docker-setup.sh
```

このスクリプトは：

- ゲートウェイイメージをローカルでビルドします（`OPENCLAW_IMAGE` が設定されている場合はリモートイメージをプルします）
- オンボーディングウィザードを実行します
- オプションのプロバイダー設定のヒントを表示します
- Docker Compose でゲートウェイを起動します
- ゲートウェイトークンを生成し、`.env` に書き込みます

オプションの環境変数：

- `OPENCLAW_IMAGE` — ローカルでビルドする代わりにリモートイメージを使用します（例：`ghcr.io/openclaw/openclaw:latest`）
- `OPENCLAW_DOCKER_APT_PACKAGES` — ビルド中に追加の apt パッケージをインストールします
- `OPENCLAW_EXTENSIONS` — ビルド時に拡張機能の依存関係をプリインストールします（スペース区切りの拡張機能名、例：`diagnostics-otel matrix`）
- `OPENCLAW_EXTRA_MOUNTS` — 追加のホストバインドマウントを追加します
- `OPENCLAW_HOME_VOLUME` — 名前付きボリュームに `/home/node` を永続化します
- `OPENCLAW_SANDBOX` — Docker ゲートウェイサンドボックスブートストラップを有効にします。明示的な truthy 値のみが有効です：`1`、`true`、`yes`、`on`
- `OPENCLAW_INSTALL_DOCKER_CLI` — ローカルイメージビルド用のビルド引数パス（`1` はイメージに Docker CLI をインストール）。`docker-setup.sh` はローカルビルドで `OPENCLAW_SANDBOX=1` のときに自動的に設定します。
- `OPENCLAW_DOCKER_SOCKET` — Docker ソケットパスをオーバーライドします（デフォルト：`DOCKER_HOST=unix://...` パス、それ以外は `/var/run/docker.sock`）
- `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` — 緊急対応：CLI/オンボーディングクライアントパスの信頼されたプライベートネットワーク `ws://` ターゲットを許可します（デフォルトはループバックのみ）
- `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` — WebGL/3D 互換性が必要な場合に、コンテナブラウザの強化フラグ `--disable-3d-apis`、`--disable-software-rasterizer`、`--disable-gpu` を無効にします。
- `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` — ブラウザフローに拡張機能が必要な場合に拡張機能を有効のままにします（デフォルトではサンドボックスブラウザで拡張機能を無効にします）。
- `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` — Chromium レンダラープロセス制限を設定します；`0` に設定するとフラグをスキップして Chromium のデフォルト動作を使用します。

完了後：

- ブラウザで `http://127.0.0.1:18789/` を開きます。
- コントロール UI（設定 → token）にトークンを貼り付けます。
- URL が再度必要ですか？`docker compose run --rm openclaw-cli dashboard --no-open` を実行してください。

### Docker ゲートウェイでのエージェントサンドボックスの有効化（オプトイン）

`docker-setup.sh` は Docker デプロイメントの `agents.defaults.sandbox.*` もブートストラップできます。

有効にするには：

```bash
export OPENCLAW_SANDBOX=1
./docker-setup.sh
```

カスタムソケットパス（例：ルートレス Docker）：

```bash
export OPENCLAW_SANDBOX=1
export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
./docker-setup.sh
```

注意事項：

- スクリプトはサンドボックスの前提条件が通過した後にのみ `docker.sock` をマウントします。
- サンドボックスのセットアップが完了できない場合、スクリプトは `agents.defaults.sandbox.mode` を `off` にリセットして、再実行時の古い/壊れたサンドボックス設定を避けます。
- `Dockerfile.sandbox` が存在しない場合、スクリプトは警告を表示して続行します；必要な場合は `scripts/sandbox-setup.sh` で `openclaw-sandbox:bookworm-slim` をビルドしてください。
- 非ローカルの `OPENCLAW_IMAGE` 値の場合、イメージにはサンドボックス実行のための Docker CLI サポートが既に含まれている必要があります。

### 自動化/CI（非インタラクティブ、TTY ノイズなし）

スクリプトと CI では、`-T` を付けて Compose の疑似 TTY 割り当てを無効にします：

```bash
docker compose run -T --rm openclaw-cli gateway probe
docker compose run -T --rm openclaw-cli devices list --json
```

### 共有ネットワークのセキュリティ注意（CLI + ゲートウェイ）

`openclaw-cli` は `network_mode: "service:openclaw-gateway"` を使用しているため、CLI コマンドが Docker の `127.0.0.1` でゲートウェイに確実に到達できます。

これを共有信頼境界として扱ってください：ループバックバインディングはこれら 2 つのコンテナ間の分離ではありません。より強い分離が必要な場合は、バンドルされた `openclaw-cli` サービスの代わりに別のコンテナ/ホストネットワークパスからコマンドを実行してください。

ホスト上の書き込みパス：

- `~/.openclaw/`
- `~/.openclaw/workspace`

VPS で実行している場合は、[Hetzner（Docker VPS）](/install/hetzner) を参照してください。

### リモートイメージを使用する（ローカルビルドをスキップ）

公式のプリビルドイメージは次の場所で公開されています：

- [GitHub Container Registry パッケージ](https://github.com/openclaw/openclaw/pkgs/container/openclaw)

イメージ名 `ghcr.io/openclaw/openclaw` を使用してください（同様の名前の Docker Hub イメージではありません）。

一般的なタグ：

- `main` — `main` からの最新ビルド
- `<version>` — リリースタグビルド（例：`2026.2.26`）
- `latest` — 最新の安定リリースタグ

### ベースイメージのメタデータ

メインの Docker イメージは現在以下を使用しています：

- `node:24-bookworm`

Docker イメージは OCI ベースイメージアノテーションを公開しています：

- `org.opencontainers.image.base.name=docker.io/library/node:24-bookworm`
- `org.opencontainers.image.base.digest=sha256:3a09aa6354567619221ef6c45a5051b671f953f0a1924d1f819ffb236e520e6b`
- `org.opencontainers.image.source=https://github.com/openclaw/openclaw`
- `org.opencontainers.image.url=https://openclaw.ai`
- `org.opencontainers.image.documentation=https://docs.openclaw.ai/install/docker`
- `org.opencontainers.image.licenses=MIT`
- `org.opencontainers.image.title=OpenClaw`
- `org.opencontainers.image.description=OpenClaw gateway and CLI runtime container image`
- `org.opencontainers.image.revision=<git-sha>`
- `org.opencontainers.image.version=<tag-or-main>`
- `org.opencontainers.image.created=<rfc3339 timestamp>`

参考：[OCI image annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)

デフォルトでは、セットアップスクリプトはソースからイメージをビルドします。プリビルドイメージをプルするには、スクリプトを実行する前に `OPENCLAW_IMAGE` を設定します：

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
./docker-setup.sh
```

### シェルヘルパー（オプション）

日常の Docker 管理を簡単にするために、`ClawDock` をインストールします：

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/shell-helpers/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
```

**シェル設定（zsh）に追加：**

```bash
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

その後、`clawdock-start`、`clawdock-stop`、`clawdock-dashboard` などを使用できます。すべてのコマンドは `clawdock-help` を実行してください。

詳細は [`ClawDock` ヘルパー README](https://github.com/openclaw/openclaw/blob/main/scripts/shell-helpers/README.md) を参照してください。

### 手動フロー（compose）

```bash
docker build -t openclaw:local -f Dockerfile .
docker compose run --rm openclaw-cli onboard
docker compose up -d openclaw-gateway
```

注意：リポジトリのルートから `docker compose ...` を実行してください。`OPENCLAW_EXTRA_MOUNTS` または `OPENCLAW_HOME_VOLUME` を有効にした場合、セットアップスクリプトは `docker-compose.extra.yml` を書き込みます；他の場所で Compose を実行するときはそれを含めてください：

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml <command>
```

### コントロール UI トークン + ペアリング（Docker）

「unauthorized」または「disconnected (1008): pairing required」が表示される場合は、新しいダッシュボードリンクを取得してブラウザデバイスを承認してください：

```bash
docker compose run --rm openclaw-cli dashboard --no-open
docker compose run --rm openclaw-cli devices list
docker compose run --rm openclaw-cli devices approve <requestId>
```

詳細：[ダッシュボード](/web/dashboard)、[デバイス](/cli/devices)。

### 追加マウント（オプション）

追加のホストディレクトリをコンテナにマウントしたい場合は、`docker-setup.sh` を実行する前に `OPENCLAW_EXTRA_MOUNTS` を設定します。これはカンマ区切りの Docker バインドマウントリストを受け取り、`docker-compose.extra.yml` を生成することで `openclaw-gateway` と `openclaw-cli` の両方に適用します。

例：

```bash
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

注意：

- macOS/Windows では、パスを Docker Desktop と共有する必要があります。
- 各エントリは `source:target[:options]` 形式でスペース、タブ、改行なし。
- `OPENCLAW_EXTRA_MOUNTS` を編集した場合は、`docker-setup.sh` を再実行して追加の Compose ファイルを再生成してください。
- `docker-compose.extra.yml` は自動生成されます。手動編集しないでください。

### コンテナホーム全体を永続化する（オプション）

コンテナ再作成後も `/home/node` を永続化したい場合は、`OPENCLAW_HOME_VOLUME` で名前付きボリュームを設定します。これは Docker ボリュームを作成して `/home/node` にマウントし、標準の設定/ワークスペースバインドマウントを維持します。バインドパスではなく名前付きボリュームをここで使用してください；バインドマウントには `OPENCLAW_EXTRA_MOUNTS` を使用してください。

例：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

追加マウントと組み合わせることもできます：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
export OPENCLAW_EXTRA_MOUNTS="$HOME/.codex:/home/node/.codex:ro,$HOME/github:/home/node/github:rw"
./docker-setup.sh
```

注意：

- 名前付きボリュームは `^[A-Za-z0-9][A-Za-z0-9_.-]*$` に一致する必要があります。
- `OPENCLAW_HOME_VOLUME` を変更した場合は、`docker-setup.sh` を再実行して追加の Compose ファイルを再生成してください。
- 名前付きボリュームは `docker volume rm <name>` で削除するまで永続化されます。

### 追加の apt パッケージをインストールする（オプション）

イメージ内にシステムパッケージが必要な場合（例：ビルドツールやメディアライブラリ）は、`docker-setup.sh` を実行する前に `OPENCLAW_DOCKER_APT_PACKAGES` を設定します。これはイメージビルド中にパッケージをインストールするため、コンテナが削除されても永続化されます。

例：

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="ffmpeg build-essential"
./docker-setup.sh
```

注意：

- スペース区切りの apt パッケージ名リストを受け取ります。
- `OPENCLAW_DOCKER_APT_PACKAGES` を変更した場合は、`docker-setup.sh` を再実行してイメージを再ビルドしてください。

### 拡張機能の依存関係をプリインストールする（オプション）

独自の `package.json` を持つ拡張機能（例：`diagnostics-otel`、`matrix`、`msteams`）は、最初の読み込み時に npm 依存関係をインストールします。代わりにそれらの依存関係をイメージに焼き込むには、`docker-setup.sh` を実行する前に `OPENCLAW_EXTENSIONS` を設定します：

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel matrix"
./docker-setup.sh
```

または直接ビルドする場合：

```bash
docker build --build-arg OPENCLAW_EXTENSIONS="diagnostics-otel matrix" .
```

注意：

- スペース区切りの拡張機能ディレクトリ名（`extensions/` 下）のリストを受け取ります。
- `package.json` を持つ拡張機能のみが影響を受けます；`package.json` のない軽量プラグインは無視されます。
- `OPENCLAW_EXTENSIONS` を変更した場合は、`docker-setup.sh` を再実行してイメージを再ビルドしてください。

### パワーユーザー / フル機能コンテナ（オプトイン）

デフォルトの Docker イメージは**セキュリティ優先**で、非 root の `node` ユーザーとして実行されます。これにより攻撃面は小さくなりますが、以下のことができません：

- 実行時のシステムパッケージインストール
- デフォルトでは Homebrew なし
- バンドルされた Chromium/Playwright ブラウザなし

より機能豊富なコンテナが必要な場合は、以下のオプトインノブを使用してください：

1. **`/home/node` を永続化**してブラウザのダウンロードとツールキャッシュを保持：

```bash
export OPENCLAW_HOME_VOLUME="openclaw_home"
./docker-setup.sh
```

2. **システム依存関係をイメージに焼き込む**（再現可能 + 永続的）：

```bash
export OPENCLAW_DOCKER_APT_PACKAGES="git curl jq"
./docker-setup.sh
```

3. **`npx` なしで Playwright ブラウザをインストール**（npm オーバーライドの競合を避ける）：

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

Playwright にシステム依存関係をインストールさせる必要がある場合は、実行時に `--with-deps` を使用する代わりに `OPENCLAW_DOCKER_APT_PACKAGES` でイメージを再ビルドしてください。

4. **Playwright ブラウザのダウンロードを永続化**：

- `docker-compose.yml` に `PLAYWRIGHT_BROWSERS_PATH=/home/node/.cache/ms-playwright` を設定します。
- `OPENCLAW_HOME_VOLUME` で `/home/node` が永続化されることを確認するか、`OPENCLAW_EXTRA_MOUNTS` で `/home/node/.cache/ms-playwright` をマウントしてください。

### 権限 + EACCES

イメージは `node`（uid 1000）として実行されます。`/home/node/.openclaw` で権限エラーが発生する場合は、ホストバインドマウントが uid 1000 によって所有されていることを確認してください。

例（Linux ホスト）：

```bash
sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
```

便宜上 root として実行することを選択する場合は、そのセキュリティトレードオフを受け入れることになります。

### 高速再ビルド（推奨）

再ビルドを高速化するには、依存関係レイヤーがキャッシュされるように Dockerfile を並べてください。これにより、ロックファイルが変更されない限り `pnpm install` の再実行を避けられます：

```dockerfile
FROM node:24-bookworm

# Bun をインストール（ビルドスクリプトに必要）
RUN curl -fsSL https://bun.sh/install | bash
ENV PATH="/root/.bun/bin:${PATH}"

RUN corepack enable

WORKDIR /app

# パッケージメタデータが変更されない限り依存関係をキャッシュ
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

### チャンネル設定（オプション）

CLI コンテナを使用してチャンネルを設定し、必要に応じてゲートウェイを再起動します。

WhatsApp（QR）：

```bash
docker compose run --rm openclaw-cli channels login
```

Telegram（bot token）：

```bash
docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"
```

Discord（bot token）：

```bash
docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
```

ドキュメント：[WhatsApp](/channels/whatsapp)、[Telegram](/channels/telegram)、[Discord](/channels/discord)

### OpenAI Codex OAuth（ヘッドレス Docker）

ウィザードで OpenAI Codex OAuth を選択すると、ブラウザ URL が開かれ、`http://127.0.0.1:1455/auth/callback` でコールバックのキャプチャが試みられます。Docker またはヘッドレスセットアップでは、そのコールバックがブラウザエラーを表示する場合があります。到達した完全なリダイレクト URL をコピーしてウィザードに貼り付けて認証を完了させてください。

### ヘルスチェック

コンテナプローブエンドポイント（認証不要）：

```bash
curl -fsS http://127.0.0.1:18789/healthz
curl -fsS http://127.0.0.1:18789/readyz
```

エイリアス：`/health` と `/ready`。

`/healthz` は「ゲートウェイプロセスが稼働中」の浅い liveness プローブです。
`/readyz` は起動グレース中は ready のまま維持され、グレース後に必要な管理チャンネルがまだ接続されていない場合、またはその後切断された場合にのみ `503` になります。

認証済みの詳細なヘルススナップショット（ゲートウェイ + チャンネル）：

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### E2E スモークテスト（Docker）

```bash
scripts/e2e/onboard-docker.sh
```

### QR インポートスモークテスト（Docker）

```bash
pnpm test:docker:qr
```

### LAN とループバック（Docker Compose）

`docker-setup.sh` は `OPENCLAW_GATEWAY_BIND=lan` をデフォルトに設定するため、Docker ポートパブリッシングで `http://127.0.0.1:18789` へのホストアクセスが機能します。

- `lan`（デフォルト）：ホストブラウザ + ホスト CLI は公開されたゲートウェイポートに到達できます。
- `loopback`：コンテナのネットワーク名前空間内のプロセスのみがゲートウェイに直接到達できます；ホストに公開されたポートアクセスが失敗する場合があります。

`gateway.bind` にはバインドモードの値（`lan` / `loopback` / `custom` / `tailnet` / `auto`）を使用し、ホストエイリアス（`0.0.0.0`、`127.0.0.1`、`localhost`、`::`、`::1`）は使用しないでください。

Docker CLI コマンドから `Gateway target: ws://172.x.x.x:18789` や繰り返しの `pairing required` エラーが表示される場合：

```bash
docker compose run --rm openclaw-cli config set gateway.mode local
docker compose run --rm openclaw-cli config set gateway.bind lan
docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
```

### 注意事項

- ゲートウェイバインドはコンテナ使用時のデフォルトとして `lan` に設定されます（`OPENCLAW_GATEWAY_BIND`）。
- Dockerfile CMD は `--allow-unconfigured` を使用します；`gateway.mode` が `local` でないマウントされた設定でも起動します。ガードを強制するには CMD をオーバーライドしてください。
- ゲートウェイコンテナはセッションの信頼できる情報源です（`~/.openclaw/agents/<agentId>/sessions/`）。

### ストレージモデル

- **永続的なホストデータ：** Docker Compose は `OPENCLAW_CONFIG_DIR` を `/home/node/.openclaw` に、`OPENCLAW_WORKSPACE_DIR` を `/home/node/.openclaw/workspace` にバインドマウントするため、これらのパスはコンテナ交換後も維持されます。
- **一時的なサンドボックス tmpfs：** `agents.defaults.sandbox` が有効な場合、サンドボックスコンテナは `/tmp`、`/var/tmp`、`/run` に `tmpfs` を使用します。これらのマウントはトップレベルの Compose スタックとは別で、サンドボックスコンテナと共に消えます。
- **ディスク増加のホットスポット：** `media/`、`agents/<agentId>/sessions/sessions.json`、トランスクリプト JSONL ファイル、`cron/runs/*.jsonl`、`/tmp/openclaw/`（または設定済みの `logging.file`）下のローリングファイルログを監視してください。

## エージェントサンドボックス（ホストゲートウェイ + Docker ツール）

詳細：[サンドボックス化](/gateway/sandboxing)

### 何をするか

`agents.defaults.sandbox` が有効な場合、**非メインセッション**は Docker コンテナ内でツールを実行します。ゲートウェイはホストに留まりますが、ツールの実行は隔離されます：

- scope：デフォルトで `"agent"`（エージェントごとに 1 つのコンテナ + ワークスペース）
- scope：セッションごとの隔離には `"session"`
- スコープごとのワークスペースフォルダーが `/workspace` にマウントされます
- オプションのエージェントワークスペースアクセス（`agents.defaults.sandbox.workspaceAccess`）
- 許可/拒否ツールポリシー（拒否優先）
- インバウンドメディアはアクティブなサンドボックスワークスペース（`media/inbound/*`）にコピーされるため、ツールが読み取れます（`workspaceAccess: "rw"` では、これはエージェントワークスペースに配置されます）

警告：`scope: "shared"` はセッション間の隔離を無効にします。すべてのセッションが 1 つのコンテナと 1 つのワークスペースを共有します。

### エージェントごとのサンドボックスプロファイル（マルチエージェント）

マルチエージェントルーティングを使用する場合、各エージェントはサンドボックス + ツール設定をオーバーライドできます：`agents.list[].sandbox` と `agents.list[].tools`（加えて `agents.list[].tools.sandbox.tools`）。これにより、1 つのゲートウェイで混合アクセスレベルを実行できます：

- フルアクセス（個人エージェント）
- 読み取り専用ツール + 読み取り専用ワークスペース（家族/仕事エージェント）
- ファイルシステム/シェルツールなし（パブリックエージェント）

例、優先順位、トラブルシューティングについては [マルチエージェントサンドボックスとツール](/tools/multi-agent-sandbox-tools) を参照してください。

### デフォルト動作

- イメージ：`openclaw-sandbox:bookworm-slim`
- エージェントごとに 1 つのコンテナ
- エージェントワークスペースアクセス：`workspaceAccess: "none"`（デフォルト）は `~/.openclaw/sandboxes` を使用
  - `"ro"` はサンドボックスワークスペースを `/workspace` に保ち、エージェントワークスペースを `/agent` に読み取り専用でマウントします（`write`/`edit`/`apply_patch` を無効化）
  - `"rw"` はエージェントワークスペースを `/workspace` に読み書き可能でマウントします
- 自動プルーニング：アイドル > 24 時間、または経過時間 > 7 日
- ネットワーク：デフォルトで `none`（出力が必要な場合は明示的にオプトイン）
  - `host` はブロックされます。
  - `container:<id>` はデフォルトでブロックされます（名前空間参加リスク）。
- デフォルト許可：`exec`、`process`、`read`、`write`、`edit`、`sessions_list`、`sessions_history`、`sessions_send`、`sessions_spawn`、`session_status`
- デフォルト拒否：`browser`、`canvas`、`nodes`、`cron`、`discord`、`gateway`

### サンドボックスの有効化

`setupCommand` でパッケージをインストールする予定がある場合は注意してください：

- デフォルトの `docker.network` は `"none"`（出力なし）です。
- `docker.network: "host"` はブロックされます。
- `docker.network: "container:<id>"` はデフォルトでブロックされます。
- 緊急対応オーバーライド：`agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`。
- `readOnlyRoot: true` はパッケージインストールをブロックします。
- `apt-get` には `user` が root である必要があります（`user` を省略するか `user: "0:0"` に設定）。

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared（デフォルトは agent）
        workspaceAccess: "none", // none | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
        },
        prune: {
          idleHours: 24, // 0 はアイドルプルーニングを無効化
          maxAgeDays: 7, // 0 は最大経過時間プルーニングを無効化
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

強化設定は `agents.defaults.sandbox.docker` 下にあります：
`network`、`user`、`pidsLimit`、`memory`、`memorySwap`、`cpus`、`ulimits`、
`seccompProfile`、`apparmorProfile`、`dns`、`extraHosts`、
`dangerouslyAllowContainerNamespaceJoin`（緊急対応のみ）。

マルチエージェント：`agents.list[].sandbox.{docker,browser,prune}.*` でエージェントごとに `agents.defaults.sandbox.{docker,browser,prune}.*` をオーバーライドします
（`agents.defaults.sandbox.scope` / `agents.list[].sandbox.scope` が `"shared"` の場合は無視されます）。

### デフォルトサンドボックスイメージのビルド

```bash
scripts/sandbox-setup.sh
```

これは `Dockerfile.sandbox` を使用して `openclaw-sandbox:bookworm-slim` をビルドします。

### サンドボックス共通イメージ（オプション）

一般的なビルドツール（Node、Go、Rust など）を含むサンドボックスイメージが必要な場合は、共通イメージをビルドします：

```bash
scripts/sandbox-common-setup.sh
```

これは `openclaw-sandbox-common:bookworm-slim` をビルドします。使用するには：

```json5
{
  agents: {
    defaults: {
      sandbox: { docker: { image: "openclaw-sandbox-common:bookworm-slim" } },
    },
  },
}
```

### サンドボックスブラウザイメージ

サンドボックス内でブラウザツールを実行するには、ブラウザイメージをビルドします：

```bash
scripts/sandbox-browser-setup.sh
```

これは `Dockerfile.sandbox-browser` を使用して `openclaw-sandbox-browser:bookworm-slim` をビルドします。コンテナは CDP が有効な Chromium とオプションの noVNC オブザーバー（Xvfb 経由のヘッドフル）を実行します。

注意事項：

- ヘッドフル（Xvfb）はヘッドレスよりもボットブロックを軽減します。
- `agents.defaults.sandbox.browser.headless=true` を設定することでヘッドレスを引き続き使用できます。
- フルデスクトップ環境（GNOME）は不要です；Xvfb がディスプレイを提供します。
- ブラウザコンテナはデフォルトでグローバルの `bridge` の代わりに専用の Docker ネットワーク（`openclaw-sandbox-browser`）を使用します。
- オプションの `agents.defaults.sandbox.browser.cdpSourceRange` は CIDR でコンテナエッジの CDP イングレスを制限します（例：`172.21.0.1/32`）。

設定の使用：

```json5
{
  agents: {
    defaults: {
      sandbox: {
        browser: { enabled: true },
      },
    },
  },
}
```

カスタムブラウザイメージ：

```json5
{
  agents: {
    defaults: {
      sandbox: { browser: { image: "my-openclaw-browser" } },
    },
  },
}
```

有効にすると、エージェントは以下を受け取ります：

- サンドボックスブラウザコントロール URL（`browser` ツール用）
- noVNC URL（有効で headless=false の場合）

注意：ツールの許可リストを使用する場合は、`browser` を追加（拒否から削除）しないと、ツールはブロックされたままです。
プルーニングルール（`agents.defaults.sandbox.prune`）はブラウザコンテナにも適用されます。

### カスタムサンドボックスイメージ

独自のイメージをビルドして設定で指定します：

```bash
docker build -t my-openclaw-sbx -f Dockerfile.sandbox .
```

```json5
{
  agents: {
    defaults: {
      sandbox: { docker: { image: "my-openclaw-sbx" } },
    },
  },
}
```

### ツールポリシー（許可/拒否）

- `deny` は `allow` よりも優先されます。
- `allow` が空の場合：すべてのツール（deny 以外）が利用可能です。
- `allow` が空でない場合：`allow` 内のツールのみが利用可能です（deny を除く）。

### プルーニング戦略

2 つの設定：

- `prune.idleHours`：X 時間使用されていないコンテナを削除します（0 = 無効）
- `prune.maxAgeDays`：X 日以上経過したコンテナを削除します（0 = 無効）

例：

- ビジーなセッションを保持しながらライフタイムを制限：
  `idleHours: 24`、`maxAgeDays: 7`
- プルーニングしない：
  `idleHours: 0`、`maxAgeDays: 0`

### セキュリティに関する注意

- ハードウォールは**ツール**（exec/read/write/edit/apply_patch）にのみ適用されます。
- browser/camera/canvas などのホスト専用ツールはデフォルトでブロックされます。
- サンドボックスで `browser` を許可すると**隔離が破れます**（ブラウザはホストで実行されます）。

## トラブルシューティング

- イメージが見つからない：[`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) でビルドするか、`agents.defaults.sandbox.docker.image` を設定してください。
- コンテナが実行されていない：セッションごとに要求に応じて自動作成されます。
- サンドボックスでの権限エラー：マウントされたワークスペースの所有権に合う UID:GID に `docker.user` を設定するか（またはワークスペースフォルダーを chown してください）。
- カスタムツールが見つからない：OpenClaw はコマンドを `sh -lc`（ログインシェル）で実行するため、`/etc/profile` がソースされ PATH がリセットされる場合があります。カスタムツールパスを先頭に追加するには `docker.env.PATH` を設定するか（例：`/custom/bin:/usr/local/share/npm-global/bin`）、Dockerfile の `/etc/profile.d/` 下にスクリプトを追加してください。
