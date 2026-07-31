---
read_when:
    - はじめにのクイックスタート以外のインストール方法が必要な場合
    - クラウドプラットフォームにデプロイしたい場合
    - 更新、移行、またはアンインストールが必要です
summary: OpenClaw のインストール - インストーラースクリプト、npm/pnpm/bun、ソース、Docker など
title: インストール
x-i18n:
    generated_at: "2026-07-26T10:17:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc6c6c33294852c90d2d2904b78ff8b0483b8e72a380d5835c5bdda67547de0c
    source_path: install/index.md
    workflow: 16
---

## システム要件

- **Node 22.22.3+、24.15+、または 25.9+** - Node 24 がデフォルトの対象です。インストーラースクリプトがこれを自動的に処理します。
- **macOS、Linux、または Windows** - Windows ユーザーは、ネイティブ Windows Hub アプリ、PowerShell CLI インストーラー、または WSL2 Gateway から始められます。[Windows](/ja-JP/platforms/windows) を参照してください。
- `pnpm` はソースからビルドする場合にのみ必要です。

## 推奨：インストーラースクリプト

最も速いインストール方法です。OS を検出し、必要に応じて Node をインストールして、OpenClaw をインストールし、オンボーディングを起動します。

<Note>
Windows デスクトップユーザーは、セットアップ、トレイステータス、チャット、Node モード、ローカル MCP モードを備えたネイティブのコンパニオンアプリ [Windows Hub](/ja-JP/platforms/windows#recommended-windows-hub) もインストールできます。
</Note>

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

オンボーディングを実行せずにインストールするには：

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

すべてのフラグと CI／自動化オプションについては、[インストーラーの内部構造](/ja-JP/install/installer)を参照してください。

## その他のインストール方法

### ローカルプレフィックスインストーラー（`install-cli.sh`）

システム全体の Node インストールに依存せず、OpenClaw と Node を
`~/.openclaw` などのローカルプレフィックス配下に保持する場合に使用します。

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

デフォルトで npm インストールをサポートし、同じプレフィックスフローで
git チェックアウトからのインストールもサポートします。詳細なリファレンス：[インストーラーの内部構造](/ja-JP/install/installer#install-clish)。

すでにインストール済みの場合は、`openclaw update --channel dev` と `openclaw update --channel stable` を使用して
パッケージインストールと git インストールを切り替えられます。
[更新](/ja-JP/install/updating#switch-between-npm-and-git-installs)を参照してください。

### npm、pnpm、または bun

Node を自分で管理している場合：

<Tabs>
  <Tab title="npm">
    ```bash
    npm install -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    ホスト型インストーラーは、OpenClaw パッケージのインストール時に `min-release-age`
    などの npm の鮮度フィルターを解除します。npm で手動インストールする場合は、独自の
    npm ポリシーが引き続き適用されます。
    </Note>

  </Tab>
  <Tab title="pnpm">
    ```bash
    pnpm add -g openclaw@latest
    pnpm approve-builds -g
    openclaw onboard --install-daemon
    ```

    <Note>
    pnpm では、ビルドスクリプトを含むパッケージに明示的な承認が必要です。初回インストール後に `pnpm approve-builds -g` を実行してください。
    </Note>

  </Tab>
  <Tab title="bun">
    ```bash
    bun add -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    Bun はグローバルパッケージをインストールできますが、OpenClaw の状態管理では `node:sqlite` を使用するため、生成される `openclaw` 実行可能ファイルには、サポート対象の Node ランタイムが必要です。
    </Note>

  </Tab>
</Tabs>

### ソースから

コントリビューター、またはローカルチェックアウトから実行したい場合：

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install && pnpm build && pnpm ui:build
pnpm link --global
openclaw onboard --install-daemon
```

またはリンクを省略し、リポジトリ内から `pnpm openclaw ...` を使用します。開発ワークフローの詳細については、[セットアップ](/ja-JP/start/setup)を参照してください。

### GitHub の main チェックアウトからインストール

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --version main
```

### コンテナとパッケージマネージャー

<CardGroup cols={2}>
  <Card title="Docker" href="/ja-JP/install/docker" icon="container">
    コンテナ化またはヘッドレスデプロイ。
  </Card>
  <Card title="Podman" href="/ja-JP/install/podman" icon="container">
    Docker に代わるルートレスコンテナ。
  </Card>
  <Card title="Nix" href="/ja-JP/install/nix" icon="snowflake">
    Nix flake による宣言的インストール。
  </Card>
  <Card title="Ansible" href="/ja-JP/install/ansible" icon="server">
    フリートの自動プロビジョニング。
  </Card>
  <Card title="Bun" href="/ja-JP/install/bun" icon="zap">
    オプションの依存関係インストーラーおよびパッケージスクリプトランナー。
  </Card>
</CardGroup>

## インストールの確認

```bash
openclaw --version      # CLI が利用可能であることを確認
openclaw doctor         # 設定の問題を確認
openclaw gateway status # Gateway が実行中であることを確認
```

インストール後に管理された自動起動を使用する場合：

- macOS：`openclaw onboard --install-daemon` または `openclaw gateway install` による LaunchAgent
- Linux／WSL2：同じコマンドによる systemd ユーザーサービス
- ネイティブ Windows：最初に Scheduled Task を使用し、タスクの作成が拒否された場合はユーザーごとの Startup フォルダーのログイン項目へフォールバック

## ホスティングとデプロイ

OpenClaw をクラウドサーバーまたは VPS にデプロイします。プロバイダー選択の全一覧
（DigitalOcean、Hetzner、Hostinger、Fly.io、GCP、Azure、Railway、
Northflank、Oracle Cloud、Raspberry Pi など）については [Linux サーバー](/ja-JP/vps)を参照するか、
[Render](/ja-JP/install/render) に宣言的にデプロイしてください。

<CardGroup cols={3}>
  <Card title="VPS" href="/ja-JP/vps">
    プロバイダーを選択します。
  </Card>
  <Card title="Docker VM" href="/ja-JP/install/docker-vm-runtime">
    共通の Docker 手順。
  </Card>
  <Card title="Kubernetes" href="/ja-JP/install/kubernetes">
    K8s デプロイ。
  </Card>
</CardGroup>

## 更新、移行、またはアンインストール

<CardGroup cols={3}>
  <Card title="更新" href="/ja-JP/install/updating" icon="refresh-cw">
    OpenClaw を最新の状態に保ちます。
  </Card>
  <Card title="移行" href="/ja-JP/install/migrating" icon="arrow-right">
    新しいマシンに移行します。
  </Card>
  <Card title="アンインストール" href="/ja-JP/install/uninstall" icon="trash-2">
    OpenClaw を完全に削除します。
  </Card>
</CardGroup>

## トラブルシューティング：`openclaw` が見つからない

ほとんどの場合は PATH の問題です。npm のグローバル bin ディレクトリがシェルの `PATH` に含まれていません。Windows のパスを含む完全な修正方法については、[Node.js のトラブルシューティング](/ja-JP/install/node#troubleshooting)を参照してください。

```bash
node -v           # Node はインストール済み？
npm prefix -g     # グローバルパッケージの場所は？
echo "$PATH"      # グローバル bin ディレクトリは PATH に含まれている？
```
