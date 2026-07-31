---
read_when:
    - OpenClawをインストールする前にNode.jsをインストールする必要があります
    - OpenClaw をインストールしたものの、`openclaw` コマンドが見つかりません
    - npm install -g が権限または PATH の問題で失敗する
summary: OpenClaw 用の Node.js のインストールと設定 - バージョン要件、インストールオプション、PATH のトラブルシューティング
title: Node.js
x-i18n:
    generated_at: "2026-07-26T09:07:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef4df255c24a11a549c757b597a07b00852e60973a5e513bdcf60796037a462a
    source_path: install/node.md
    workflow: 16
---

OpenClaw には **Node 22.22.3 以降、Node 24.15 以降、または Node 25.9 以降**が必要です。インストール、CI、リリースワークフローでは、**Node 24 がデフォルトかつ推奨のランタイム**です。Node 22 もアクティブな LTS 系列を通じて引き続きサポートされます。Node 23 はサポートされません。[インストーラースクリプト](/ja-JP/install#alternative-install-methods)は Node を自動的に検出してインストールします。Node を自分でセットアップする場合（バージョン、PATH、グローバルインストール）は、このページを参照してください。

## バージョンを確認する

```bash
node -v
```

`v24.15.0` 以降の 24.x が推奨されるデフォルトです。`v22.22.3` 以降の 22.x は、サポートされる Node 22 LTS の選択肢です。Node `v25.9.0+` もサポートされます。Node 23 はサポートされません。Node がインストールされていない場合や、サポート対象範囲外の場合は、以下からインストール方法を選択してください。

## Node をインストールする

<Tabs>
  <Tab title="macOS">
    **Homebrew**（推奨）:

    ```bash
    brew install node
    ```

    または、[nodejs.org](https://nodejs.org/) から macOS 用インストーラーをダウンロードします。

  </Tab>
  <Tab title="Linux">
    **Ubuntu / Debian:**

    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt-get install -y nodejs
    ```

    **Fedora / RHEL:**

    ```bash
    sudo dnf install nodejs
    ```

    または、バージョンマネージャーを使用します（後述）。

  </Tab>
  <Tab title="Windows">
    **winget**（推奨）:

    ```powershell
    winget install OpenJS.NodeJS.LTS
    ```

    **Chocolatey:**

    ```powershell
    choco install nodejs-lts
    ```

    または、[nodejs.org](https://nodejs.org/) から Windows 用インストーラーをダウンロードします。

  </Tab>
</Tabs>

<Accordion title="バージョンマネージャーを使用する（nvm、fnm、mise、asdf）">
  バージョンマネージャーを使用すると、Node のバージョンを簡単に切り替えられます。一般的な選択肢は次のとおりです。

- [**fnm**](https://github.com/Schniz/fnm) - 高速でクロスプラットフォーム
- [**nvm**](https://github.com/nvm-sh/nvm) - macOS/Linux で広く使用されている
- [**mise**](https://mise.jdx.dev/) - 多言語対応（Node、Python、Ruby など）

fnm の例:

```bash
fnm install 24
fnm use 24
```

  <Warning>
  シェルの起動ファイル（`~/.zshrc` または `~/.bashrc`）でバージョンマネージャーを初期化してください。これを行わないと、PATH に Node の bin ディレクトリが含まれないため、新しいターミナルセッションで `openclaw` が見つからない場合があります。
  </Warning>
</Accordion>

## トラブルシューティング

### `openclaw: command not found`

ほとんどの場合、npm のグローバル bin ディレクトリが PATH に含まれていないことが原因です。

<Steps>
  <Step title="npm のグローバルプレフィックスを確認する">
    ```bash
    npm prefix -g
    ```
  </Step>
  <Step title="PATH に含まれているか確認する">
    ```bash
    echo "$PATH"
    ```

    出力に `<npm-prefix>/bin`（macOS/Linux）または `<npm-prefix>`（Windows）が含まれているか確認します。

  </Step>
  <Step title="シェルの起動ファイルに追加する">
    <Tabs>
      <Tab title="macOS / Linux">
        `~/.zshrc` または `~/.bashrc` に以下を追加します。

        ```bash
        export PATH="$(npm prefix -g)/bin:$PATH"
        ```

        その後、新しいターミナルを開きます（または zsh では `rehash`、bash では `hash -r` を実行します）。
      </Tab>
      <Tab title="Windows">
        Settings → System → Environment Variables から、`npm prefix -g` の出力をシステムの PATH に追加します。
      </Tab>
    </Tabs>

  </Step>
</Steps>

### `npm install -g` での権限エラー（Linux）

`EACCES` エラーが表示される場合は、npm のグローバルプレフィックスをユーザーが書き込み可能なディレクトリに変更します。

```bash
mkdir -p "$HOME/.npm-global"
npm config set prefix "$HOME/.npm-global"
export PATH="$HOME/.npm-global/bin:$PATH"
```

永続的に適用するには、`export PATH=...` の行を `~/.bashrc` または `~/.zshrc` に追加します。

## 関連項目

- [インストールの概要](/ja-JP/install) - すべてのインストール方法
- [アップデート](/ja-JP/install/updating) - OpenClaw を最新の状態に保つ方法
- [はじめに](/ja-JP/start/getting-started) - インストール後の最初のステップ
