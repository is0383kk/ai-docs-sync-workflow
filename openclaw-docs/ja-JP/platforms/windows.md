---
read_when:
    - Windows への OpenClaw のインストール
    - Windows Hub、ネイティブ Windows、WSL2 の選択
    - Windows コンパニオンアプリまたは Windows Node モードのセットアップ
summary: Windows サポート：Windows Hub、ネイティブ CLI と Gateway、WSL2 Gateway セットアップ、Node モード、トラブルシューティング
title: Windows
x-i18n:
    generated_at: "2026-07-26T09:30:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw には、ネイティブの **Windows Hub** コンパニオンアプリと Windows CLI サポートが付属しています。
セットアップ、トレイのステータス、チャット、Command Center の診断、
Windows Node の機能を備えたデスクトップアプリには Windows Hub を使用します。
CLI/Gateway を直接使用するには PowerShell インストーラーを使用します。
Linux との互換性が最も高い Gateway ランタイムには WSL2 を使用します。

## 推奨: Windows Hub

Windows Hub は、Windows 10 20H2 以降および Windows 11 向けのネイティブ WinUI
コンパニオンアプリです。管理者権限なしでインストールでき、署名済みの x64
および ARM64 インストーラーが専用のリリースページで提供されます。

Windows Hub は、OpenClaw CLI および Gateway とは独立して公開されます。
最新の安定版 Hub インストーラーは、
[Windows Hub リリースページ](https://github.com/openclaw/openclaw-windows-node/releases/latest)
または `releases/latest/download` から直接ダウンロードしてください。

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

上記のリンクで 404 エラーが発生する場合は、[Windows Hub リリースページ](https://github.com/openclaw/openclaw-windows-node/releases)
にアクセスし、最新の安定版 Windows Hub リリースを開いてください。通常の OpenClaw
安定版リリースにも、リリース検証済みの固定された Windows Hub ビルドがミラーされますが、
そのミラーは新しい単独の Hub リリースより遅れる場合があります。

インストール後、スタートメニューまたはシステムトレイから **OpenClaw Companion**
を起動します。インストーラーは、Gateway セットアップ、チャット、設定、
更新の確認、アンインストール用のショートカットも追加します。

### Windows Hub に含まれる機能

- システムトレイのステータスとログイン時の起動。
- アプリが所有するローカル WSL Gateway の初回セットアップ。
- ローカル、リモート、SSH トンネル経由の Gateway の接続設定。
- ネイティブのチャットウィンドウと、ブラウザー版 Control UI へのアクセス。
- セッション、使用量、チャンネル、Node、ペアリング、
  修復コマンドに関する Command Center の診断。
- エージェントが制御するキャンバス、画面、カメラ、
  通知、デバイスの状態、音声会話、制御された `system.run` のための Windows Node モード。
- Claude Desktop、Claude Code、Cursor などの MCP クライアント向けの
  ローカル MCP サーバーモード。

### 初回起動

初回起動時、使用可能な保存済み Gateway がない場合、Windows Hub はセットアップを開きます。
最も速い方法は **ローカルでセットアップ** です。これにより、アプリが所有する
`OpenClawGateway` WSL ディストリビューションがプロビジョニングされ、その内部に Gateway がインストールされ、
アプリとペアリングされます。既存の Ubuntu ディストリビューションがエクスポートまたは変更されることはありません。

すでに Gateway がある場合は、**詳細セットアップ** を選択するか、接続タブを開きます。
次の対象に接続できます。

- この PC 上のローカル Gateway
- この PC 上の WSL Gateway
- URL とトークンまたはセットアップコードを使用するリモート Gateway
- SSH トンネル経由で到達する Gateway

セットアップが完了すると、トレイアイコンが緑色になります。トレイから **Command Center**
を開き、接続、ペアリング、Node の状態、チャンネルの正常性を確認します。

## Windows Node モード

Windows Hub は OpenClaw Node として登録できるため、エージェントは Gateway を介して、
宣言された Windows ネイティブ機能を使用できます。Node コマンドを実行するには、
Node によって宣言され、Gateway ポリシーによって許可されている必要があります。
完全な許可/拒否モデルについては、[Node](/ja-JP/nodes#command-policy)を参照してください。

一般的なコマンド:

| ファミリー | コマンド                                                                             |
| ------ | ------------------------------------------------------------------------------------ |
| キャンバス | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot` |
| 画面 | `screen.snapshot`; `screen.record` には明示的なオプトインが必要です                          |
| カメラ | `camera.list`; `camera.snap`, `camera.clip` には明示的なオプトインが必要です                  |
| システム | `system.notify`, `system.run`, `system.run.prepare`, `system.which`                  |
| デバイス | `location.get`, `device.info`, `device.status`                                       |
| 音声会話   | `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel`, `talk.ptt.once`, `talk.speak`  |

Node モードには Gateway とのペアリングが必要です。アプリにペアリング要求が表示された場合は、
Gateway ホストから承認します。

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Gateway は、Node が宣言し、サーバーポリシーが許可するコマンドのみを転送します。
`screen.record`、`camera.snap`、
`camera.clip` など、プライバシーに関わるコマンドには、明示的な `gateway.nodes.commands.allow` オプトインが必要です。

## ローカル MCP モード

Windows Hub は、同じ Windows ネイティブ機能レジストリを local loopback 上の
ローカル MCP サーバーとして公開できます。これにより、OpenClaw Gateway を実行せずに、
ローカル MCP クライアントから Windows の機能を操作できます。

Windows Hub の設定にある開発者向け/詳細セクションで有効にします。
サーバーを有効にすると、アプリに local loopback エンドポイントとベアラートークンが表示されます。

モード一覧:

| Node モード | MCP サーバー | 動作                           |
| --------- | ---------- | ---------------------------------- |
| オフ       | オフ        | オペレーター専用デスクトップアプリ          |
| オン        | オフ        | Gateway に接続された Windows Node     |
| オフ       | オン         | ローカル MCP サーバーのみ              |
| オン        | オン         | Gateway Node とローカル MCP サーバー |

## ネイティブ Windows CLI と Gateway

ターミナル中心で使用する場合は、PowerShell から OpenClaw をインストールします。

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

確認:

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

管理された起動では、利用可能な場合に Windows タスクスケジューラを使用します。
タスクは、読み取り可能な `gateway.cmd` スクリプトを OpenClaw の状態ディレクトリに保持しますが、
生成された `gateway.vbs` WScript ラッパーを介して起動するため、バックグラウンドの Gateway
が表示可能なコンソールウィンドウを開くことはありません。タスクの作成が拒否された場合、
OpenClaw はユーザー単位のスタートアップフォルダーのログイン項目にフォールバックします。

Gateway サービスをインストールします。

```powershell
openclaw gateway install
openclaw gateway status --json
```

管理された Gateway サービスを使用せず、CLI のみを使用する場合:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## WSL2 Gateway

WSL2 は引き続き、Windows 上で Linux との互換性が最も高い Gateway ランタイムです。
Windows Hub でアプリ所有の WSL Gateway をセットアップすることも、
独自のディストリビューション内に手動でインストールすることもできます。

手動セットアップ:

```powershell
wsl --install
# または、ディストリビューションを明示的に選択:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

WSL 内で systemd を有効にします。

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

PowerShell から WSL を再起動します。

```powershell
wsl --shutdown
```

次に、Linux クイックスタートを使用して WSL 内に OpenClaw をインストールします。

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## Windows ログイン前の Gateway 自動起動

ヘッドレス WSL セットアップでは、Windows に誰もログインしていない場合でも、
ブートチェーン全体が実行されるようにします。

WSL 内:

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

管理者として PowerShell を実行:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

`Ubuntu` を、次のコマンドで確認できるディストリビューション名に置き換えます。

```powershell
wsl --list --verbose
```

<Note>
以前の手順からの変更点は 2 つあります。

- **`/bin/true` ではなく `dbus-launch true`**: WSL >= 2.6.1.0 では、
  リグレッション（[microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416)）により、
  linger が有効でも、最後のクライアントが終了してから 15-20 秒後に
  ディストリビューションがアイドル終了します。`dbus-launch true` は、
  回避策として init の子プロセスを存続させます
  （コミュニティでの議論: [microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245)）。
- **`/ru SYSTEM` ではなく `/ru "$env:USERNAME"`**: ユーザー単位の WSL ディストリビューション
  （デフォルトのセットアップ）は SYSTEM アカウントから認識できないため、タスクは実行されたように見えても、
  ディストリビューションは起動しません。自分のアカウントで実行すればこの問題を回避できます。
  タスクの作成時に Windows がパスワードの入力を求めます。

</Note>

再起動後、WSL から確認します。

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## WSL サービスを LAN 経由で公開する

WSL には独自の仮想ネットワークがあります。別のマシンから WSL 内のサービスに
到達する必要がある場合は、Windows ポートを現在の WSL IP に転送します。
WSL IP は再起動後に変わる可能性があるため、必要に応じて転送ルールを更新してください。

管理者として PowerShell で実行する例:

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP が見つかりません。" }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

注意事項:

- 別のマシンから SSH 接続する場合は、Windows ホストの IP（例: `ssh user@windows-host -p 2222`）を接続先にします。
- リモート Node は、`127.0.0.1` ではなく、到達可能な Gateway URL を指定する必要があります。
- LAN アクセスには `listenaddress=0.0.0.0`、ローカルのみのアクセスには `127.0.0.1` を使用します。

## トラブルシューティング

### トレイアイコンが表示されない

タスクマネージャーで `OpenClaw.Tray.WinUI.exe` を確認します。実行中の場合は、
非表示のトレイアイコン領域を開いて固定します。実行されていない場合は、
スタートメニューから **OpenClaw Companion** を起動します。

### ローカルセットアップに失敗する

Windows Hub からセットアップログを開くか、次を確認します。

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

一般的な原因には、WSL が無効になっていること、仮想化がブロックされていること、
アプリ所有の WSL 状態が古いこと、Gateway パッケージのインストール中に
ネットワーク障害が発生したことがあります。

### アプリにペアリングが必要と表示される

Gateway からオペレーターまたは Node の要求を承認します。

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

デバイスにすでにトークンがある場合は、承認後に接続タブから再接続します。

### Web チャットからリモート Gateway に到達できない

リモート Web チャットには HTTPS または localhost が必要です。
自己署名証明書の場合は Windows で証明書を信頼するか、
SSH トンネルを使用して localhost の URL に接続します。

### `screen.snapshot`、カメラ、または音声コマンドが失敗する

カメラ、マイク、画面キャプチャ、通知に対する Windows の権限を確認します。
パッケージ版インストールでは保護された機能が宣言されていますが、
コマンドで初めて使用するときに Windows が確認を求める場合があります。

### Git または GitHub への接続に失敗する

ネットワークによっては、GitHub への HTTPS 接続がブロックまたは帯域制限されます。
`git clone` または `gh auth login` が失敗する場合は、
別のネットワーク、VPN、または HTTP/HTTPS プロキシを試してください。

現在のセッションでトークンベースの `gh` 認証を使用する場合:

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

トークンをコミットしたり、Issue やプルリクエストに貼り付けたりしないでください。

## 関連項目

- [インストールの概要](/ja-JP/install)
- [Node.js のセットアップ](/ja-JP/install/node)
- [Node](/ja-JP/nodes)
- [Control UI](/ja-JP/web/control-ui)
- [Gateway の設定](/ja-JP/gateway/configuration)
