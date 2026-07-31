---
read_when:
    - メインの macOS 環境から OpenClaw を分離したい場合
    - サンドボックスで iMessage 連携を利用したい場合
    - クローン可能でリセットできる macOS 環境が必要な場合
    - ローカルとホスト型の macOS VM オプションを比較したい場合
summary: 分離環境または iMessage が必要な場合は、サンドボックス化された macOS VM（ローカルまたはホスト環境）で OpenClaw を実行します
title: macOS 仮想マシン
x-i18n:
    generated_at: "2026-07-26T09:27:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e6b963faaf40f65adce1081715bc295059b8bed278a8c71a05a86e04ad7a7a5
    source_path: install/macos-vm.md
    workflow: 16
---

## 推奨されるデフォルト（ほとんどのユーザー向け）

- 常時稼働する Gateway を低コストで運用するには、**小規模な Linux VPS**。[VPS ホスティング](/ja-JP/vps)を参照してください。
- 完全な制御と、ブラウザ自動化用の**住宅用 IP**が必要な場合は、**専用ハードウェア**（Mac mini または Linux マシン）。多くのサイトはデータセンターの IP をブロックするため、ローカルでのブラウジングのほうが適切に動作することがよくあります。
- **ハイブリッド**：Gateway は安価な VPS 上で維持し、ブラウザ／UI 自動化が必要なときに Mac を **node** として接続します。[Node](/ja-JP/nodes)と[リモート Gateway](/ja-JP/gateway/remote)を参照してください。

iMessage など macOS 専用の機能が特に必要な場合や、普段使いの Mac から厳密に分離したい場合にのみ、macOS VM を使用してください。

## macOS VM の選択肢

### Apple Silicon Mac 上のローカル VM（Lume）

[Lume](https://cua.ai/docs/lume) を使用して、既存の Apple Silicon Mac 上にあるサンドボックス化された macOS VM 内で OpenClaw を実行します。これにより、次の利点が得られます。

- 分離された完全な macOS 環境（ホストをクリーンな状態に維持）
- `imsg` による iMessage サポート。デフォルトのローカルパスは Linux／Windows では使用できません
- VM のクローンによる即時リセット
- 追加のハードウェア費用やクラウド費用が不要

### ホスティング型 Mac プロバイダー（クラウド）

クラウド上で macOS を使用したい場合は、ホスティング型 Mac プロバイダーも利用できます。

- [MacStadium](https://www.macstadium.com/)（ホスティング型 Mac）
- その他のホスティング型 Mac ベンダーも利用できます。各ベンダーの VM と SSH に関するドキュメントに従ってください

macOS VM への SSH アクセスを確立したら、以下の「[OpenClaw をインストールする](#6-install-openclaw)」に進んでください。

## クイック手順（Lume、経験者向け）

1. Lume をインストールします。
2. `lume create openclaw --os macos --ipsw latest`
3. Setup Assistant を完了し、Remote Login（SSH）を有効にします。
4. `lume run openclaw --no-display`
5. SSH で接続し、OpenClaw をインストールしてチャンネルを設定します。
6. 完了です。

## 必要なもの（Lume）

- Apple Silicon Mac（M1／M2／M3／M4）
- ホスト上の macOS Sequoia 以降
- VM ごとに約 60 GB の空きディスク容量
- 約 20 分

## 1) Lume をインストールする

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

`~/.local/bin` が PATH にない場合：

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

確認します。

```bash
lume --version
```

ドキュメント：[Lume のインストール](https://cua.ai/docs/lume/guide/getting-started/installation)

## 2) macOS VM を作成する

```bash
lume create openclaw --os macos --ipsw latest
```

これにより macOS がダウンロードされ、VM が作成されます。VNC ウィンドウが自動的に開きます。

<Note>
接続環境によっては、ダウンロードに時間がかかる場合があります。
</Note>

## 3) Setup Assistant を完了する

VNC ウィンドウで次の操作を行います。

1. 言語と地域を選択します。
2. Apple ID をスキップします（後で iMessage を使用する場合はサインインします）。
3. ユーザーアカウントを作成します（ユーザー名とパスワードを記録しておいてください）。
4. すべてのオプション機能をスキップします。

セットアップの完了後：

1. SSH を有効にします：System Settings -> General -> Sharing を開き、"Remote Login" を有効にします。
2. VM をヘッドレスで使用するには、自動ログインを有効にします：System Settings -> Users & Groups を開き、"Automatically log in as:" を選択して、VM ユーザーを選びます。

## 4) VM の IP アドレスを取得する

```bash
lume get openclaw
```

IP アドレス（通常は `192.168.64.x`）を確認します。

## 5) VM に SSH で接続する

```bash
ssh youruser@192.168.64.X
```

`youruser` を作成したアカウントに置き換え、IP を VM の IP に置き換えます。

## 6) OpenClaw をインストールする

VM 内で次を実行します。

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

オンボーディングのプロンプトに従い、モデルプロバイダー（Anthropic、OpenAI など）を設定します。

## 7) チャンネルを設定する

設定ファイルを編集します。

```bash
nano ~/.openclaw/openclaw.json
```

チャンネルを追加します。

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

次に WhatsApp にログインします（QR をスキャン）。

```bash
openclaw channels login
```

## 8) VM をヘッドレスで実行する

VM を停止し、ディスプレイなしで再起動します。

```bash
lume stop openclaw
lume run openclaw --no-display
```

VM はバックグラウンドで実行され、OpenClaw のデーモンが Gateway の稼働を維持します。ステータスを確認するには、次を実行します。

```bash
ssh youruser@192.168.64.X "openclaw status"
```

## おまけ：iMessage 連携

これは macOS 上で実行する最大の利点です。[iMessage](/ja-JP/channels/imessage) と `imsg` を使用して、Messages を OpenClaw に追加します。

VM 内で次の操作を行います。

1. Messages にサインインします。
2. `imsg` をインストールします。
3. OpenClaw／`imsg` を実行するプロセスに、Full Disk Access と Automation の権限を付与します。
4. `imsg rpc --help` で RPC サポートを確認します。

OpenClaw の設定に追加します。

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
    },
  },
}
```

Gateway を再起動します。これでエージェントが iMessage を送受信できるようになります。セットアップの詳細：[iMessage チャンネル](/ja-JP/channels/imessage)。

## ゴールデンイメージを保存する

さらにカスタマイズする前に、クリーンな状態のスナップショットを作成します。

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

いつでもリセットできます。

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

## 24/7 の実行

次の方法で VM の稼働を維持します。

- Mac を電源に接続したままにする
- System Settings -> Energy Saver でスリープを無効にする
- 必要に応じて `caffeinate` を使用する

常時稼働が必要な場合は、専用の Mac mini または小規模な VPS を検討してください。[VPS ホスティング](/ja-JP/vps)を参照してください。

## トラブルシューティング

| 問題                     | 解決方法                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------- |
| VM に SSH 接続できない   | VM の System Settings で "Remote Login" が有効になっていることを確認します          |
| VM の IP が表示されない  | VM が完全に起動するまで待ち、`lume get openclaw` を再度実行します                    |
| Lume コマンドが見つからない | `~/.local/bin` を PATH に追加します                                           |
| WhatsApp の QR をスキャンできない | `openclaw channels login` の実行時に、ホストではなく VM にログインしていることを確認します |

## 関連ドキュメント

- [VPS ホスティング](/ja-JP/vps)
- [Node](/ja-JP/nodes)
- [リモート Gateway](/ja-JP/gateway/remote)
- [iMessage チャンネル](/ja-JP/channels/imessage)
- [Lume クイックスタート](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [Lume CLI リファレンス](https://cua.ai/docs/lume/reference/cli-reference)
- [無人 VM セットアップ](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup)（上級者向け）
- [Docker サンドボックス](/ja-JP/install/docker)（代替の分離方法）
