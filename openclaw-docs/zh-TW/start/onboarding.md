---
read_when:
    - 設計 macOS 初始設定助理
    - 實作驗證或身分設定
sidebarTitle: 'Onboarding: macOS App'
summary: OpenClaw 首次執行設定流程（macOS App）
title: 新手引導（macOS App）
x-i18n:
    generated_at: "2026-07-26T08:43:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 55154774886c530de92b2110d367af24e2142fac48b901f288582d8552a6ca10
    source_path: start/onboarding.md
    workflow: 16
---

macOS App 的首次執行流程：選擇閘道的執行位置、連接已驗證的 AI 後端、授予權限，然後交由代理程式執行自己的啟動設定流程。
如需命令列介面引導設定及兩種路徑的比較，請參閱[引導設定概覽](/zh-TW/start/onboarding-overview)。

<Steps>
<Step title="核准 macOS 警告">
<Frame>
<img src="/assets/macos-onboarding/01-macos-warning.jpeg" alt="" />
</Frame>
</Step>
<Step title="允許尋找區域網路">
<Frame>
<img src="/assets/macos-onboarding/02-local-networks.jpeg" alt="" />
</Frame>
</Step>
<Step title="歡迎與安全性注意事項">
<Frame caption="閱讀顯示的安全性注意事項，並據此決定">
<img src="/assets/macos-onboarding/03-security-notice.png" alt="" />
</Frame>

安全性信任模型：

- OpenClaw 預設為個人代理程式：採用單一受信任操作者邊界。
- 共用／多使用者設定需要加強鎖定：分隔信任邊界、將工具存取權限降至最低，並遵循[安全性](/zh-TW/gateway/security)指引。
- 本機引導設定會將新設定預設為 `tools.profile: "coding"`，讓全新設定保有檔案系統／執行階段工具，同時不使用不受限制的 `full` 設定檔。
- 若已啟用掛鉤／網路鉤子或其他不受信任的內容來源，請使用強大的現代模型層級，並維持嚴格的工具政策／沙箱隔離。

</Step>
<Step title="本機與遠端">
<Frame>
<img src="/assets/macos-onboarding/04-choose-gateway.png" alt="" />
</Frame>

**閘道**要在哪裡執行？

- **這台 Mac（僅限本機）：**引導設定會在本機設定驗證並寫入認證資訊。
- **遠端（透過 SSH／Tailnet）：**引導設定**不會**設定本機驗證；
  認證資訊必須已存在於閘道主機上。遠端閘道權杖
  欄位會儲存 macOS App 用來連接該閘道的權杖；
  現有的 `gateway.remote.token` SecretRef 值會保留，直到你
  將其取代。
- **稍後設定：**略過設定，讓 App 維持未設定狀態。

<Tip>
**閘道驗證提示：**

- 即使繫結至回送介面，閘道驗證模式也預設為 `token`，因此本機 WS 用戶端必須進行驗證。
- 設定 `gateway.auth.mode: "none"` 後，任何本機程序皆可連線；僅限在完全受信任的機器上使用此設定。
- 多機器存取或非回送介面繫結應使用權杖。

</Tip>
</Step>
<Step title="命令列介面">
  本機設定會透過 npm、pnpm 或 bun 安裝全域 `openclaw` 命令列介面，
  並優先使用 npm。對閘道本身而言，Node 仍是建議的執行階段。
  現有且相容的安裝會直接重複使用。
</Step>
<Step title="連接你的 AI">
  若已連線的閘道已有設定完成的代理程式模型，便會完全略過此
  頁面，並開啟一般代理程式 UI。只有全新或未完成設定的閘道
  才會執行 OpenClaw 與供應商設定。

閘道就緒後，引導設定會尋找你已有的 AI 存取方式：
Claude Code 或 Codex 登入、`OPENAI_API_KEY`／`ANTHROPIC_API_KEY`，或是已
安裝於可連線的 Ollama 或 LM Studio 伺服器中、支援工具且經測量具備至少
16K 有效上下文的模型。偵測會在閘道主機上執行，
包括 macOS App 連接至 Linux 閘道時。系統會以真實的補全測試最佳
選項，且只在其成功回應後儲存；
測試失敗時，App 會自動嘗試下一個選項，
並顯示上一個選項失敗的原因。若找到多個選項，你可以
在繼續之前切換選項。自動本機探索絕不會提取
或下載模型。

若要在閘道主機沒有 Claude 命令列介面登入的情況下使用 Claude 訂閱，請在
任何已安裝 Claude Code 的機器上執行
`claude setup-token`，然後將輸出的權杖貼到 **Connect with an API key or
token** 下的 **Anthropic setup-token**。

若已安裝 Gemini CLI、Antigravity、Pi 和 OpenCode 命令列介面，但無法將其選為可重複使用的引導式設定推論路徑，
仍會顯示這些工具以供參考。
Gemini 和 Antigravity 無法強制執行不使用工具的推論探測。Pi 和
OpenCode 是完整的代理程式框架，而非設定推論路徑；其
工作階段整合需要另外設定執行階段與外掛。

你也可以透過供應商自有的 OAuth 或裝置配對流程登入。
內建選項包括 OpenAI／ChatGPT、OpenRouter、GitHub Copilot、Google
Gemini CLI、xAI、MiniMax Global 和 CN，以及 Chutes。此清單來自
閘道啟用中的文字推論供應商外掛，而非 App 的固定清單，
因此其他供應商不需要新增供應商專用的 macOS 程式碼即可選擇加入。

手動金鑰／權杖選擇器使用相同的供應商登錄。在每種路徑中，
供應商會提供其入門模型與設定；OpenClaw 會在儲存其驗證設定檔前，
使用相同的即時測試驗證認證資訊。在任一後端通過測試前，
下一步會保持鎖定，因此沒有可正常運作的推論時，無法
開始第一次代理程式聊天。通過即時檢查後，即可使用 OpenClaw
協助設定其餘的工作區、閘道、頻道及
其他選用功能。當 OpenClaw 提供簡短的選項清單時，App
會顯示原生選項卡片；選擇其中一項會送出該選擇，而 **Skip for
now** 一律會讓該選項維持非必選。之後也可在
Settings → OpenClaw 中使用 OpenClaw。
</Step>
<Step title="匯入記憶（偵測到時顯示）">
針對本機閘道，引導設定會檢查 Mac 上來自支援 AI
工具的記憶：Claude Code 自動記憶、Codex 整合記憶，以及 Hermes 記憶
檔案。找到任何記憶時，此頁面會列出各個來源及其記憶數量，
並讓你將選取的來源匯入代理程式工作區的
`memory/imports/` 下，以供建立索引後喚回。已匯入的檔案會略過，
而沒有任何可匯入項目時，此頁面絕不會出現。略過此步驟不會造成問題；
之後可在儀表板的「記憶匯入」頁面進行相同的匯入，並可逐一控制檔案。
</Step>
<Step title="權限">

<Frame caption="選擇你要授予 OpenClaw 哪些權限">
<img src="/assets/macos-onboarding/05-permissions.png" alt="" />
</Frame>

引導設定會要求下列 TCC 權限：自動化（AppleScript）、通知、輔助使用、螢幕錄製、麥克風、語音辨識、相機及定位服務。

</Step>
<Step title="完成">
  推論通過後，OpenClaw 會接手其餘選用設定，
  並可將你帶到一般代理程式聊天。完成權限引導流程
  也會開啟相同的聊天；App 不會在 OpenClaw 之前建立工作區，
  或啟動另一個代理程式設定對話。若要了解代理程式首次實際互動期間
  在閘道主機上發生的情況，請參閱
  [啟動設定](/zh-TW/start/bootstrapping)。
</Step>
</Steps>

## 相關內容

- [引導設定概覽](/zh-TW/start/onboarding-overview)
- [開始使用](/zh-TW/start/getting-started)
