---
read_when:
    - 全新安裝、初始設定卡住或首次執行錯誤
    - 選擇驗證方式與供應商訂閱方案
    - 無法存取 docs.openclaw.ai、無法開啟儀表板、安裝卡住
sidebarTitle: First-run FAQ
summary: 常見問題：快速入門與首次執行設定 — 安裝、初始設定、驗證、訂閱、初始失敗狀況
title: 常見問題：首次執行設定
x-i18n:
    generated_at: "2026-07-26T08:19:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

快速入門與首次執行問答。如需日常操作、模型、驗證、工作階段
與疑難排解資訊，請參閱主要的[常見問題](/zh-TW/help/faq)。

## 快速入門與首次執行設定

<AccordionGroup>
  <Accordion title="我卡住了，最快的排除方式是什麼？">
    使用能夠**查看你的機器**的本機 AI 代理程式。大多數「我卡住了」的情況
    都是遠端協助者無法檢查的**本機設定或環境問題**，因此這比
    在 Discord 詢問更有效。

    - **Claude Code**：[https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**：[https://openai.com/codex/](https://openai.com/codex/)

    透過可修改的 (git) 安裝方式，讓代理程式取得完整的原始碼簽出內容，以便讀取
    程式碼與文件，並針對你實際執行的確切版本進行推理：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    要求代理程式逐步規劃並監督修正過程，然後只執行
    必要的命令；差異越小越容易稽核。

    尋求協助時（在 Discord 或 GitHub issue 中），請分享以下輸出：

    | 命令 | 顯示內容 |
    | --- | --- |
    | `openclaw status` | 閘道／代理程式健康狀態與基本設定快照 |
    | `openclaw status --all` | 可貼上的完整唯讀診斷 |
    | `openclaw models status` | 提供者驗證與模型可用性 |
    | `openclaw doctor` | 驗證並修復常見的設定／狀態問題 |
    | `openclaw logs --follow` | 即時日誌追蹤 |
    | `openclaw gateway status --deep` | 深度閘道／設定／外掛健康檢查 |
    | `openclaw health --verbose` | 詳細健康狀態報告 |

    發現真正的錯誤或修正方式了嗎？請建立 issue 或提交 PR：
    [Issues](https://github.com/openclaw/openclaw/issues) /
    [PR](https://github.com/openclaw/openclaw/pulls)。

    快速偵錯流程：[發生故障時的前 60 秒](/zh-TW/help/faq#first-60-seconds-if-something-is-broken)。
    安裝文件：[安裝](/zh-TW/install)、[安裝程式旗標](/zh-TW/install/installer)、[更新](/zh-TW/install/updating)。

  </Accordion>

  <Accordion title="心跳偵測一直略過。略過原因代表什麼？">
    | 略過原因 | 意義 |
    | --- | --- |
    | `quiet-hours` | 不在設定的活躍時段範圍內 |
    | `empty-heartbeat-file` | 心跳偵測監控草稿存在，但只包含空白、註解、標題、圍欄或空白核取清單的骨架內容 |
    | `alerts-disabled` | 所有心跳偵測可見性都已關閉（`showOk`、`showAlerts` 和 `useIndicator` 均已停用） |

    較舊的心跳偵測 `tasks:` 區塊會透過 `openclaw doctor --fix` 移轉為獨立排程的排程工作。

    文件：[心跳偵測](/zh-TW/gateway/heartbeat)、[自動化](/zh-TW/automation)。

  </Accordion>

  <Accordion title="安裝與設定 OpenClaw 的建議方式">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    從原始碼安裝（貢獻者／開發人員）：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    尚未全域安裝嗎？請改為執行 `pnpm openclaw onboard`。如果缺少 Control UI 資產，
    初始設定會嘗試自行建置，若失敗則改用 `pnpm ui:build`。

  </Accordion>

  <Accordion title="完成初始設定後，如何開啟儀表板？">
    初始設定完成後會立即在瀏覽器中開啟乾淨（不含權杖）的儀表板 URL，
    並在摘要中顯示該連結。請保持該分頁開啟；如果瀏覽器沒有啟動，
    請在同一台機器上複製並貼上顯示的 URL。
  </Accordion>

  <Accordion title="如何在 localhost 與遠端環境中驗證儀表板？">
    **Localhost（同一台機器）：**

    - 開啟 `http://127.0.0.1:18789/`。
    - 如果系統要求共用密鑰驗證，請將設定的權杖或密碼貼入 Control UI 設定。
    - 權杖來源：`gateway.auth.token`（或 `OPENCLAW_GATEWAY_TOKEN`）。
    - 密碼來源：`gateway.auth.password`（或 `OPENCLAW_GATEWAY_PASSWORD`）。
    - 尚未設定共用密鑰嗎？請執行 `openclaw doctor --generate-gateway-token`（或 `openclaw doctor --fix --generate-gateway-token`）。

    **非 localhost：**

    - **Tailscale Serve**（建議）：維持繫結至迴路介面，執行 `openclaw gateway --tailscale serve`，然後開啟 `https://<magicdns>/`。使用 `gateway.auth.allowTailscale: true` 時，身分識別標頭可滿足 Control UI／WebSocket 驗證要求（不必貼上共用密鑰，前提是信任閘道主機）；HTTP API 仍需要共用密鑰驗證，除非你刻意使用私人入口的 `none` 或受信任 Proxy HTTP 驗證。
      同一用戶端並行發出的錯誤驗證 Serve 嘗試，會在驗證失敗限制器記錄前依序處理，因此第二次錯誤重試可能已經顯示 `retry later`。
    - **Tailnet 繫結**：執行 `openclaw gateway --bind tailnet --token "<token>"`（或設定密碼驗證），開啟 `http://<tailscale-ip>:18789/`，並將相符的共用密鑰貼入儀表板設定。
    - **具身分識別功能的反向 Proxy**：讓閘道保持在受信任 Proxy 後方，設定 `gateway.auth.mode: "trusted-proxy"`，然後開啟 Proxy URL。同主機的迴路 Proxy 需要明確設定 `gateway.auth.trustedProxy.allowLoopback: true`。
    - **SSH 通道**：執行 `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`，然後開啟 `http://127.0.0.1:18789/`。透過通道時仍須使用共用密鑰驗證；如果系統提示，請貼上設定的權杖或密碼。

    如需繫結模式與驗證詳細資訊，請參閱[儀表板](/zh-TW/web/dashboard)和 [Web 介面](/zh-TW/web)。

  </Accordion>

  <Accordion title="為什麼聊天核准有兩種 exec 核准設定？">
    它們控制不同的層級：

    - `approvals.exec`－將核准提示轉送至聊天目的地。
    - `channels.<channel>.execApprovals`－讓該頻道成為 exec 核准的原生核准用戶端。

    主機 exec 原則仍是真正的核准閘門；聊天設定只控制提示出現的位置，
    以及使用者如何回覆提示。

    通常不需要同時使用兩者：

    - 如果聊天已支援命令與回覆，同一聊天中的 `/approve` 可透過共用路徑運作。
    - 當支援的原生頻道能安全推斷核准者，且 `channels.<channel>.execApprovals.enabled` 未設定或為 `"auto"` 時，OpenClaw 會自動啟用以私訊優先的原生核准。
    - 如果有原生核准卡片／按鈕，應優先使用該 UI；只有在工具結果表示聊天核准不可用時，才提及手動 `/approve` 命令。
    - 只有在提示也必須送達其他聊天或明確指定的維運聊天室時，才使用 `approvals.exec`。
    - 只有在你希望將核准提示回傳至原始聊天室／主題時，才使用 `channels.<channel>.execApprovals.target: "channel"` 或 `"both"`。
    - 外掛核准是獨立的：預設使用同一聊天中的 `/approve`，可選擇透過 `approvals.plugin` 轉送，而且只有部分原生頻道也會繼續以原生方式處理這些核准。

    簡單來說：轉送用於路由，原生用戶端設定則用於提供更豐富的頻道專屬使用者體驗。
    請參閱 [Exec 核准](/zh-TW/tools/exec-approvals)。

  </Accordion>

  <Accordion title="需要什麼執行環境？">
    必須使用 Node **22.22.3+**、**24.15+** 或 **25.9+**（建議使用 Node 24）。`pnpm` 是此儲存庫的套件管理員。
    Bun 可以安裝相依套件並執行套件指令碼，但無法執行 OpenClaw 命令列介面或閘道，因為它缺少 `node:sqlite`。
  </Accordion>

  <Accordion title="可以在 Raspberry Pi 上執行嗎？">
    可以，但請先檢查 RAM：Pi 5 和 Pi 4（2 GB 以上）最為合適；Pi 3B+（1 GB）可以運作但速度較慢；不建議使用 Pi Zero 2 W（512 MB）。

    | 型號 | RAM | 適用程度 |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | 最佳 |
    | Pi 4 | 4 GB | 良好 |
    | Pi 4 | 2 GB | 尚可，請增加交換空間 |
    | Pi 4 | 1 GB | 吃緊 |
    | Pi 3B+ | 1 GB | 緩慢 |
    | Pi Zero 2 W | 512 MB | 不建議 |

    絕對最低需求：1 GB RAM、1 個核心、500 MB 可用磁碟空間、64 位元作業系統。由於 Pi 只執行
    閘道（模型會呼叫雲端 API），即使規格普通的 Pi 也能處理此負載。

    小型 Pi／VPS 也可以只代管閘道，同時將筆記型電腦／手機上的**節點**
    配對，以使用本機螢幕／相機／畫布或執行命令。請參閱[節點](/zh-TW/nodes)。

    完整設定逐步指南：[Raspberry Pi](/zh-TW/install/raspberry-pi)。

  </Accordion>

  <Accordion title="安裝在 Raspberry Pi 上有什麼建議？">
    - 使用 **64 位元**作業系統；不要使用 32 位元 Raspberry Pi OS。
    - 在 2 GB 或更小容量的主機板上增加交換空間。
    - 為了效能與使用壽命，優先使用 **USB SSD**，而非 SD 卡。
    - 優先使用可修改的 (git) 安裝方式，以便查看日誌並快速更新。
    - 一開始不要啟用頻道／Skills，之後再逐一新增。
    - 奇怪的二進位檔失敗（「exec format error」）通常是因為選用的 Skill 工具缺少 ARM64 組建版本。

    完整指南：[Raspberry Pi](/zh-TW/install/raspberry-pi)。另請參閱 [Linux](/zh-TW/platforms/linux)。

  </Accordion>

  <Accordion title="畫面卡在 wake up my friend／初始設定無法孵化。該怎麼辦？">
    該畫面需要閘道可連線且已通過驗證。設定模型提供者後，終端介面也會在首次孵化時
    自動傳送「Wake up, my friend!」。如果你略過模型／驗證設定，初始設定會顯示
    「Model auth missing」提示，並直接開啟終端介面而不傳送任何內容；請使用
    `openclaw configure --section model` 新增提供者。
    如果你看到喚醒訊息但**沒有回覆**，而且權杖數量維持在 0，代表代理程式從未執行。

    1. 重新啟動閘道：

    ```bash
    openclaw gateway restart
    ```

    2. 檢查狀態與驗證：

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. 仍然卡住嗎？請執行：

    ```bash
    openclaw doctor
    ```

    如果閘道位於遠端，請確認通道／Tailscale 連線正常，而且 UI 指向正確的閘道。
    請參閱[遠端存取](/zh-TW/gateway/remote)。

  </Accordion>

  <Accordion title="可以將設定移轉到新機器，而不必重新執行初始設定嗎？">
    可以。複製**狀態目錄**和**工作區**，然後執行一次 Doctor：

    1. 在新機器上安裝 OpenClaw。
    2. 從舊機器複製 `$OPENCLAW_STATE_DIR`（預設值：`~/.openclaw`）。
    3. 複製你的工作區（預設值：`~/.openclaw/workspace`）。
    4. 執行 `openclaw doctor`，然後重新啟動閘道服務。

    這會保留設定、驗證設定檔、WhatsApp 認證資訊、工作階段與記憶；只要複製**這兩個**
    位置，你的機器人就會維持完全相同。在遠端模式中，閘道主機擁有工作階段儲存區與工作區。

    **重要：**如果只將工作區提交／推送到 GitHub，你備份的是
    **記憶與啟動載入檔案**，但不包含工作階段歷程記錄或驗證資料。這些內容位於
    `~/.openclaw/` 下（例如 `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`）。

    相關資訊：[移轉](/zh-TW/install/migrating)、[檔案在磁碟上的儲存位置](/zh-TW/help/faq#where-things-live-on-disk)、
    [代理程式工作區](/zh-TW/concepts/agent-workspace)、[Doctor](/zh-TW/gateway/doctor)、
    [遠端模式](/zh-TW/gateway/remote)。

  </Accordion>

  <Accordion title="在哪裡可以查看最新版本的新功能？">
    查看 GitHub 變更記錄：
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    最新項目位於頂端。如果最上方區段為**尚未發布**，下一個有日期的
    區段就是最新已發布版本。項目會分組至**重點**、**變更**
    和**修正**（並在需要時加入文件／其他區段）。

  </Accordion>

  <Accordion title="無法存取 docs.openclaw.ai（SSL 錯誤）">
    某些 Comcast／Xfinity 連線會遭 Xfinity Advanced Security 錯誤封鎖
    `docs.openclaw.ai`。請停用該功能或將 `docs.openclaw.ai` 加入允許清單，然後重試。請協助我們
    解除封鎖：[https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status)。

    仍然受阻嗎？文件已鏡像到 GitHub：
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="穩定版與測試版之間的差異">
    **穩定版**與**測試版**是 **npm dist-tags**，並非不同的程式碼分支：

    - `latest` = 穩定版
    - `beta` = 供測試使用的早期建置版本（當測試版不存在或比目前的穩定版本舊時，會回退至 `latest`）

    穩定版本通常會先發布至**測試版**，接著透過明確的升級步驟，
    在不變更版本號的情況下，將同一版本移至 `latest`。維護者
    也可以直接發布至 `latest`。因此，升級後測試版與穩定版可能會指向
    **同一版本**。

    查看變更內容：[CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)。

    如需安裝單行指令，以及測試版與開發版之間的差異，請參閱下一個折疊區塊。

  </Accordion>

  <Accordion title="如何安裝測試版？測試版與開發版有何差異？">
    **測試版**是 npm dist-tag `beta`（升級後可能與 `latest` 相同）。
    **開發版**是 `main`（git）持續變動的最新版本；發布至 npm 時會使用 dist-tag `dev`。

    單行指令（macOS/Linux）：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    Windows 安裝程式（PowerShell）：`iwr -useb https://openclaw.ai/install.ps1 | iex`

    更多詳細資訊：[開發通道](/zh-TW/install/development-channels)與[安裝程式旗標](/zh-TW/install/installer)。

  </Accordion>

  <Accordion title="如何試用最新版本？">
    有兩種選項：

    1. **開發通道（現有安裝）：**

    ```bash
    openclaw update --channel dev
    ```

    此指令會切換至 `main` 的 git checkout、在上游版本上執行 rebase、進行建置，並從
    該 checkout 安裝命令列介面。

    2. **可修改的（git）安裝（全新機器）：**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    建議手動複製儲存庫：

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    文件：[更新](/zh-TW/cli/update)、[開發通道](/zh-TW/install/development-channels)、[安裝](/zh-TW/install)。

  </Accordion>

  <Accordion title="安裝與初始設定通常需要多久？">
    大致時間如下：

    - **安裝：**2-5 分鐘。
    - **快速開始初始設定：**數分鐘（迴路閘道、自動權杖、預設工作區）。
    - **進階／完整初始設定：**如果供應商登入、頻道配對、常駐程式安裝、網路下載或 Skills 需要額外設定，所需時間會更長。

    精靈會預先顯示此時間表。你可以略過選用步驟，稍後再使用
    `openclaw configure` 返回設定。

    卡住了嗎？請參閱上方的[我卡住了](#quick-start-and-first-run-setup)。

  </Accordion>

  <Accordion title="安裝程式卡住了？如何取得更多回饋？">
    加上 `--verbose` 重新執行：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` 沒有專用的詳細輸出開關；請改用 `Set-PSDebug -Trace 1` /
    `-Trace 0` 包裝它。完整旗標參考：[安裝程式旗標](/zh-TW/install/installer)。

  </Accordion>

  <Accordion title="Windows 安裝時顯示找不到 git 或無法辨識 openclaw">
    Windows 上常見的兩個問題：

    **1) npm 錯誤 spawn git／找不到 git**

    - 安裝 **Git for Windows**，並確認 `git` 位於 PATH 中。
    - 關閉並重新開啟 PowerShell，然後再次執行安裝程式。

    **2) 安裝後無法辨識 openclaw**

    - 你的 npm 全域 bin 資料夾不在 PATH 中。
    - 檢查方式：`npm config get prefix`。
    - 將該目錄加入你的使用者 PATH（不需要 `\bin` 後綴；在大多數系統上是 `%AppData%\npm`）。
    - 關閉並重新開啟 PowerShell。

    偏好桌面應用程式嗎？請使用 **Windows Hub**。若只使用終端機設定：PowerShell
    安裝程式和 WSL2 閘道路徑都受到支援。文件：[Windows](/zh-TW/platforms/windows)。

  </Accordion>

  <Accordion title="Windows exec 輸出顯示亂碼中文，該怎麼辦？">
    通常是原生 Windows shell 的主控台字碼頁不相符。

    症狀：`system.run`/`exec` 的輸出將中文顯示為亂碼；相同指令
    在另一個終端機設定檔中則顯示正常。

    PowerShell 中的因應方式：

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    接著重新啟動閘道並重試：

    ```powershell
    openclaw gateway restart
    ```

    在最新的 OpenClaw 上仍會重現嗎？請在此追蹤／回報：[Issue #30640](https://github.com/openclaw/openclaw/issues/30640)。

  </Accordion>

  <Accordion title="文件沒有解答我的問題，如何取得更好的答案？">
    使用可修改的（git）安裝方式，讓完整原始碼與文件都儲存在本機，然後
    **從該資料夾**向你的機器人（或 Claude/Codex）提問，讓它能讀取儲存庫並精確回答。

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    更多詳細資訊：[安裝](/zh-TW/install)與[安裝程式旗標](/zh-TW/install/installer)。

  </Accordion>

  <Accordion title="如何在 Linux 上安裝 OpenClaw？">
    - Linux 快速安裝路徑與服務安裝：[Linux](/zh-TW/platforms/linux)。
    - 完整操作指南：[開始使用](/zh-TW/start/getting-started)。
    - 安裝程式與更新：[安裝與更新](/zh-TW/install/updating)。

  </Accordion>

  <Accordion title="如何在 VPS 上安裝 OpenClaw？">
    任何 Linux VPS 都可以使用。在伺服器上安裝後，透過 SSH/Tailscale 連線至閘道。

    指南：[exe.dev](/zh-TW/install/exe-dev)、[Hetzner](/zh-TW/install/hetzner)、[Fly.io](/zh-TW/install/fly)。
    遠端存取：[閘道遠端存取](/zh-TW/gateway/remote)。

  </Accordion>

  <Accordion title="雲端／VPS 安裝指南在哪裡？">
    常見供應商的託管中心：

    - [VPS 託管](/zh-TW/vps)（所有供應商集中於一處）
    - [Fly.io](/zh-TW/install/fly)
    - [Hetzner](/zh-TW/install/hetzner)
    - [exe.dev](/zh-TW/install/exe-dev)

    在雲端環境中，**閘道會在伺服器上執行**，你可以從筆記型電腦／手機
    透過控制介面（或 Tailscale/SSH）存取。你的狀態與工作區都位於伺服器上，因此
    請將主機視為唯一真實來源，並加以備份。

    將**節點**（Mac/iOS/Android/無介面裝置）與該雲端閘道配對，即可在閘道持續位於
    雲端時，使用筆記型電腦本機的螢幕／相機／畫布，或在本機執行指令。

    中心：[平台](/zh-TW/platforms)。遠端存取：[閘道遠端存取](/zh-TW/gateway/remote)。
    節點：[節點](/zh-TW/nodes)、[節點命令列介面](/zh-TW/cli/nodes)。

  </Accordion>

  <Accordion title="可以要求 OpenClaw 自行更新嗎？">
    可以，但不建議。更新流程可能重新啟動閘道（導致
    使用中的工作階段中斷）、可能需要乾淨的 git checkout，且可能提示你確認。
    由操作者從 shell 執行更新會更安全。

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    從代理程式執行自動化：

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    文件：[更新](/zh-TW/cli/update)、[更新 OpenClaw](/zh-TW/install/updating)。

  </Accordion>

  <Accordion title="初始設定實際上會執行哪些操作？">
    `openclaw onboard` 是建議的設定路徑。在**本機模式**中，它會引導你完成：

    1. **模型／驗證** - 供應商 OAuth、API 金鑰或手動驗證（包括 LM Studio 等本機選項）；選擇預設模型。
    2. **工作區** - 位置與啟動程序檔案。
    3. **閘道** - 連接埠、繫結位址、驗證模式、Tailscale 公開方式。
    4. **頻道** - 內建與官方外掛聊天頻道：iMessage、Discord、Feishu、Google Chat、Mattermost、Microsoft Teams、QQ Bot、Signal、Slack、Telegram、WhatsApp 等。
    5. **常駐程式** - LaunchAgent（macOS）、systemd 使用者單元（Linux/WSL2）或原生 Windows Scheduled Task。
    6. **健康狀態檢查** - 啟動閘道並確認它正在執行。
    7. **Skills** - 安裝建議的 Skills 與選用相依套件。

    它會預先說明所需時間，並在設定的模型不明
    或缺少驗證資訊時發出警告。完整說明：[初始設定（命令列介面）](/zh-TW/start/wizard)。

  </Accordion>

  <Accordion title="是否需要訂閱 Claude 或 OpenAI 才能執行此程式？">
    不需要。你可以使用 **API 金鑰**（Anthropic/OpenAI／其他供應商）或**僅限本機的模型**
    執行 OpenClaw，讓資料保留在你的裝置上。訂閱方案（Claude Pro/Max、ChatGPT/Codex）
    只是驗證這些供應商的選用方式。

    對 Anthropic 而言：**API 金鑰**採用標準的隨用隨付計費；**Claude CLI**
    會重複使用同一主機上的現有 Claude Code 登入。目前 Anthropic 將
    Claude CLI 的非互動式 `claude -p` 路徑視為 Agent SDK／程式化使用方式，
    仍會占用你的訂閱方案額度限制——依賴訂閱行為前，請先查看 Anthropic 目前的計費
    文件。對於長期運作的閘道主機與共用
    自動化，Anthropic API 金鑰是更可預測的選擇。

    代理模型完全支援 OpenAI Codex OAuth（ChatGPT/Codex 訂閱）。
    OpenClaw 也支援託管的訂閱型選項，包括 **Qwen Cloud
    Coding Plan**、**MiniMax Coding Plan** 與 **Z.AI / GLM Coding Plan**。

    文件：[Anthropic](/zh-TW/providers/anthropic)、[OpenAI](/zh-TW/providers/openai)、
    [Qwen Cloud](/zh-TW/providers/qwen)、[MiniMax](/zh-TW/providers/minimax)、[Z.AI (GLM)](/zh-TW/providers/zai)、
    [本機模型](/zh-TW/gateway/local-models)、[模型](/zh-TW/concepts/models)。

  </Accordion>

  <Accordion title="可以在沒有 API 金鑰的情況下使用 Claude Max 訂閱嗎？">
    可以。OpenClaw 支援重複使用 Pro/Max/Team/Enterprise 方案的 Claude CLI。Anthropic
    目前將 OpenClaw 使用的 `claude -p` 路徑視為受你方案額度限制的訂閱方案用量，
    而非獨立的免費額度——請參閱
    [Anthropic](/zh-TW/providers/anthropic)，瞭解目前的計費詳細資訊，以及 Anthropic
    官方支援文章的連結。如需最可預測的伺服器端設定，請改用
    Anthropic API 金鑰。
  </Accordion>

  <Accordion title="是否支援 Claude 訂閱驗證（Claude Pro 或 Max）？">
    支援，可透過重複使用 Claude CLI 實現。Anthropic 對 `claude -p`/Agent SDK 用量的計費方式
    已隨時間改變；在依賴特定計費行為前，請參閱 [Anthropic](/zh-TW/providers/anthropic)
    以瞭解目前狀態，以及 Anthropic 支援文章的附日期連結。

    Anthropic setup-token 認證仍是支援的權杖途徑，但若可用，OpenClaw 會優先使用
    Claude 命令列介面重用與 `claude -p`。對於正式環境或多使用者
    工作負載，Anthropic API 金鑰仍是較安全且更可預測的選擇。其他
    訂閱式託管選項：[OpenAI](/zh-TW/providers/openai)、[Qwen Cloud](/zh-TW/providers/qwen)、
    [MiniMax](/zh-TW/providers/minimax)、[Z.AI (GLM)](/zh-TW/providers/zai)。

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="為什麼會看到來自 Anthropic 的 HTTP 429 rate_limit_error？">
    目前時段的 **Anthropic 配額／速率限制**已用盡。在 **Claude
    命令列介面**中，請等待時段重設或升級方案。若使用 **Anthropic API 金鑰**，
    請在 Anthropic Console 中檢查用量／帳務，並視需要提高限制。

    如果訊息明確為 `Extra usage is required for long context requests`，
    表示要求正嘗試使用 Anthropic 的 1M 上下文視窗（支援正式提供的 1M Claude 4.x
    模型，或舊版 `params.context1m: true` 設定），而你目前的認證資訊
    不符合長上下文計費資格。

    設定**備援模型**，讓供應商受到速率限制時，OpenClaw 仍能持續回覆。
    請參閱[模型](/zh-TW/cli/models)、[OAuth](/zh-TW/concepts/oauth)，以及
    [Anthropic 429：長上下文需要額外用量](/zh-TW/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context)。

  </Accordion>

  <Accordion title="支援 AWS Bedrock 嗎？">
    支援。OpenClaw 內建 **Amazon Bedrock (Converse)** 供應商。若存在 AWS 環境
    標記（`AWS_ACCESS_KEY_ID`、`AWS_PROFILE`、`AWS_BEARER_TOKEN_BEDROCK`），
    OpenClaw 會自動啟用隱含的 Bedrock 供應商以探索模型；否則
    請設定 `plugins.entries.amazon-bedrock.config.discovery.enabled: true` 或新增手動
    供應商項目。請參閱 [Amazon Bedrock](/zh-TW/providers/bedrock) 與[模型供應商](/zh-TW/providers/models)。
    如果偏好受管理的金鑰流程，在 Bedrock 前方使用 OpenAI 相容的 Proxy 仍是可行選項。
  </Accordion>

  <Accordion title="Codex 認證如何運作？">
    OpenClaw 透過 OAuth（ChatGPT 登入）支援 **OpenAI Codex**。未設定主要模型的全新
    設定會使用確切的 `openai/gpt-5.6-sol`，進行
    ChatGPT/Codex 訂閱認證並使用原生 Codex app-server 執行。
    重新認證會保留既有的明確模型設定，包括
    `openai/gpt-5.5`。如果 Codex 工作區未提供 GPT-5.6，請明確選取
    `openai/gpt-5.5`；OpenClaw 不會在未告知的情況下降級。舊版
    Codex 前綴模型參照屬於舊版設定，會由 `openclaw doctor
    --fix` 修復。對於非代理程式的 OpenAI
    API 介面，仍可直接使用 OpenAI API 金鑰；透過排序過的 `openai` API 金鑰設定檔，
    代理程式模型也同樣可用。請參閱[模型供應商](/zh-TW/concepts/model-providers)與
    [新手引導（命令列介面）](/zh-TW/start/wizard)。
  </Accordion>

  <Accordion title="為什麼 OpenClaw 仍會提到舊版 OpenAI Codex 前綴？">
    `openai` 是 OpenAI API 金鑰與
    ChatGPT/Codex OAuth 目前共同使用的供應商及認證設定檔 ID，OpenAI Codex 已整合至其中。你可能仍會在舊版設定與遷移警告中看到舊版
    `openai-codex` 前綴：

    - `openai/gpt-5.6-sol` = 全新的 ChatGPT/Codex 訂閱設定，代理程式回合使用原生 Codex 執行階段。
    - `openai/gpt-5.5` = 既有設定或無法存取 GPT-5.6 的帳號可明確選取的受支援選項。
    - 舊版 `openai-codex/*` 模型參照 = 由 `openclaw doctor --fix` 修復的舊版路由。
    - `openai/gpt-5.5` 加上排序過的 `openai` API 金鑰設定檔 = OpenAI 代理程式模型的 API 金鑰認證。
    - 舊版 `openai-codex` 認證設定檔 ID = 由 `openclaw doctor --fix` 遷移的舊版 ID。

    想直接使用 OpenAI Platform 計費？請設定 `OPENAI_API_KEY`。想使用 ChatGPT/Codex
    訂閱認證？請執行 `openclaw models auth login --provider openai`。請將
    模型參照保留在標準 `openai/*` 供應商下。全新訂閱
    設定會使用確切的 `openai/gpt-5.6-sol`；doctor 會修復具有舊版 Codex 前綴的
    參照，而不會升級明確的 `openai/gpt-5.5` 選項。

  </Accordion>

  <Accordion title="為什麼 Codex OAuth 限制可能與 ChatGPT 網頁版不同？">
    Codex OAuth 使用由 OpenAI 管理、取決於方案的配額時段；即使使用相同帳號，
    也可能與 ChatGPT 網站／應用程式的體驗不同。

    `openclaw models status` 會顯示目前可見的供應商用量／配額時段，但
    不會虛構權益，也不會將 ChatGPT 網頁版權益正規化為直接 API 存取。如要使用
    OpenAI Platform 的直接計費／限制途徑，請搭配 API 金鑰使用 `openai/*`。

  </Accordion>

  <Accordion title="支援 OpenAI 訂閱認證（Codex OAuth）嗎？">
    是，完整支援。OpenAI 明確允許在 OpenClaw 等外部
    工具／工作流程中使用訂閱 OAuth。新手引導可代你執行 OAuth 流程。

    請參閱 [OAuth](/zh-TW/concepts/oauth)、[模型供應商](/zh-TW/concepts/model-providers)與[新手引導（命令列介面）](/zh-TW/start/wizard)。

  </Accordion>

  <Accordion title="如何設定 Gemini 命令列介面 OAuth？">
    Gemini 命令列介面使用**外掛認證流程**，而不是 `openclaw.json` 中的用戶端 ID 或密鑰。

    1. 在本機安裝 Gemini 命令列介面，讓 `gemini` 位於 `PATH`：
       - Homebrew：`brew install gemini-cli`
       - npm：`npm install -g @google/gemini-cli`
    2. 啟用外掛：`openclaw plugins enable google`
    3. 登入：`openclaw models auth login --provider google-gemini-cli --set-default`
    4. 登入後的預設模型：`google/gemini-3.1-pro-preview`（執行階段為 `google-gemini-cli`）
    5. 登入後要求失敗？請在閘道主機上設定 `GOOGLE_CLOUD_PROJECT` 或 `GOOGLE_CLOUD_PROJECT_ID`，然後重試。

    OAuth 權杖會儲存在閘道主機的認證設定檔中。詳細資訊：[Google](/zh-TW/providers/google)、[模型供應商](/zh-TW/concepts/model-providers)。

  </Accordion>

  <Accordion title="本機模型適合日常聊天嗎？">
    通常不適合。OpenClaw 需要大型上下文與強大的安全防護；小型顯示卡會截斷上下文，
    並略過供應商端的安全篩選器。如果一定要使用，請在本機執行可負荷的**最大型**
    模型版本（LM Studio）—請參閱[本機模型](/zh-TW/gateway/local-models)。較小型／量化的
    模型會提高提示注入風險—請參閱[安全性](/zh-TW/gateway/security)。
  </Accordion>

  <Accordion title="如何讓託管模型流量留在特定區域？">
    選擇固定區域的端點。OpenRouter 為 MiniMax、Kimi
    與 GLM 提供位於美國的選項；請選擇位於美國的版本，讓資料留在該區域。你仍可透過 `models.mode: "merge"`
    將 Anthropic/OpenAI 與這些選項一併列出，讓備援保持
    可用，同時遵循所選的區域供應商。
  </Accordion>

  <Accordion title="一定要購買 Mac Mini 才能安裝嗎？">
    不用。OpenClaw 可在 macOS 或 Linux 上執行（Windows 則透過 WSL2）。Mac mini 是熱門的
    常時開機主機選擇，但小型 VPS、家用伺服器或 Raspberry Pi 等級的裝置也能使用。

    只有使用 **macOS 專用工具**時才需要 Mac。若要使用 iMessage，請在任何已登入 Messages 的 Mac 上
    搭配 `imsg` 使用 [iMessage](/zh-TW/channels/imessage)；如果閘道在 Linux 或其他位置執行，
    請將 `channels.imessage.cliPath` 設為 SSH 包裝器，以在該 Mac 上執行 `imsg`。對於其他
    macOS 專用工具，請在 Mac 上執行閘道，或配對 macOS 節點。

    文件：[iMessage](/zh-TW/channels/imessage)、[節點](/zh-TW/nodes)、[Mac 遠端模式](/zh-TW/platforms/mac/remote)。

  </Accordion>

  <Accordion title="支援 iMessage 是否需要 Mac mini？">
    需要**某部 macOS 裝置**登入 Messages，但不一定是 Mac mini，任何
    Mac 都可以。請搭配 `imsg` 使用 [iMessage](/zh-TW/channels/imessage)；閘道可在該
    Mac 上執行，也可在其他位置透過 SSH 包裝器 `cliPath` 執行。

    常見設定：

    - 閘道位於 Linux/VPS，將 `channels.imessage.cliPath` 設為 SSH 包裝器，以在已登入 Messages 的 Mac 上執行 `imsg`。
    - 全部在同一部 Mac 上執行，這是最簡單的單機設定。

    文件：[iMessage](/zh-TW/channels/imessage)、[節點](/zh-TW/nodes)、[Mac 遠端模式](/zh-TW/platforms/mac/remote)。

  </Accordion>

  <Accordion title="如果購買 Mac mini 執行 OpenClaw，可以將它連接到 MacBook Pro 嗎？">
    可以。**Mac mini 可執行閘道**，而 MacBook Pro 則以**節點**
    （配套裝置）身分連線。節點不會執行閘道，而是新增該裝置上的
    螢幕／相機／畫布與 `system.run` 等功能。

    常見模式：閘道在常時開機的 Mac mini 上執行；MacBook Pro 則執行 macOS 應用程式或
    節點主機，並與閘道配對。使用 `openclaw nodes status`／`openclaw nodes list` 檢查。

    文件：[節點](/zh-TW/nodes)、[節點命令列介面](/zh-TW/cli/nodes)。

  </Accordion>

  <Accordion title="可以使用 Bun 嗎？">
    可以使用 Bun 安裝相依套件或執行套件指令碼。OpenClaw 命令列介面與
    閘道需要**節點**，因為標準狀態儲存區使用 `node:sqlite`；Bun
    不提供該 API。
  </Accordion>

  <Accordion title="Telegram：allowFrom 中應填入什麼？">
    `channels.telegram.allowFrom` 是**真人傳送者的 Telegram 使用者 ID**（數字），
    不是 Bot 使用者名稱。設定只接受數字使用者 ID；`openclaw doctor --fix`
    可嘗試解析舊版 `@username` 項目。

    較安全（不使用第三方 Bot）：私訊你的 Bot，執行 `openclaw logs --follow`，讀取 `from.id`。

    官方 Bot API：私訊你的 Bot，呼叫 `https://api.telegram.org/bot<bot_token>/getUpdates`，讀取 `message.from.id`。

    第三方（隱私性較低）：私訊 `@userinfobot` 或 `@getidsbot`。

    請參閱 [Telegram 存取控制](/zh-TW/channels/telegram#access-control-and-activation)。

  </Accordion>

  <Accordion title="多個人可以透過不同的 OpenClaw 執行個體共用一個 WhatsApp 號碼嗎？">
    可以，透過**多代理程式路由**。將每位傳送者的 WhatsApp 私訊（`peer: { kind: "direct", id: "+15551234567" }`）繫結至不同的 `agentId`，讓每個人都有自己的工作區與工作階段儲存區。回覆仍會來自**同一個 WhatsApp 帳號**；每個帳號的私訊存取控制（`channels.whatsapp.dmPolicy`／`channels.whatsapp.allowFrom`）為全域設定。請參閱[多代理程式路由](/zh-TW/concepts/multi-agent)與 [WhatsApp](/zh-TW/channels/whatsapp)。
  </Accordion>

  <Accordion title='可以同時執行「快速聊天」代理程式與「使用 Opus 編寫程式碼」代理程式嗎？'>
    可以。使用多代理程式路由：為每個代理程式設定各自的預設模型，然後將傳入
    路由（供應商帳號或特定對象）繫結至各代理程式。設定範例：
    [多代理程式路由](/zh-TW/concepts/multi-agent)。另請參閱[模型](/zh-TW/concepts/models)與
    [設定](/zh-TW/gateway/configuration)。
  </Accordion>

  <Accordion title="Homebrew 可在 Linux 上使用嗎？">
    可以，透過 Linuxbrew：

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    透過 systemd 執行 OpenClaw：請確保服務的 PATH 包含
    `/home/linuxbrew/.linuxbrew/bin`（或你的 brew 前綴），讓以 `brew` 安裝的工具
    能在非登入 Shell 中解析。近期版本也會在 Linux
    systemd 服務中前置常見的使用者二進位目錄（例如 `~/.local/bin`、`~/.npm-global/bin`、
    `~/.local/share/pnpm`、`~/.bun/bin`），並在已設定時採用 `PNPM_HOME`、`NPM_CONFIG_PREFIX`、
    `BUN_INSTALL`、`VOLTA_HOME`、`ASDF_DATA_DIR`、`NVM_DIR` 與 `FNM_DIR`。

  </Accordion>

  <Accordion title="可修改的 git 安裝與 npm 安裝有何差異？">
    - **可修改（git）安裝：**完整原始碼簽出，可編輯，最適合貢獻者。你可以在本機建置及修補程式碼／文件。
    - **npm 安裝：**全域命令列介面安裝，不含儲存庫，最適合「安裝後直接執行」。更新來自 npm dist-tags。

    文件：[開始使用](/zh-TW/start/getting-started)、[更新](/zh-TW/install/updating)。

  </Accordion>

  <Accordion title="之後可以在 npm 與 git 安裝之間切換嗎？">
    可以，在現有安裝上使用 `openclaw update --channel ...` 即可。這**不會
    刪除你的資料**，只會變更 OpenClaw 程式碼的安裝方式。狀態（`~/.openclaw`）和
    工作區（`~/.openclaw/workspace`）都不受影響。

    從 npm 切換至 git：

    ```bash
    openclaw update --channel dev
    ```

    從 git 切換至 npm：

    ```bash
    openclaw update --channel stable
    ```

    加上 `--dry-run`，可先預覽規劃的模式切換。更新程式會執行 Doctor
    後續作業、重新整理目標頻道的外掛來源，並重新啟動閘道，
    除非你傳入 `--no-restart`。

    安裝程式也可以強制使用任一模式：

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    備份提示：[檔案在磁碟上的位置](/zh-TW/help/faq#where-things-live-on-disk)。

  </Accordion>

  <Accordion title="我應該在筆記型電腦還是 VPS 上執行閘道？">
    想要 24/7 的可靠性嗎？請使用 **VPS**。想要最省事，而且可以接受
    睡眠／重新啟動嗎？請在本機執行。

    **筆記型電腦（本機閘道）**

    - **優點：**無伺服器成本、可直接存取本機檔案，並有可見的瀏覽器視窗。
    - **缺點：**睡眠／網路中斷會使其離線、作業系統更新／重新啟動會造成中斷，而且電腦必須保持喚醒。

    **VPS／雲端**

    - **優點：**持續上線、網路穩定、不受筆記型電腦睡眠影響，也更容易維持運作。
    - **缺點：**通常沒有圖形介面（請使用螢幕截圖）、只能遠端存取檔案，而且更新時需要 SSH。

    WhatsApp／Telegram／Slack／Mattermost／Discord 都能在 VPS 上順利運作，真正的
    取捨在於無頭瀏覽器與可見視窗之間的選擇。請參閱[瀏覽器](/zh-TW/tools/browser)。

    預設建議：如果你過去曾遇到閘道中斷連線，請使用 VPS；如果你會主動使用 Mac，
    並需要存取本機檔案或透過可見瀏覽器介面進行自動化，本機執行會很合適。

  </Accordion>

  <Accordion title="在專用機器上執行 OpenClaw 有多重要？">
    這不是必要條件，但為了可靠性與隔離性，建議這麼做。

    - **專用主機（VPS／Mac mini／Raspberry Pi）：**持續上線、較少因睡眠／重新啟動而中斷、權限更單純，也更容易維持運作。
    - **共用筆記型／桌上型電腦：**適合測試與主動使用，但機器進入睡眠或更新時，預期會暫停運作。

    兼得兩者優點的方法：將閘道保留在專用主機上，並將筆記型電腦配對為
    **節點**，以使用本機螢幕／相機／執行工具。請參閱[節點](/zh-TW/nodes)和[安全性](/zh-TW/gateway/security)。

  </Accordion>

  <Accordion title="VPS 的最低需求和建議作業系統是什麼？">
    - **絕對最低需求：**1 個 vCPU、1 GB RAM、約 500 MB 磁碟空間。
    - **建議配備：**1-2 個 vCPU、2 GB 以上 RAM，以保留餘裕（記錄、媒體、多個頻道）。節點工具和瀏覽器自動化可能會耗用大量資源。

    作業系統：**Ubuntu LTS**（或任何現代版本的 Debian／Ubuntu），這是經過最充分測試的 Linux 安裝途徑。

    文件：[Linux](/zh-TW/platforms/linux)、[VPS 託管](/zh-TW/vps)。

  </Accordion>

  <Accordion title="我可以在 VM 中執行 OpenClaw 嗎？需求為何？">
    可以。將 VM 視同 VPS：它必須持續開機、可連線，並具備足夠的 RAM，
    供閘道和你啟用的所有頻道使用。

    - **絕對最低需求：**1 個 vCPU、1 GB RAM。
    - **建議配備：**若使用多個頻道、瀏覽器自動化或媒體工具，建議配備 2 GB 以上 RAM。
    - **作業系統：**Ubuntu LTS 或其他現代版本的 Debian／Ubuntu。

    在 Windows 上，請使用 **Windows Hub** 進行桌面設定，或使用 WSL2 建立 Linux 風格的閘道 VM，
    以獲得廣泛的工具相容性。請參閱 [Windows](/zh-TW/platforms/windows)、[VPS 託管](/zh-TW/vps)。
    在 VM 中執行 macOS：請參閱 [macOS VM](/zh-TW/install/macos-vm)。

  </Accordion>
</AccordionGroup>

## 相關內容

- [常見問題](/zh-TW/help/faq)－主要常見問題（模型、工作階段、閘道、安全性及更多內容）
- [安裝概覽](/zh-TW/install)
- [開始使用](/zh-TW/start/getting-started)
- [疑難排解](/zh-TW/help/troubleshooting)
