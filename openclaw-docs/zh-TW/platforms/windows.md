---
read_when:
    - 在 Windows 上安裝 OpenClaw
    - 在 Windows Hub、原生 Windows 與 WSL2 之間做選擇
    - 設定 Windows 配套應用程式或 Windows 節點模式
summary: Windows 支援：Windows Hub、原生命令列介面與閘道、WSL2 閘道設定、節點模式及疑難排解
title: Windows
x-i18n:
    generated_at: "2026-07-26T08:01:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw 隨附原生的 **Windows Hub** 夥伴應用程式，並支援 Windows 命令列介面。
若需要具備設定、系統匣狀態、聊天、Command Center 診斷及 Windows 節點功能的桌面應用程式，請使用 Windows Hub。若要直接使用命令列介面／閘道，請使用 PowerShell
安裝程式。若要獲得與 Linux 相容性最高的閘道執行環境，請使用 WSL2。

## 建議：Windows Hub

Windows Hub 是適用於 Windows 10 20H2+ 和
Windows 11 的原生 WinUI 夥伴應用程式。安裝不需要系統管理員權限，並透過專屬發行頁面提供已簽署的 x64
和 ARM64 安裝程式。

Windows Hub 與 OpenClaw 命令列介面及閘道分開發布。請從
[Windows Hub 發行頁面](https://github.com/openclaw/openclaw-windows-node/releases/latest)
下載最新穩定版 Hub 安裝程式，或直接透過 `releases/latest/download` 下載：

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

如果上述連結傳回 404，請前往 [Windows Hub 發行頁面](https://github.com/openclaw/openclaw-windows-node/releases)
並開啟最新的 Windows Hub 穩定版本。OpenClaw 的一般穩定版本
也會鏡像經固定及發行驗證的 Windows Hub 組建；該鏡像可能會落後於
較新的獨立 Hub 版本。

安裝後，請從開始功能表或系統匣啟動 **OpenClaw Companion**。
安裝程式也會新增 Gateway Setup、Chat、Settings、
Check for Updates 及解除安裝的捷徑。

### Windows Hub 包含的功能

- 系統匣狀態及登入時啟動。
- 首次執行時設定由本機應用程式擁有的 WSL 閘道。
- 本機、遠端及透過 SSH 通道連線之閘道的連線設定。
- 原生聊天視窗，以及瀏覽器版控制介面的存取權。
- Command Center 診斷，涵蓋工作階段、用量、頻道、節點、配對
  及修復命令。
- Windows 節點模式，提供由代理程式控制的畫布、螢幕、相機、
  通知、裝置狀態、語音及受控的 `system.run`。
- 適用於 Claude Desktop、Claude Code
  及 Cursor 等 MCP 用戶端的本機 MCP 伺服器模式。

### 首次啟動

首次啟動時，如果沒有可用的已儲存
閘道，Windows Hub 會開啟設定。最快的方式是 **Set up locally**，此選項會佈建
由應用程式擁有的 `OpenClawGateway` WSL 發行版、在其中安裝閘道，並
配對應用程式。此操作不會匯出或修改你現有的 Ubuntu 發行版。

如果你已有閘道，請選擇 **Advanced setup** 或開啟 Connections 分頁。
你可以連線至：

- 此電腦上的本機閘道
- 此電腦上的 WSL 閘道
- 透過 URL 和權杖或設定代碼連線的遠端閘道
- 透過 SSH 通道連線的閘道

設定完成後，系統匣圖示會變為綠色。請從
系統匣開啟 **Command Center**，確認連線、配對、節點狀態及頻道健康狀態。

## Windows 節點模式

Windows Hub 可以註冊為 OpenClaw 節點，讓代理程式能透過閘道使用已宣告的
Windows 原生功能。節點命令必須由節點
宣告並獲閘道原則允許後才能執行；完整的允許／拒絕模型請參閱
[節點](/zh-TW/nodes#command-policy)。

常用命令：

| 類別 | 命令                                                                             |
| ------ | ------------------------------------------------------------------------------------ |
| 畫布 | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot` |
| 螢幕 | `screen.snapshot`；`screen.record` 需要明確選擇啟用                          |
| 相機 | `camera.list`；`camera.snap`、`camera.clip` 需要明確選擇啟用                  |
| 系統 | `system.notify`, `system.run`, `system.run.prepare`, `system.which`                  |
| 裝置 | `location.get`, `device.info`, `device.status`                                       |
| 語音   | `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel`, `talk.ptt.once`, `talk.speak`  |

節點模式需要與閘道配對。如果應用程式顯示配對要求，
請從閘道主機核准：

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

閘道只會轉送節點所宣告且伺服器原則
允許的命令。`screen.record`、`camera.snap`
和 `camera.clip` 等涉及隱私的命令，需要明確選擇啟用 `gateway.nodes.commands.allow`。

## 本機 MCP 模式

Windows Hub 可以在回送介面上，將相同的 Windows 原生功能登錄公開為本機
MCP 伺服器，讓本機 MCP 用戶端能在沒有執行中 OpenClaw 閘道的情況下
驅動 Windows 功能。

請在 Windows Hub Settings 的開發人員／進階區段中啟用。伺服器啟用後，
應用程式會顯示回送端點及持有人權杖。

模式矩陣：

| 節點模式 | MCP 伺服器 | 行為                           |
| --------- | ---------- | ---------------------------------- |
| 關閉       | 關閉        | 僅供操作員使用的桌面應用程式          |
| 開啟        | 關閉        | 連線至閘道的 Windows 節點     |
| 關閉       | 開啟         | 僅本機 MCP 伺服器              |
| 開啟        | 開啟         | 閘道節點加上本機 MCP 伺服器 |

## 原生 Windows 命令列介面和閘道

若主要透過終端機使用，請從 PowerShell 安裝 OpenClaw：

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

驗證：

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

可用時，受管理的啟動機制會使用 Windows 排定的工作。該工作會將
可讀的 `gateway.cmd` 指令碼保留在 OpenClaw 狀態目錄中，但會透過
產生的 `gateway.vbs` WScript 包裝函式啟動，因此背景閘道
不會開啟可見的主控台視窗。如果建立工作遭拒，OpenClaw
會改用每位使用者 Startup 資料夾中的登入項目。

安裝閘道服務：

```powershell
openclaw gateway install
openclaw gateway status --json
```

若只使用命令列介面而不使用受管理的閘道服務：

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## WSL2 閘道

WSL2 仍是 Windows 上與 Linux 相容性最高的閘道執行環境。Windows
Hub 可以為你設定由應用程式擁有的 WSL 閘道，也可以在
你自己的發行版中手動安裝。

手動設定：

```powershell
wsl --install
# 或明確選擇發行版：
wsl --list --online
wsl --install -d Ubuntu-24.04
```

在 WSL 內啟用 systemd：

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

從 PowerShell 重新啟動 WSL：

```powershell
wsl --shutdown
```

接著使用 Linux 快速入門在 WSL 內安裝 OpenClaw：

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## 在登入 Windows 前自動啟動閘道

對於無頭式 WSL 設定，請確保即使沒有人
登入 Windows，完整的開機鏈仍會執行。

在 WSL 內：

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

在以系統管理員身分執行的 PowerShell 中：

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

將 `Ubuntu` 替換為以下命令列出的發行版名稱：

```powershell
wsl --list --verbose
```

<Note>
與舊版操作方式相比有兩項變更：

- **使用 `dbus-launch true` 而非 `/bin/true`**：在 WSL >= 2.6.1.0 上，有一項
  迴歸問題（[microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416)）
  會在最後一個用戶端結束 15-20 秒後，因閒置而終止發行版，即使已
  啟用 linger 也是如此。`dbus-launch true` 會讓 init 的子處理程序保持執行，
  作為因應措施（社群討論：[microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245)）。
- **使用 `/ru "$env:USERNAME"` 而非 `/ru SYSTEM`**：每位使用者的 WSL 發行版（
  預設設定）對 SYSTEM 帳戶不可見，因此工作看似
  正在執行，但發行版從未啟動。改用你自己的帳戶執行即可避免
  此問題；建立工作時，Windows 會提示輸入你的密碼。

</Note>

重新開機後，從 WSL 驗證：

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## 透過區域網路公開 WSL 服務

WSL 擁有自己的虛擬網路。如果另一台電腦必須存取
WSL 內的服務，請將 Windows 連接埠轉送至目前的 WSL IP。WSL IP 可能會在
重新啟動後變更，因此請在需要時重新整理轉送規則。

在以系統管理員身分執行的 PowerShell 中範例：

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "找不到 WSL IP。" }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

注意事項：

- 從另一台電腦透過 SSH 連線時，目標是 Windows 主機 IP，例如 `ssh user@windows-host -p 2222`。
- 遠端節點必須指向可連線的閘道 URL，而非 `127.0.0.1`。
- 區域網路存取請使用 `listenaddress=0.0.0.0`，僅限本機存取請使用 `127.0.0.1`。

## 疑難排解

### 系統匣圖示未出現

請在工作管理員中檢查 `OpenClaw.Tray.WinUI.exe`。如果它正在執行，請開啟
隱藏的系統匣圖示區域並將其釘選。如果沒有執行，請從
開始功能表啟動 **OpenClaw Companion**。

### 本機設定失敗

從 Windows Hub 開啟設定記錄，或檢查：

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

常見原因包括：WSL 已停用、虛擬化遭封鎖、由應用程式擁有的 WSL
狀態過時，或安裝閘道套件時發生網路錯誤。

### 應用程式顯示需要配對

從閘道核准操作員或節點要求：

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

如果裝置已有權杖，核准後請從 Connections 分頁重新連線。

### 網頁聊天無法連線至遠端閘道

遠端網頁聊天需要 HTTPS 或 localhost。若使用自我簽署的憑證，請在
Windows 中信任該憑證，或使用 SSH 通道連線至 localhost URL。

### `screen.snapshot`、相機或音訊命令失敗

請確認 Windows 對相機、麥克風、螢幕擷取及
通知的權限。封裝版安裝已宣告受保護的功能，但
Windows 在命令首次使用這些功能時仍可能提示。

### Git 或 GitHub 連線失敗

某些網路會封鎖或限制連至 GitHub 的 HTTPS。如果 `git clone` 或
`gh auth login` 失敗，請嘗試其他網路、VPN 或 HTTP/HTTPS Proxy。

若要在目前的工作階段中使用基於權杖的 `gh` 驗證：

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

絕對不要提交權杖，或將權杖貼到議題或 PR 中。

## 相關內容

- [安裝概覽](/zh-TW/install)
- [Node.js 設定](/zh-TW/install/node)
- [節點](/zh-TW/nodes)
- [控制介面](/zh-TW/web/control-ui)
- [閘道設定](/zh-TW/gateway/configuration)
