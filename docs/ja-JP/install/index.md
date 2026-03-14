---
summary: "OpenClaw のインストール — インストーラースクリプト、npm/pnpm、ソースから、Docker など"
read_when:
  - はじめに（Getting Started）以外のインストール方法が必要な場合
  - クラウドプラットフォームにデプロイしたい場合
  - 更新、移行、またはアンインストールが必要な場合
title: "インストール"
---

# インストール

既に [はじめに](/start/getting-started) を完了しましたか？それなら準備完了です — このページは代替のインストール方法、プラットフォーム固有の手順、およびメンテナンスのためのものです。

## システム要件

- **[Node 24（推奨）](/install/node)**（Node 22 LTS、現在 `22.16+`、互換性のためにサポート継続；[インストーラースクリプト](#install-methods)は Node が不足している場合に Node 24 をインストールします）
- macOS、Linux、または Windows
- `pnpm`はソースからビルドする場合のみ必要

<Note>
Windows では、[WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) 上で OpenClaw を実行することを強くお勧めします。
</Note>

## インストール方法

<Tip>
**インストーラースクリプト**が OpenClaw のインストールに推奨される方法です。Node の検出、インストール、オンボーディングを 1 ステップで処理します。
</Tip>

<Warning>
VPS/クラウドホストには、可能であればサードパーティの「ワンクリック」マーケットプレイスイメージを避けてください。クリーンなベース OS イメージ（例：Ubuntu LTS）を使用し、インストーラースクリプトで OpenClaw を自分でインストールしてください。
</Warning>

<AccordionGroup>
  <Accordion title="インストーラースクリプト" icon="rocket" defaultOpen>
    CLI をダウンロードし、npm でグローバルにインストールして、オンボーディングウィザードを起動します。

    <Tabs>
      <Tab title="macOS / Linux / WSL2">
        ```bash
        curl -fsSL https://openclaw.ai/install.sh | bash
        ```
      </Tab>
      <Tab title="Windows (PowerShell)">
        ```powershell
        iwr -useb https://openclaw.ai/install.ps1 | iex
        ```
      </Tab>
    </Tabs>

    以上です — スクリプトが Node の検出、インストール、オンボーディングを処理します。

    オンボーディングをスキップしてバイナリだけをインストールするには：

    <Tabs>
      <Tab title="macOS / Linux / WSL2">
        ```bash
        curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
        ```
      </Tab>
      <Tab title="Windows (PowerShell)">
        ```powershell
        & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
        ```
      </Tab>
    </Tabs>

    すべてのフラグ、環境変数、CI/自動化オプションについては、[インストーラーの内部](/install/installer) を参照してください。

  </Accordion>

  <Accordion title="npm / pnpm" icon="package">
    すでに Node を自分で管理している場合は、Node 24 をお勧めします。OpenClaw は互換性のために Node 22 LTS（現在 `22.16+`）もサポートしています：

    <Tabs>
      <Tab title="npm">
        ```bash
        npm install -g openclaw@latest
        openclaw onboard --install-daemon
        ```

        <Accordion title="sharp のビルドエラー？">
          Homebrew 経由で libvips をグローバルにインストールしている場合（macOS でよくある）に `sharp` が失敗する場合は、プリビルドバイナリを強制使用してください：

          ```bash
          SHARP_IGNORE_GLOBAL_LIBVIPS=1 npm install -g openclaw@latest
          ```

          `sharp: Please add node-gyp to your dependencies` と表示される場合は、ビルドツールをインストール（macOS：Xcode CLT + `npm install -g node-gyp`）するか、上記の環境変数を使用してください。
        </Accordion>
      </Tab>
      <Tab title="pnpm">
        ```bash
        pnpm add -g openclaw@latest
        pnpm approve-builds -g        # openclaw、node-llama-cpp、sharp などを承認
        openclaw onboard --install-daemon
        ```

        <Note>
        pnpm はビルドスクリプトを持つパッケージの明示的な承認が必要です。最初のインストールで「Ignored build scripts」の警告が表示された後、`pnpm approve-builds -g` を実行して一覧表示されたパッケージを選択してください。
        </Note>
      </Tab>
    </Tabs>

  </Accordion>

  <Accordion title="ソースから" icon="github">
    コントリビューターまたはローカルチェックアウトから実行したい方向け。

    <Steps>
      <Step title="クローンとビルド">
        [OpenClaw リポジトリ](https://github.com/openclaw/openclaw) をクローンしてビルドします：

        ```bash
        git clone https://github.com/openclaw/openclaw.git
        cd openclaw
        pnpm install
        pnpm ui:build
        pnpm build
        ```
      </Step>
      <Step title="CLI をリンク">
        `openclaw` コマンドをグローバルに使用可能にします：

        ```bash
        pnpm link --global
        ```

        または、リンクをスキップしてリポジトリ内から `pnpm openclaw ...` でコマンドを実行することもできます。
      </Step>
      <Step title="オンボーディングの実行">
        ```bash
        openclaw onboard --install-daemon
        ```
      </Step>
    </Steps>

    より詳細な開発ワークフローについては、[セットアップ](/start/setup) を参照してください。

  </Accordion>
</AccordionGroup>

## その他のインストール方法

<CardGroup cols={2}>
  <Card title="Docker" href="/install/docker" icon="container">
    コンテナ化またはヘッドレスデプロイメント。
  </Card>
  <Card title="Podman" href="/install/podman" icon="container">
    ルートレスコンテナ：`setup-podman.sh` を 1 度実行してから起動スクリプトを使用。
  </Card>
  <Card title="Nix" href="/install/nix" icon="snowflake">
    Nix による宣言型インストール。
  </Card>
  <Card title="Ansible" href="/install/ansible" icon="server">
    フリートの自動プロビジョニング。
  </Card>
  <Card title="Bun" href="/install/bun" icon="zap">
    Bun ランタイム経由の CLI のみの使用。
  </Card>
</CardGroup>

## インストール後

すべてが正常に動作していることを確認します：

```bash
openclaw doctor         # 設定の問題を確認
openclaw status         # ゲートウェイのステータス
openclaw dashboard      # ブラウザ UI を開く
```

カスタムランタイムパスが必要な場合は以下を使用します：

- `OPENCLAW_HOME`：ホームディレクトリベースの内部パス
- `OPENCLAW_STATE_DIR`：可変状態の場所
- `OPENCLAW_CONFIG_PATH`：設定ファイルの場所

優先順位と詳細については、[環境変数](/help/environment) を参照してください。

## トラブルシューティング：`openclaw` が見つからない

<Accordion title="PATH の診断と修正">
  クイック診断：

```bash
node -v
npm -v
npm prefix -g
echo "$PATH"
```

`$(npm prefix -g)/bin`（macOS/Linux）または `$(npm prefix -g)`（Windows）が `$PATH` に**含まれていない**場合、シェルはグローバル npm バイナリ（`openclaw` を含む）を見つけることができません。

修正 — シェルの起動ファイル（`~/.zshrc` または `~/.bashrc`）に追加してください：

```bash
export PATH="$(npm prefix -g)/bin:$PATH"
```

Windows では、`npm prefix -g` の出力を PATH に追加してください。

その後、新しいターミナルを開くか（zsh では `rehash` / bash では `hash -r`）してください。
</Accordion>

## 更新 / アンインストール

<CardGroup cols={3}>
  <Card title="更新" href="/install/updating" icon="refresh-cw">
    OpenClaw を最新の状態に保ちます。
  </Card>
  <Card title="移行" href="/install/migrating" icon="arrow-right">
    新しいマシンに移行します。
  </Card>
  <Card title="アンインストール" href="/install/uninstall" icon="trash-2">
    OpenClaw を完全に削除します。
  </Card>
</CardGroup>
