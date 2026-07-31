---
read_when:
    - 尋找作業系統支援或安裝路徑
    - 決定閘道的執行位置
summary: 平台支援概覽（閘道 + 配套應用程式）
title: 平台
x-i18n:
    generated_at: "2026-07-26T08:38:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40494f8567c0159d9b6024c174cf0f316a45b46c633a578efaf2388f679a88f2
    source_path: platforms/index.md
    workflow: 16
---

OpenClaw 核心以 TypeScript 撰寫。**節點是必要的執行環境**，因為
標準狀態儲存區使用 `node:sqlite`。Bun 仍可用於
安裝相依套件及執行套件指令碼；請參閱 [Bun](/zh-TW/install/bun)。

Windows Hub、macOS（選單列 App）與行動節點
（iOS/Android）均有配套 App。Linux 配套 App 尚在規劃中，但閘道目前已
獲得完整支援。在 Windows 上，若要使用桌面 App，請選擇 Windows Hub；
若以終端機操作為主，請選擇原生 PowerShell 安裝；若需要與 Linux
最相容的閘道執行環境，請選擇 WSL2。

## 選擇你的作業系統

- macOS：[macOS](/zh-TW/platforms/macos)
- iOS：[iOS](/zh-TW/platforms/ios)
- Android：[Android](/zh-TW/platforms/android)
- Windows：[Windows](/zh-TW/platforms/windows)
- Linux：[Linux](/zh-TW/platforms/linux)

## VPS 與託管

- VPS 中樞：[VPS 託管](/zh-TW/vps)
- Fly.io：[Fly.io](/zh-TW/install/fly)
- Hetzner（Docker）：[Hetzner](/zh-TW/install/hetzner)
- GCP（Compute Engine）：[GCP](/zh-TW/install/gcp)
- Azure（Linux VM）：[Azure](/zh-TW/install/azure)
- exe.dev（VM + HTTPS Proxy）：[exe.dev](/zh-TW/install/exe-dev)
- EasyRunner（Podman + Caddy）：[EasyRunner](/zh-TW/platforms/easyrunner)

## 常用連結

- 安裝指南：[開始使用](/zh-TW/start/getting-started)
- Windows Hub：[Windows](/zh-TW/platforms/windows)
- 閘道操作手冊：[閘道](/zh-TW/gateway)
- 閘道設定：[設定](/zh-TW/gateway/configuration)
- 服務狀態：`openclaw gateway status`

## 安裝閘道服務（命令列介面）

請使用以下任一方式（均受支援）：

- 精靈（建議）：`openclaw onboard --install-daemon`
- 直接安裝：`openclaw gateway install`
- 設定流程：`openclaw configure` → 選取 **閘道服務**
- 修復／遷移：`openclaw doctor`（可選擇安裝或修復服務）

服務目標取決於作業系統：

- macOS：LaunchAgent（`ai.openclaw.gateway`，或針對具名設定檔使用 `ai.openclaw.<profile>`）
- Linux/WSL2：systemd 使用者服務（`openclaw-gateway[-<profile>].service`）
- 原生 Windows：排定的工作（`OpenClaw Gateway` 或 `OpenClaw Gateway (<profile>)`）；若建立工作遭拒，則改用每位使用者的「啟動」資料夾登入項目

## 相關內容

- [安裝概覽](/zh-TW/install)
- [Windows Hub](/zh-TW/platforms/windows)
- [macOS App](/zh-TW/platforms/macos)
- [iOS App](/zh-TW/platforms/ios)
