---
read_when:
    - 你想在 OpenClaw 中使用 OpenAI 模型
    - 你想使用 Codex 訂閱驗證，而不是 API 金鑰
    - 你需要更嚴格的 GPT-5 代理程式執行行為
summary: 在 OpenClaw 中透過 API 金鑰或 Codex 訂閱使用 OpenAI
title: OpenAI
x-i18n:
    generated_at: "2026-07-26T08:34:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 612a36760899e01126364ddca523f0a6340036253cf349ae2755ba15c6451ba6
    source_path: providers/openai.md
    workflow: 16
---

OpenClaw 對直接 API 金鑰驗證與 ChatGPT/Codex 訂閱驗證使用同一個提供者 ID：`openai`。
`openai/*` 是標準模型路由。對於執行階段原則未設定或設為
`auto` 的內嵌代理程式回合，OpenAI 的路由資訊會決定 OpenClaw 是否可隱含選取隨附的 Codex app-server 執行階段。僅有 `openai/*` 前綴不會選取執行階段。

- **代理程式模型** - 透過明確的 `agentRuntime` 設定或 OpenAI 的隱含路由原則所選取的執行階段，使用 `openai/*`。若要使用 ChatGPT/Codex 訂閱，請以 Codex 驗證登入；若要採用金鑰計費，請設定 API 金鑰驗證設定檔。
- **非代理程式 OpenAI API** - 透過 `OPENAI_API_KEY` 或 `openai` API 金鑰驗證設定檔直接存取 OpenAI Platform，並按用量計費。
- **舊版設定** - `codex/*` 和 `openai-codex/*` 參照會由 `openclaw doctor --fix` 修復為 `openai/*`，並加上模型範圍的 `agentRuntime.id: "codex"`。

OpenAI 明確支援在 OpenClaw 等外部工具和工作流程中使用訂閱 OAuth。

## 用量與成本追蹤

OpenClaw 會將訂閱配額與 Platform API 計費分開處理：

- ChatGPT/Codex OAuth 會顯示訂閱方案、配額週期和點數餘額。
- `OPENAI_ADMIN_KEY` 會在 Control UI 的**用量**中顯示提供者回報的最近 30 天組織成本與 completions 用量，包括每日支出、請求／權杖總數、熱門模型和成本類別。
- `OPENAI_PROJECT_ID` 可選擇將 Admin API 歷程限定於單一專案。
- OpenClaw 絕不會將 `OPENAI_API_KEY` 或 `openai` 推論設定檔傳送至組織 API；這些認證資訊可能屬於自訂、Azure 或代理程式本機端點。

明確指定的 Admin 金鑰優先於 OAuth。提供者回報的歷程不會與 OpenClaw 根據工作階段推算的估計成本合併；其中可能包含其他用戶端的 API 活動及提供者端的計費調整。

OpenAI 的 [API 用量儀表板](https://help.openai.com/en/articles/10478918)文件說明了存取用量資料所需的組織擁有者身分，以及明確的 Usage Dashboard 權限要求。

提供者、模型、執行階段和頻道是彼此獨立的層級。如果這些標籤混淆在一起，請先閱讀[代理程式執行階段](/zh-TW/concepts/agent-runtimes)，再變更設定。

## 快速選擇

| 目標                                              | 使用                                                                | 備註                                                               |
| ------------------------------------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| ChatGPT/Codex 訂閱、原生 Codex 執行階段  | `openai/gpt-5.6-sol`                                               | 全新訂閱設定；以 Codex 驗證登入。                  |
| 代理程式回合直接以 API 金鑰計費            | `openai/gpt-5.6` 加上有序的 API 金鑰驗證設定檔              | 全新 API 金鑰設定；未限定的直接 API ID 會解析為 Sol。        |
| 選擇確切的 GPT-5.6 層級                      | `openai/gpt-5.6-sol`、`-terra` 或 `-luna`                         | 查看 `models list`，確認此帳號可用的層級。        |
| 無 GPT-5.6 存取權的帳號                    | `openai/gpt-5.5`                                                   | 明確的復原選項；OpenClaw 不會無提示地降級。     |
| 直接以 API 金鑰計費，明確使用 OpenClaw 執行階段 | `openai/gpt-5.6` 加上提供者／模型 `agentRuntime.id: "openclaw"` | 選擇一般的 `openai` API 金鑰設定檔。                           |
| 最新的 ChatGPT Instant 模型別名                | `openai/chat-latest`                                               | 僅限直接 API 金鑰；這是動態別名，不是穩定的預設值。          |
| 產生或編輯圖片                       | `openai/gpt-image-2`                                               | 可搭配 `OPENAI_API_KEY` 或 Codex OAuth 使用。                         |
| 透明背景圖片                     | `openai/gpt-image-1.5`                                             | 將 `outputFormat` 設為 `png` 或 `webp`，並設定 `background=transparent`。 |

## 名稱對照表

| 看到的名稱                            | 層級             | 意義                                                                                  |
| --------------------------------------- | ----------------- | ---------------------------------------------------------------------------------------- |
| `openai`                                | 提供者前綴   | 標準 OpenAI 模型路由；路由資訊會決定隱含執行階段。                |
| `codex` 外掛                          | 外掛            | 提供原生 Codex app-server 執行階段和 `/codex` 聊天控制項的隨附外掛。 |
| 提供者／模型 `agentRuntime.id: codex` | 代理程式執行階段     | 強制符合條件的內嵌回合使用原生 Codex app-server 控制框架。                   |
| `/codex ...`                            | 聊天命令集  | 從對話繫結／控制 Codex app-server 執行緒。                               |
| `runtime: "acp", agentId: "codex"`      | ACP 工作階段路由 | 明確的備援路徑，透過 ACP/acpx 執行 Codex。                                 |

## 隱含代理程式執行階段

當提供者／模型 `agentRuntime` 原則未設定或設為 `auto` 時，由 OpenAI 提供者擁有的路由原則會根據實際端點和配接器選擇隱含執行階段：

| 實際路由資訊                                                                                                                                                  | 隱含執行階段      |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------- |
| 確切的官方 Platform HTTPS 端點搭配 `openai-responses`，或確切的官方 ChatGPT HTTPS 端點搭配 `openai-chatgpt-responses`；沒有自行指定的請求覆寫 | 可選取 Codex |
| 自行指定的 `openai-completions` 配接器                                                                                                                                  | OpenClaw              |
| 自訂端點                                                                                                                                                        | OpenClaw              |
| 明確指定且完全符合官方端點，但使用 HTTP                                                                                                                            | 拒絕              |
| 具有自行指定的提供者／模型請求覆寫之路由                                                                                                                 | OpenClaw              |

明確指定的非預設提供者／模型 `agentRuntime.id` 仍具決定權。例如，`agentRuntime.id: "openclaw"` 會讓原本符合 Codex 資格的路由繼續使用 OpenClaw，而 `agentRuntime.id: "codex"` 則要求使用 Codex，並在實際路由未宣告與 Codex 相容時採取封閉式失敗。執行階段選擇不會變更認證資訊類型或計費方式：Platform API 金鑰驗證和 ChatGPT/Codex 訂閱驗證仍彼此獨立。

`openclaw doctor --fix` 會將舊版 `codex/*` 和 `openai-codex/*` 模型參照、舊版 Codex 驗證設定檔 ID，以及舊版 Codex 驗證順序項目遷移至標準 `openai` 路由。遷移後的模型參照會獲得模型範圍的 `agentRuntime.id: "codex"`；新的驗證順序設定請使用 `auth.order.openai`。

<Note>
只有在尚未設定主要模型時，全新的 OpenAI 設定才會套用 GPT-5.6 主要模型。新增或重新整理 OpenAI 驗證時，會保留現有的明確選擇，包括 `openai/gpt-5.5`，除非你明確使用 `models auth login --set-default` 或 `models set`。只有在代理程式模型需要使用 API 金鑰驗證時，才使用 API 金鑰驗證設定檔。
</Note>

## GPT-5.6 限量預覽

OpenClaw 可辨識確切的 `openai/gpt-5.6-sol`、`openai/gpt-5.6-terra` 和 `openai/gpt-5.6-luna` 模型 ID。在目前的目錄中，這三者都提供 `xhigh` 和 `max` 推理。OpenAI 將 Sol 描述為旗艦層級、Terra 為均衡層級，而 Luna 則為快速且成本較低的層級。請參閱 [GPT-5.6 發布公告](https://openai.com/index/previewing-gpt-5-6-sol/)和[存取指南](https://help.openai.com/en/articles/20001325-a-preview-of-gpt-5-6-sol-terra-and-luna)。

使用直接 OpenAI API 金鑰驗證時，未限定的 `openai/gpt-5.6` ID 是 Sol 的別名，也是全新設定的預設值。原生 Codex 目錄不會在用戶端套用此直接 API 別名；視工作區的存取權而定，它可能顯示確切的 Sol、Terra 和 Luna ID。因此，全新的 ChatGPT/Codex OAuth 設定會使用 `openai/gpt-5.6-sol`。請使用以下命令查看目前帳號：

```bash
openclaw models list --provider openai
```

API 組織與 Codex 工作區的存取權可能不同。如果 GPT-5.6 無法使用，請明確選取 GPT-5.5：

```bash
openclaw models set openai/gpt-5.5
```

OpenClaw 會顯示上游存取錯誤，不會無提示地將 GPT-5.6 選項替換為 GPT-5.5。

<Note>
當執行階段原則未設定或設為 `auto` 時，符合條件且完全相符的官方 HTTPS 路由可以選取隨附的 Codex app-server 外掛；自行指定的 Completions 路由、自訂端點和請求傳輸覆寫仍會使用 OpenClaw。使用純文字 HTTP 的官方端點會遭拒絕。明確的提供者／模型執行階段設定仍具決定權。執行 `openclaw doctor --fix`，可修復過時的舊版 Codex 模型參照、`codex-cli/*` 參照，或並非由明確執行階段設定所指定的舊執行階段工作階段固定值。
</Note>

## OpenClaw 功能涵蓋範圍

| OpenAI 功能         | OpenClaw 介面                                                                              | 狀態                                                          |
| ------------------------- | --------------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| 聊天／Responses          | `openai/<model>` 模型供應商                                                               | 是                                                             |
| Codex 訂閱模型 | 搭配 OpenAI OAuth 的 `openai/<model>`                                                            | 是                                                             |
| 舊版 Codex 模型參照   | 舊版 Codex 模型參照、`codex-cli/<model>`                                                     | 由 doctor 修復為 `openai/<model>`                          |
| Codex app-server 執行框架  | 執行階段未設定／`auto` 的 Codex 相容 HTTPS 路由，或明確指定 `agentRuntime.id: codex`  | 是                                                             |
| 伺服器端網頁搜尋    | 原生 OpenAI Responses 工具                                                                  | 是，前提是已啟用網頁搜尋且未固定使用其他供應商 |
| 圖片                    | `image_generate`                                                                              | 是                                                             |
| 影片                    | `video_generate`                                                                              | 是                                                             |
| 文字轉語音            | `tts.provider: "openai"`／`tts`                                                              | 是                                                             |
| 批次語音轉文字      | `tools.media.audio`／媒體理解                                                     | 是                                                             |
| 串流語音轉文字  | 語音通話 `streaming.provider: "openai"`                                                     | 是                                                             |
| 即時語音            | 語音通話 `realtime.provider: "openai"`／控制介面對話 `talk.realtime.provider: "openai"` | 是（OpenAI Platform API 金鑰）                                   |
| 嵌入                | 記憶嵌入供應商                                                                     | 是                                                             |

<Note>
OpenAI 即時語音會透過公開的 **OpenAI Platform Realtime
API**，且需要 Platform API 金鑰。Codex OAuth 權杖則用於驗證
ChatGPT Codex 後端；兩者無法互換，Codex OAuth 權杖不能用於公開 Realtime 端點的 Platform API
金鑰驗證。

若使用 API 金鑰驗證時回報缺少計費設定，請前往
[platform.openai.com/account/billing](https://platform.openai.com/account/billing)
為即時認證資訊所屬的組織儲值 Platform 點數。即時語音接受由
`openclaw onboard --auth-choice openai-api-key` 建立的 `openai` API 金鑰驗證設定檔、透過
`talk.realtime.providers.openai.apiKey` 為控制介面對話設定的 Platform API 金鑰、
為語音通話設定的 `plugins.entries.voice-call.config.realtime.providers.openai.apiKey`，
或 `OPENAI_API_KEY` 環境變數。

在控制介面視訊對話中，OpenAI WebRTC 會依需求接收攝影機內容：
當模型呼叫 `describe_view` 時，瀏覽器會透過即時資料通道傳送一張有大小限制的 JPEG。
OpenClaw 不會將連續的攝影機軌道附加至 OpenAI 工作階段。
</Note>

## 記憶嵌入

OpenClaw 可使用 OpenAI 或 OpenAI 相容的嵌入端點，為
`memory_search` 建立索引及產生查詢嵌入：

```json5
{
  memory: {
    search: {
      provider: "openai",
      model: "text-embedding-3-small",
    },
  },
}
```

對於需要非對稱嵌入標籤的 OpenAI 相容端點，請在 `memory.search` 下設定
`queryInputType` 和 `documentInputType`。OpenClaw
會將其轉送為供應商特定的 `input_type` 請求欄位：查詢
嵌入使用 `queryInputType`；已建立索引的記憶區塊和批次索引則使用
`documentInputType`。完整範例請參閱
[記憶設定參考](/zh-TW/reference/memory-config#provider-specific-config)。

## 開始使用

<Tabs>
  <Tab title="API 金鑰（OpenAI Platform）">
    **最適合：**直接存取 API 並依使用量計費。

    <Steps>
      <Step title="取得 API 金鑰">
        從 [OpenAI Platform 儀表板](https://platform.openai.com/api-keys)建立或複製 API 金鑰。
      </Step>
      <Step title="執行初始設定">
        ```bash
        openclaw onboard --auth-choice openai-api-key
        ```

        或直接傳入金鑰：

        ```bash
        openclaw onboard --openai-api-key "$OPENAI_API_KEY"
        ```
      </Step>
      <Step title="確認模型可用">
        ```bash
        openclaw models list --provider openai
        ```
      </Step>
    </Steps>

    ### 路由摘要

    | 模型參照        | 執行階段政策或路由資訊                                 | 路由                     | 驗證                              |
    | ---------------- | ------------------------------------------------------------- | ------------------------- | --------------------------------- |
    | `openai/gpt-5.6` | 未設定／`auto`、完全相符的官方 HTTPS 原生路由、無請求覆寫 | 可選取 Codex     | 按順序使用 API 金鑰驗證設定檔      |
    | `openai/gpt-5.6` | 供應商／模型 `agentRuntime.id: "openclaw"`                  | OpenClaw 內嵌執行階段 | 選取的 `openai` API 金鑰設定檔 |
    | `openai/gpt-5.5` | 明確指定供應商／模型 `agentRuntime.id`                     | 選取的代理程式執行階段    | 選取的 OpenAI API 金鑰設定檔   |
    | `openai/*`       | 自行指定的 Completions、自訂項目或請求覆寫 | OpenClaw 內嵌執行階段 | 認證資訊類型維持不變 |
    | `openai/*`       | 明文官方 HTTP 端點                  | 拒絕                 | 不傳送認證資訊             |

    <Note>
    當執行階段未設定或為 `auto` 時，只有符合資格且完全相符的官方 HTTPS 原生
    路由可隱含選取 Codex app-server 執行框架。若要對代理程式模型使用 API 金鑰驗證，
    請建立 `openai` API 金鑰驗證設定檔，並使用
    `auth.order.openai` 排定其順序；`OPENAI_API_KEY` 仍是非代理程式
    OpenAI API 介面的直接備援。執行 `openclaw doctor --fix` 以遷移較舊的
    舊版 Codex 驗證順序項目。
    </Note>

    ### 設定範例

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
    }
    ```

    不含修飾詞的直接 API `gpt-5.6` ID 會解析為 Sol 層級。如果此 API
    組織未提供 GPT-5.6，請將主要模型明確設定為
    `openai/gpt-5.5`。

    若要透過 OpenAI API 試用 ChatGPT 目前的 Instant 模型，請將模型
    設定為 `openai/chat-latest`：

    ```json5
    {
      env: { OPENAI_API_KEY: "example-openai-key-not-real" },
      agents: { defaults: { model: { primary: "openai/chat-latest" } } },
    }
    ```

    `chat-latest` 是會變動的別名。全新的 OpenAI API 金鑰設定改用
    `openai/gpt-5.6`，其不含修飾詞的直接 API ID 會解析為 Sol。現有的
    明確主要模型（包括 `openai/gpt-5.5`）維持不變。
    `chat-latest` 別名僅接受 `medium` 文字詳細程度；對此模型，
    OpenClaw 會將其他任何要求的詳細程度強制設為 `medium`。

    <Warning>
    OpenClaw **不會**在直接 OpenAI API 金鑰路由上提供 `gpt-5.3-codex-spark`。
    只有當你登入的帳戶提供該模型時，才能透過 Codex 訂閱目錄項目使用。
    </Warning>

  </Tab>

  <Tab title="Codex 訂閱">
    **最適合：**使用你的 ChatGPT／Codex 訂閱，以原生 Codex
    app-server 執行取代個別 API 金鑰。Codex 雲端服務需要
    登入 ChatGPT。

    <Steps>
      <Step title="執行 Codex OAuth">
        ```bash
        openclaw onboard --auth-choice openai
        ```

        或直接執行 OAuth：

        ```bash
        openclaw models auth login --provider openai
        ```

        對於無頭或不便接收回呼的設定，請加入 `--device-code`，改用
        ChatGPT 裝置代碼流程登入，而非使用 localhost 瀏覽器
        回呼：

        ```bash
        openclaw models auth login --provider openai --device-code
        ```
      </Step>
      <Step title="使用標準 OpenAI 模型路由">
        ```bash
        openclaw config set agents.defaults.model.primary openai/gpt-5.6-sol
        ```

        此完全相符的官方 HTTPS 原生路由不需要執行階段設定。
        它可能會自動選取 Codex app-server 執行階段，而當選取該執行階段時，
        OpenClaw 會安裝或修復內建的 Codex 外掛。
      </Step>
      <Step title="確認 Codex 驗證可用">
        ```bash
        openclaw models list --provider openai
        ```

        閘道啟動後，在聊天中傳送 `/codex status` 或 `/codex models`
        以確認原生 app-server 執行階段。
      </Step>
    </Steps>

    ### 路由摘要

    | 模型參照                | 執行階段政策或路由資訊                                 | 路由                                                    | 驗證                                               |
    | ------------------------ | ------------------------------------------------------------- | -------------------------------------------------------- | -------------------------------------------------- |
    | `openai/gpt-5.6-sol`     | 未設定／`auto`、完全相符的官方 HTTPS 原生路由、無請求覆寫 | 可選取 Codex                                    | Codex 登入，或按順序使用 `openai` 驗證設定檔 |
    | `openai/gpt-5.6-terra`   | 未設定／`auto`、完全相符的官方 HTTPS 原生路由、無請求覆寫 | 可選取 Codex                                    | 當目錄提供 Terra 時使用 Codex 登入       |
    | `openai/gpt-5.6-luna`    | 未設定／`auto`、完全相符的官方 HTTPS 原生路由、無請求覆寫 | 可選取 Codex                                    | 當目錄提供 Luna 時使用 Codex 登入        |
    | `openai/gpt-5.6-sol`     | 供應商／模型 `agentRuntime.id: "openclaw"`                  | OpenClaw 內嵌執行階段、內部 Codex 驗證傳輸 | 選取的 `openai` OAuth 設定檔                    |
    | `openai/gpt-5.5`         | 明確指定供應商／模型 `agentRuntime.id`                     | 選取的代理程式執行階段                                   | 選取的 OpenAI 驗證設定檔                       |
    | `openai/*`               | 自行指定的 Completions、自訂項目或請求覆寫 | OpenClaw 內嵌執行階段                                | 認證資訊要求仍依路由而異      |
    | `openai/*`               | 明文官方 HTTP 端點                  | 拒絕                                                 | 不傳送認證資訊                              |
    | 舊版 Codex GPT-5.5 參照 | 由 doctor 修復                                            | 改寫為 `openai/gpt-5.5`                            | 已遷移的 OpenAI OAuth 設定檔                      |
    | `codex-cli/gpt-5.5`      | 由 doctor 修復                                            | 改寫為 `openai/gpt-5.5`                            | Codex app-server 驗證                              |

    <Warning>
    全新的訂閱支援設定會使用確切的 `openai/gpt-5.6-sol`；
    原生 Codex 目錄也可能公開確切的 Terra 或 Luna 參照。如果
    帳戶未提供 GPT-5.6，請明確選取 `openai/gpt-5.5`。較舊的
    Codex GPT 參照是舊版 OpenClaw 路由，而不是原生 Codex 執行階段
    路徑；執行 `openclaw doctor --fix` 即可遷移它們，而不會升級
    現有明確選取的 GPT-5.5。`gpt-5.3-codex-spark` 仍僅限於
    Codex 訂閱目錄宣告支援該模型的帳戶；其直接 OpenAI
    API 金鑰與 Azure 參照仍會被隱藏。
    </Warning>

    <Note>
    新設定應將 OpenAI 代理程式驗證順序放在 `auth.order.openai` 下；
    doctor 會遷移較舊的舊版 Codex 驗證順序項目。
    </Note>

    ### 設定範例

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
    }
    ```

    若有 API 金鑰備援，請將選取的模型保留在 `openai/*` 下，並將
    驗證順序放在 `openai` 下。OpenClaw 會先嘗試訂閱，接著
    嘗試 API 金鑰，同時維持使用 Codex 測試框架：

    ```json5
    {
      plugins: { entries: { codex: { enabled: true } } },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-sol" },
        },
      },
      auth: {
        order: {
          openai: [
            "openai:user@example.com",
            "openai:api-key-backup",
          ],
        },
      },
    }
    ```

    <Note>
    新手設定不再從 `~/.codex` 匯入 OAuth 資料。請使用
    瀏覽器 OAuth（預設）或上述裝置代碼流程登入；OpenClaw 會在
    自己的代理程式驗證儲存區中管理產生的認證資訊。
    </Note>

    ### 檢查並復原 Codex OAuth 路由

    ```bash
    openclaw models status
    openclaw models auth list --provider openai
    openclaw config get agents.defaults.model --json
    openclaw config get models.providers.openai.agentRuntime --json
    ```

    若要指定代理程式，請加入 `--agent <id>`：

    ```bash
    openclaw models status --agent <id>
    openclaw models auth list --agent <id> --provider openai
    ```

    如果較舊的設定仍包含舊版 Codex GPT 參照，或存在沒有明確執行階段設定的
    過時 OpenAI 執行階段工作階段固定項目，請修復它：

    ```bash
    openclaw doctor --fix
    openclaw config validate
    ```

    如果 `models auth list --provider openai` 顯示沒有可用的設定檔，請
    重新登入：

    ```bash
    openclaw models auth login --provider openai
    openclaw models status --probe --probe-provider openai
    ```

    若要在同一代理程式中使用多個 Codex OAuth 登入，請使用 `--profile-id`，然後
    透過驗證順序或 `/model ...@<profileId>` 控制它們：

    ```bash
    openclaw models auth login --provider openai --profile-id openai:ritsuko
    openclaw models auth login --provider openai --profile-id openai:lain
    ```

    在依賴設定檔順序之前，請執行 `openclaw doctor --fix`，以遷移較舊的舊版
    OpenAI Codex 前綴設定檔 ID 與順序項目。

    ### 狀態指示器

    聊天中的 `/status` 會顯示目前工作階段啟用的模型執行階段。
    當符合資格的隱含路由或明確的供應商／模型執行階段原則選取隨附的
    Codex app-server 測試框架時，它會顯示為
    `Runtime: OpenAI Codex`。

    ### Doctor 警告

    如果設定或工作階段狀態中仍有舊版 Codex 模型參照或過時的 OpenAI
    執行階段固定項目，除非 OpenClaw 已明確設定，否則
    `openclaw doctor --fix` 會使用 Codex 執行階段將它們重寫為 `openai/*`。

    ### 上下文視窗預設值與長上下文選擇加入

    OpenClaw 將原生模型容量與啟用中的執行階段預算
    視為兩個不同的值：

    - `contextWindow` 宣告供應商的模型總視窗。
    - `contextTokens` 限制 OpenClaw 將該視窗的多少部分用於啟用中的輸入。

    ChatGPT/Codex OAuth 會遵循即時 Codex 帳戶目錄。目前的
    目錄通常會為 GPT-5.6 宣告 `272000` 個權杖的啟用中視窗。
    使用直接 API 金鑰的 GPT-5.5 與 GPT-5.6 模型也會預設為 `272000`
    `contextTokens`，即使 Platform API 公開了更大的原生
    視窗。這可讓不同驗證模式下的一般延遲、品質與成本特性保持一致。
    已設定的 `agents.defaults.contextTokens` 值可以進一步降低該預算，
    但無法將模型提高到超過其設定的 `contextTokens` 上限。

    對於使用直接 API 金鑰的 GPT-5.5 與 GPT-5.6，OpenAI 記載的供應商視窗為
    `1050000` 個權杖，最大輸出權杖數為 `128000`。保留完整的
    輸出配額後，輸入可使用 `922000` 個權杖。這是推導出的
    運作預算，而不是供應商另行發布的輸入限制。請參閱官方
    [模型比較](https://developers.openai.com/api/docs/models/compare)
    與 [GPT-5.5 模型頁面](https://developers.openai.com/api/docs/models/gpt-5.5)。
    下列範例讓一個 Terra 模型選擇加入該配額，並要求
    OpenAI 在啟用中的權杖達到 `700000` 時進行壓縮：

    ```json5
    {
      models: {
        providers: {
          openai: {
            models: [
              {
                id: "gpt-5.6-terra",
                name: "GPT-5.6 Terra",
                contextWindow: 1050000,
                contextTokens: 922000,
                maxTokens: 128000,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "openai/gpt-5.6-terra" },
          models: {
            "openai/gpt-5.6-terra": {
              agentRuntime: { id: "openclaw" },
              params: {
                responsesServerCompaction: true,
                responsesCompactThreshold: 700000,
              },
            },
          },
        },
      },
    }
    ```

    在此範例中使用 `agentRuntime.id: "openclaw"` 是刻意的。它證明
    內嵌的 OpenClaw Responses 路徑正在使用上述模型中繼資料與伺服器端
    壓縮設定。原生 Codex 測試框架執行緒則改由 Codex 設定管理其上下文
    預算；請參閱
    [Codex 測試框架長上下文](/zh-TW/plugins/codex-harness#direct-api-long-context)。

    <Warning>
    當 GPT-5.5 或 GPT-5.6 要求超過 `272000` 個輸入權杖時，
    OpenAI 會套用更高的長上下文定價：整個符合條件的要求會按
    2× 輸入費率與 1.5× 輸出費率計費。大型提示會在多輪對話間重新傳送或
    壓縮，因此即使可見的回覆很短，選擇加入的工作階段成本仍可能遠高於
    預設值。請參閱
    [OpenAI API 定價](https://developers.openai.com/api/docs/pricing)。API
    仍是帳戶存取權、實際限制與計費的權威依據。
    </Warning>

    ### 目錄復原

    當上游 Codex 目錄中繼資料包含 `gpt-5.5` 時，OpenClaw 會使用它。
    如果帳戶已通過驗證，但即時 Codex 探索省略了 `gpt-5.5` 資料列，
    OpenClaw 會合成該 OAuth 模型資料列，使排程、子代理程式與已設定的
    預設模型執行不會因 `Unknown model` 而失敗。

  </Tab>
</Tabs>

## 原生 Codex app-server 驗證

當符合資格且完全相符的官方 HTTPS 路由隱含選取原生 Codex app-server
測試框架，或供應商／模型 `agentRuntime.id: "codex"` 明確選取它時，該框架會使用
`openai/*` 模型參照。其驗證仍以帳戶為基礎。OpenClaw 會依下列順序
選取驗證方式：

1. 代理程式的已排序 OpenAI 驗證設定檔，最好放在
   `auth.order.openai` 下。執行 `openclaw doctor --fix`，以遷移較舊的舊版
   Codex 驗證設定檔 ID 與驗證順序。
2. app-server 的現有帳戶，例如本機 Codex 命令列介面 ChatGPT
   登入。對於預設的隔離代理程式主目錄，OpenClaw 會透過其登入 RPC 將該原生
   命令列介面帳戶橋接至 app-server；它不會共用命令列介面的設定、外掛或執行緒儲存區。
3. 僅適用於本機 stdio app-server 啟動，且僅在 app-server
   回報沒有帳戶時：`CODEX_API_KEY`，接著是 `OPENAI_API_KEY`。

即使閘道程序也有用於直接 OpenAI 模型或嵌入的 `OPENAI_API_KEY`，
本機 ChatGPT/Codex 訂閱登入也不會因此被取代。環境 API 金鑰備援僅適用於
本機 stdio 無帳戶路徑；絕不會透過 WebSocket app-server 連線傳送。選取
訂閱型 Codex 設定檔時，OpenClaw 也會阻止 `CODEX_API_KEY` 與
`OPENAI_API_KEY` 進入產生的 stdio app-server 子程序，並改透過
app-server 登入 RPC 傳送選取的認證資訊。

當該訂閱設定檔因 Codex 使用量限制而受阻時，OpenClaw 會將設定檔標記為
受阻，直到 Codex 宣告的重設時間，並讓驗證順序輪替至下一個
`openai:*` 設定檔，而不會變更選取的模型或退出 Codex 測試框架。
重設時間一過，該訂閱設定檔便會再次符合使用資格。

## 圖片生成

隨附的 `openai` 外掛會透過 `image_generate` 工具註冊圖片生成。
它支援透過相同的 `openai/gpt-image-2` 模型參照，使用 OpenAI API 金鑰與
Codex OAuth 進行圖片生成。

| 功能                      | OpenAI API 金鑰                     | Codex OAuth                          |
| ------------------------- | ---------------------------------- | ------------------------------------ |
| 模型參照                  | `openai/gpt-image-2`               | `openai/gpt-image-2`                 |
| 驗證                      | `OPENAI_API_KEY`                   | OpenAI Codex OAuth 登入               |
| 傳輸方式                  | OpenAI Images API                  | Codex Responses 後端                  |
| 每次要求的圖片上限        | 4                                  | 4                                    |
| 編輯模式                  | 已啟用（最多 5 張參考圖片）         | 已啟用（最多 5 張參考圖片）           |
| 尺寸覆寫                  | 支援，包括 2K/4K 尺寸               | 支援，包括 2K/4K 尺寸                 |
| 長寬比／解析度            | 不會轉送至 OpenAI Images API        | 在安全時對應至支援的尺寸               |

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "openai/gpt-image-2" },
    },
  },
}
```

<Note>
如需共用工具參數、供應商選取與容錯移轉行為，請參閱
[圖片生成](/zh-TW/tools/image-generation)。
</Note>

`gpt-image-2` 是 OpenAI 文字生成圖片與圖片編輯的預設值。
`gpt-image-1.5`、`gpt-image-1` 與 `gpt-image-1-mini` 仍可作為
明確的模型覆寫使用。若要輸出透明背景的 PNG/WebP，請使用
`openai/gpt-image-1.5`；目前的 `gpt-image-2` API 會拒絕
`background: "transparent"`。

對於透明背景要求，請使用 `model: "openai/gpt-image-1.5"`、`outputFormat: "png"` 或
`"webp"`，以及 `background: "transparent"` 呼叫 `image_generate`；
較舊的 `openai.background` 供應商選項仍可使用。OpenClaw 也會將預設的
`openai/gpt-image-2` 透明要求重寫為 `gpt-image-1.5`，以保護公開的
OpenAI 與 OpenAI Codex OAuth 路由；Azure 與自訂 OpenAI 相容端點則會保留
其設定的部署／模型名稱。

無介面命令列介面執行也提供相同設定：

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "A simple red circle sticker on a transparent background" \
  --json
```

從輸入檔案開始時，請搭配 `openclaw infer image edit` 使用相同的
`--output-format` 與 `--background` 旗標。
`--openai-background` 仍可作為 OpenAI 專用別名使用。使用
`--quality low|medium|high|auto` 控制 OpenAI Images 的品質與成本。
使用 `--openai-moderation low|auto`，從 `image generate` 或
`image edit` 傳遞 OpenAI 的內容審核提示。

對於 ChatGPT/Codex OAuth 安裝，請維持使用相同的 `openai/gpt-image-2` ref。設定
`openai` OAuth 設定檔後，OpenClaw 會解析該已儲存的 OAuth
存取權杖，並透過 Codex Responses 後端傳送圖片請求；它
不會先嘗試 `OPENAI_API_KEY`，也不會無提示地改用 API 金鑰。
若要改用直接的 OpenAI Images API 路徑，請明確設定
`models.providers.openai`，並提供 API 金鑰、自訂基礎
URL 或 Azure 端點。如果該自訂圖片端點位於受信任的區域網路／私人位址，
也請設定 `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true`；除非提供此明確選擇，
否則 OpenClaw 會持續封鎖私人／內部的 OpenAI 相容圖片端點。

生成：

```
/tool image_generate model=openai/gpt-image-2 prompt="適用於 macOS 上 OpenClaw 的精緻發表海報" size=3840x2160 count=1
```

生成透明 PNG：

```
/tool image_generate model=openai/gpt-image-1.5 prompt="透明背景上的簡單紅色圓形貼紙" outputFormat=png background=transparent
```

編輯：

```
/tool image_generate model=openai/gpt-image-2 prompt="保留物件形狀，將材質變更為半透明玻璃" image=/path/to/reference.png size=1024x1536
```

## 影片生成

隨附的 `openai` 外掛會透過
`video_generate` 工具註冊影片生成功能。

| 功能             | 值                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------- |
| 預設模型         | `openai/sora-2`                                                                 |
| 模式             | 文字轉影片、圖片轉影片、單一影片編輯                                               |
| 參考輸入         | 1 張圖片或 1 部影片                                                                |
| 尺寸覆寫         | 文字轉影片與圖片轉影片支援此功能                                                   |
| 長寬比           | 轉換為最接近的支援尺寸，不會直接轉送原始值                                         |
| 其他覆寫         | 不支援 `resolution`、`audio`、`watermark`，會捨棄並顯示工具警告 |

OpenAI 圖片轉影片請求使用 `POST /v1/videos`，並附帶圖片
`input_reference`。單一影片編輯使用 `POST /v1/videos/edits`，並將
上傳的影片放在 `video` 欄位中。

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "openai/sora-2" },
    },
  },
}
```

<Note>
如需共用工具參數、提供者選擇與容錯移轉行為，請參閱[影片生成](/zh-TW/tools/video-generation)。

OpenAI 提供者會宣告 `supportsSize`，但不會宣告 `supportsAspectRatio` 或
`supportsResolution`。OpenClaw 的共用正規化層會先將要求的
`aspectRatio` 轉換為最接近且相符的 OpenAI `size`，再讓請求
送達提供者，因此長寬比請求通常仍可運作。
`resolution` 沒有尺寸備援值，因此會被捨棄，並以
`Ignored unsupported overrides for openai/<model>: resolution=<value>` 的形式回報給呼叫端。
</Note>

## GPT-5 提示貢獻

對於 `openai` 提供者上的 GPT-5 系列模型，OpenClaw 會加入共用的
GPT-5 提示貢獻（包括正規化為 `openai/*` 的修復前舊版 Codex ref）。
其他同樣提供 GPT-5 系列模型 ID 的提供者（例如 OpenRouter 或 opencode 路徑）
不會收到此覆寫層；其套用條件是提供者 ID `openai`，
而非僅依據模型 ID。較舊的 GPT-4.x 模型絕不會收到此覆寫層。

原生 Codex app-server 控制框架不會透過開發者指示收到角色／工具
紀律行為合約或友善互動風格覆寫層；原生 Codex 會保留由 Codex 擁有的基礎、
模型與專案文件行為，而 OpenClaw 會停用原生執行緒中 Codex 的內建個性，
以確保代理程式工作區的個性檔案維持最高優先權。
OpenClaw 僅會為原生 Codex 執行緒提供執行階段情境：頻道
傳遞、OpenClaw 動態工具、ACP 委派、工作區情境與
OpenClaw Skills。同一項貢獻中的心跳偵測指引文字是
唯一例外：原生 Codex 的心跳偵測回合確實會收到該文字，但它會以專用的
協作指示注入，而非透過共用提示貢獻
掛鉤。

GPT-5 貢獻會為符合條件且由 OpenClaw 組裝的提示加入帶標籤的行為合約，
涵蓋角色持續性、執行安全性、工具紀律、輸出形式、完成
檢查與驗證。頻道特定的回覆與靜默訊息行為仍由共用的 OpenClaw 系統
提示及對外傳遞政策負責。友善互動風格層
彼此獨立，且可進行設定。

| 值                     | 效果                         |
| ---------------------- | ---------------------------- |
| `"friendly"`（預設） | 啟用友善互動風格層           |
| `"on"`     | `"friendly"` 的別名    |
| `"off"`     | 僅停用友善風格層             |

<Tabs>
  <Tab title="設定">
    ```json5
    {
      agents: {
        defaults: {
          promptOverlays: {
            gpt5: { personality: "friendly" },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="命令列介面">
    ```bash
    openclaw config set agents.defaults.promptOverlays.gpt5.personality off
    ```
  </Tab>
</Tabs>

<Tip>
執行階段不區分值的大小寫，因此 `"Off"` 和 `"off"` 都會停用
友善風格層。
</Tip>

<Note>
當共用的
`agents.defaults.promptOverlays.gpt5.personality` 設定尚未設定時，仍會讀取舊版 `plugins.entries.openai.config.personality`
作為相容性備援。
</Note>

## 語音與語音處理

<AccordionGroup>
  <Accordion title="語音合成（TTS）">
    隨附的 `openai` 外掛會為
    `tts` 介面註冊語音合成功能。

    | 設定         | 設定路徑                                                  | 預設值                                      |
    | ------------ | --------------------------------------------------------- | ------------------------------------------- |
    | 模型         | `tts.providers.openai.model`                                        | `gpt-4o-mini-tts`                          |
    | 語音         | `tts.providers.openai.speakerVoice`                                        | `coral`                          |
    | 速度         | `tts.providers.openai.speed`                                        | （未設定）                                  |
    | 指示         | `tts.providers.openai.instructions`                                        | （未設定，僅限 `gpt-4o-mini-tts`）         |
    | 格式         | `tts.providers.openai.responseFormat`                                        | 語音留言使用 `opus`，檔案使用 `mp3` |
    | API 金鑰     | `tts.providers.openai.apiKey`                                        | 備援為 `OPENAI_API_KEY`                   |
    | 基礎 URL     | `tts.providers.openai.baseUrl`                                        | `https://api.openai.com/v1`                          |
    | 額外主體     | `tts.providers.openai.extraBody` / `extra_body`                   | （未設定）                                  |

    可用模型：`gpt-4o-mini-tts`、`tts-1`、`tts-1-hd`。可用語音：
    `alloy`、`ash`、`ballad`、`cedar`、`coral`、`echo`、`fable`、`juniper`、
    `marin`、`onyx`、`nova`、`sage`、`shimmer`、`verse`。

    OpenClaw 產生欄位後，`extraBody` 會合併至 `/audio/speech` 請求 JSON，
    因此可用於需要 `lang` 等額外金鑰的 OpenAI 相容端點。
    原型鍵會被忽略。

    ```json5
    {
      tts: {
        providers: {
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "coral" },
        },
      },
    }
    ```

    <Note>
    設定 `OPENAI_TTS_BASE_URL` 可覆寫 TTS 基礎 URL，而不影響
    聊天 API 端點。OpenAI TTS 與 Realtime 語音都透過
    OpenAI Platform API 金鑰設定；僅使用 OAuth 的安裝仍可使用
    Codex 後端聊天模型，但無法使用 OpenAI 即時語音回覆。
    </Note>

  </Accordion>

  <Accordion title="語音轉文字">
    隨附的 `openai` 外掛會透過
    OpenClaw 的媒體理解轉錄介面註冊批次語音轉文字功能。

    - 預設模型：`gpt-4o-transcribe`
    - 端點：OpenAI REST `/v1/audio/transcriptions`
    - 輸入路徑：multipart 音訊檔案上傳
    - 用於所有會從 `tools.media.audio` 讀取傳入音訊轉錄的地方，
      包括 Discord 語音頻道片段與頻道音訊附件

    若要強制使用 OpenAI 進行傳入音訊轉錄：

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "openai",
                model: "gpt-4o-transcribe",
              },
            ],
          },
        },
      },
    }
    ```

    若共用音訊媒體設定或每次呼叫的轉錄請求提供語言與提示線索，
    這些內容會轉送給 OpenAI。

  </Accordion>

  <Accordion title="即時轉錄">
    隨附的 `openai` 外掛會為
    Voice Call 外掛註冊即時轉錄功能。

    | 設定             | 設定路徑                                                        | 預設值 |
    | ---------------- | --------------------------------------------------------------- | ------ |
    | 模型             | `plugins.entries.voice-call.config.streaming.providers.openai.model`                                              | `gpt-4o-transcribe` |
    | 語言             | `...openai.language`                                              | （未設定） |
    | 提示             | `...openai.prompt`                                              | （未設定） |
    | 靜音持續時間     | `...openai.silenceDurationMs`                                              | `800` |
    | VAD 閾值         | `...openai.vadThreshold`                                              | `0.5` |
    | 驗證             | `...openai.apiKey`、`OPENAI_API_KEY` 或 `openai` API 金鑰設定檔 | 必須使用 Platform API 金鑰 |

    <Note>
    使用連至 `wss://api.openai.com/v1/realtime` 的 WebSocket 連線，並採用
    G.711 u-law（`g711_ulaw` / `audio/pcmu`）音訊。若使用 `openai` API 金鑰
    設定檔，閘道會在開啟 WebSocket 前鑄造暫時性的 Realtime 轉錄用戶端
    密鑰。此串流提供者用於 Voice Call 的即時轉錄路徑；Discord 語音目前會錄製簡短
    片段，並改用批次 `tools.media.audio` 轉錄路徑。
    </Note>

  </Accordion>

  <Accordion title="即時語音">
    隨附的 `openai` 外掛會為 Voice Call
    外掛註冊即時語音功能。

    | 設定                                    | 設定路徑                                                                     | 預設值                           |
    | --------------------------------------- | ---------------------------------------------------------------------------- | ---------------------- |
    | 模型                                    | `plugins.entries.voice-call.config.realtime.providers.openai.model`     | `gpt-realtime-2.1`  |
    | 語音                                    | `...openai.voice`                                                       | `alloy`             |
    | 溫度（Azure 部署橋接器）                | `...openai.temperature`                                                 | `0.8`               |
    | VAD 閾值                                | `...openai.vadThreshold`                                                | `0.5`                |
    | 靜音持續時間                            | `...openai.silenceDurationMs`                                           | `500`                |
    | 前置填充                                | `...openai.prefixPaddingMs`                                             | `300`                |
    | 推理強度                                | `...openai.reasoningEffort`                                             | （未設定）                       |
    | 驗證                                    | `openai` API 金鑰設定檔、`...openai.apiKey` 或 `OPENAI_API_KEY` | 需要 OpenAI Platform API 金鑰 |

    `gpt-realtime-2.1` 可用的內建即時語音：`alloy`、`ash`、
    `ballad`、`coral`、`echo`、`sage`、`shimmer`、`verse`、`marin`、`cedar`。
    OpenAI 建議使用 `marin` 和 `cedar`，以獲得最佳即時品質。這組語音
    與上述文字轉語音的語音分開；僅限 TTS 的語音（例如
    `fable`、`nova` 或 `onyx`）無法用於即時工作階段。
    若偏好規模較小、成本較低的 Realtime 2.1 變體，
    請明確將模型設為 `gpt-realtime-2.1-mini`。

    <Note>
    **GPT-Live（即將推出）。** OpenAI 的全雙工 `gpt-live-1` 和
    `gpt-live-1-mini` 模型已於 2026 年 7 月取代 ChatGPT 語音模式；
    開發者 API 正逐步開放給搶先體驗組織。OpenClaw
    可辨識此模型系列，但尚未執行它：GPT-Live 工作階段
    僅支援 WebRTC、自行管理輪流發言（不使用 VAD），並透過
    OpenClaw 即時傳輸層尚未實作的交接事件協定
    委派代理程式工作。設定 `gpt-live-*` 模型時會採取封閉式失敗，
    並針對 WebSocket 橋接器及 Talk 瀏覽器工作階段提供指引，
    而不會在代理程式無法存取的情況下默默連接音訊。在搶先體驗期間，
    API 存取權也會依 OpenAI 組織受到限制。在 GPT-Live 支援推出之前，
    請繼續使用 `gpt-realtime-2.1`（預設值）。
    </Note>

    <Note>
    後端 OpenAI 即時橋接器使用正式推出的即時 WebSocket 工作階段
    格式，該格式不接受 `session.temperature`。Azure OpenAI
    部署仍可透過 `azureEndpoint` 和 `azureDeployment` 使用，並
    保留與部署相容的工作階段格式（包括 `temperature`）。
    支援雙向工具呼叫和 G.711 u-law 音訊。
    </Note>

    <Note>
    建立工作階段時會選定即時語音。OpenAI 允許稍後變更大多數
    工作階段欄位，但模型在該工作階段輸出音訊後，
    就無法再變更語音。OpenClaw 目前將
    內建即時語音 ID 公開為字串。
    </Note>

    <Note>
    Control UI Talk 使用 OpenAI 瀏覽器即時工作階段，由閘道
    簽發暫時性用戶端密鑰，並由瀏覽器直接與
    OpenAI Realtime API 交換 WebRTC SDP。閘道使用
    所選的 `openai` 認證資訊簽發該用戶端密鑰。已設定的金鑰、API 金鑰設定檔和
    `OPENAI_API_KEY` 優先；`openai` OAuth 設定檔或外部
    Codex 登入則作為備援。閘道轉送和 Voice Call 後端即時
    WebSocket 橋接器對原生 OpenAI 端點使用相同的認證資訊順序。
    維護者可使用
    `OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts`
    進行即時驗證；OpenAI 測試階段會同時驗證後端 WebSocket 橋接器和瀏覽器
    WebRTC SDP 交換，且不記錄密鑰。
    傳入 `--openai-only`，即可在沒有 Google 認證資訊的情況下執行這兩個測試階段。
    </Note>

  </Accordion>
</AccordionGroup>

## Azure OpenAI 端點

內建的 `openai` 提供者可透過覆寫基底 URL，將影像
生成目標設為 Azure OpenAI 資源。在影像生成路徑上，OpenClaw
會偵測 `models.providers.openai.baseUrl` 上的 Azure 主機名稱，並自動切換為
Azure 的請求格式。

<Note>
即時語音使用獨立的設定路徑
（`plugins.entries.voice-call.config.realtime.providers.openai.azureEndpoint`），
不受 `models.providers.openai.baseUrl` 影響。請參閱[語音與語音功能](#voice-and-speech)下方的 **即時
語音** 手風琴，了解其 Azure 設定。
</Note>

適合使用 Azure OpenAI 的情況：

- 你已擁有 Azure OpenAI 訂閱、配額或企業
  合約
- 你需要 Azure 提供的區域資料落地或合規控制
- 你希望流量留在現有的 Azure 租用戶內

### 設定

若要透過內建的 `openai` 提供者使用 Azure 影像生成，請將
`models.providers.openai.baseUrl` 指向你的 Azure 資源，並將 `apiKey` 設為
Azure OpenAI 金鑰（而非 OpenAI Platform 金鑰）：

```json5
{
  models: {
    providers: {
      openai: {
        baseUrl: "https://<your-resource>.openai.azure.com",
        apiKey: "<azure-openai-api-key>",
      },
    },
  },
}
```

OpenClaw 可辨識下列 Azure 主機尾碼，並用於 Azure 影像生成
路由：

- `*.openai.azure.com`
- `*.services.ai.azure.com`
- `*.cognitiveservices.azure.com`

對已辨識 Azure 主機上的影像生成請求，OpenClaw 會：

- 傳送 `api-key` 標頭，而非 `Authorization: Bearer`
- 使用部署範圍路徑（`/openai/deployments/{deployment}/...`）
- 在每個請求附加 `?api-version=...`
- Azure 影像生成呼叫的預設請求逾時時間為 600s。
  各次呼叫的 `timeoutMs` 值仍會覆寫此預設值。

其他基底 URL（公開 OpenAI、OpenAI 相容 Proxy）會維持標準
OpenAI 影像請求格式。

<Note>
`openai` 提供者影像生成路徑的 Azure 路由需要
OpenClaw 2026.4.22 或更新版本。較早版本會將任何自訂
`openai.baseUrl` 視為公開 OpenAI 端點，因此無法搭配 Azure 影像
部署使用。
</Note>

### API 版本

設定 `AZURE_OPENAI_API_VERSION`，即可為 Azure 影像生成路徑
鎖定特定 Azure 預覽版或正式版版本：

```bash
export AZURE_OPENAI_API_VERSION="2024-12-01-preview"
```

未設定該變數時，預設值為 `2024-12-01-preview`。

### 模型名稱即部署名稱

Azure OpenAI 會將模型繫結至部署。對透過內建
`openai` 提供者路由的 Azure 影像生成請求，OpenClaw 中的 `model` 欄位
必須是你在 Azure 入口網站設定的 **Azure 部署名稱**，而非
公開 OpenAI 模型 ID。

如果你建立名為 `gpt-image-2-prod`、提供 `gpt-image-2` 的部署：

```
/tool image_generate model=openai/gpt-image-2-prod prompt="乾淨的海報" size=1024x1024 count=1
```

相同的部署名稱規則適用於透過
內建 `openai` 提供者路由的任何影像生成呼叫。

### 區域可用性

Azure 影像生成目前僅在部分區域提供
（例如 `eastus2`、`swedencentral`、`polandcentral`、`westus3`、
`uaenorth`）。建立部署前，請查看 Microsoft 最新的區域清單，
並確認你的區域提供該特定模型。

### 參數差異

Azure OpenAI 和公開 OpenAI 不一定接受相同的影像參數。
Azure 可能拒絕公開 OpenAI 允許的選項（例如 `gpt-image-2` 上的某些
`background` 值），或僅在特定模型版本上提供這些選項。
這些差異來自 Azure 和底層模型，而非 OpenClaw。
如果 Azure 請求因驗證錯誤而失敗，請在 Azure 入口網站中查看
你的特定部署和 API 版本所支援的參數集。

<Note>
Azure OpenAI 使用原生傳輸和相容行為，但不會收到
OpenClaw 的隱藏歸因標頭——請參閱[進階設定](#advanced-configuration)下方的 **原生與 OpenAI 相容
路由** 手風琴。

對於 Azure 上的聊天或 Responses 流量（影像生成以外），請使用
初始設定流程或專用的 Azure 提供者設定；僅有 `openai.baseUrl`
不會套用 Azure API／驗證格式。此外另有
`azure-openai-responses/*` 提供者；請參閱下方的伺服器端壓縮
手風琴。
</Note>

## 進階設定

下方各模型的 `params` 範例會塑造 OpenClaw 的內嵌提供者
請求。設定這些參數屬於明確編寫的請求行為，因此原本符合資格的
`auto` 路由會留在 OpenClaw 上，而不會隱含選取 Codex。原生
Codex app-server 控制框架擁有自己的傳輸和請求設定；若有效路由
未宣告為與 Codex 相容，明確的 `agentRuntime.id: "codex"` 會採取封閉式失敗。

<AccordionGroup>
  <Accordion title="傳輸（WebSocket 與 SSE）">
    OpenClaw 對 `openai/*` 優先使用 WebSocket，並以 SSE 作為備援（`"auto"`）。

    在 `"auto"` 模式中，OpenClaw 會：
    - 在回退至 SSE 前，重試一次早期 WebSocket 失敗
    - 失敗後，將 WebSocket 標記為降級 60 秒，並在
      冷卻期間使用 SSE
    - 附加穩定的工作階段與回合識別標頭，以供重試和
      重新連線使用
    - 在不同傳輸變體間正規化用量計數器（`input_tokens`／`prompt_tokens`）

    | 值                     | 行為                              |
    | ---------------------- | ------------------------------------ |
    | `"auto"`（預設）   | 優先使用 WebSocket，SSE 備援     |
    | `"sse"`              | 強制僅使用 SSE                    |
    | `"websocket"`        | 強制僅使用 WebSocket              |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: { transport: "auto" },
            },
          },
        },
      },
    }
    ```

    相關 OpenAI 文件：
    - [透過 WebSocket 使用 Realtime API](https://platform.openai.com/docs/guides/realtime-websocket)
    - [串流 API 回應（SSE）](https://platform.openai.com/docs/guides/streaming-responses)

  </Accordion>

  <Accordion title="快速模式">
    OpenClaw 為 `openai/*` 提供共用的快速模式切換開關：

    - **聊天／UI：** `/fast status|auto|on|off`
    - **設定：** `agents.defaults.models["<provider>/<model>"].params.fastMode`

    啟用時，OpenClaw 會將快速模式對應至 OpenAI 優先處理
    （`service_tier = "priority"`）。現有的 `service_tier` 值會
    保留，且快速模式不會重寫 `reasoning` 或
    `text.verbosity`。`fastMode: "auto"` 會讓新的模型呼叫在自動截止時間前使用快速模式，
    之後的重試、備援、工具結果或接續呼叫則不使用快速模式。
    截止時間預設為 60 秒；在作用中的模型上設定 `params.fastAutoOnSeconds`
    即可變更。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { fastMode: "auto", fastAutoOnSeconds: 30 } },
          },
        },
      },
    }
    ```

    <Note>
    工作階段覆寫優先於設定。在 Sessions UI 中清除工作階段覆寫，
    即可讓工作階段恢復使用設定的預設值。
    </Note>

  </Accordion>

  <Accordion title="優先處理（service_tier）">
    OpenAI 的 API 透過 `service_tier` 提供優先處理。請在 OpenClaw 中為各個
    模型設定：

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": { params: { serviceTier: "priority" } },
          },
        },
      },
    }
    ```

    支援的值：`auto`、`default`、`flex`、`priority`。

    <Warning>
    `serviceTier` 僅會轉送至原生 OpenAI 端點
    （`api.openai.com`）與原生 Codex 端點（`chatgpt.com/backend-api`）。
    如果你透過 Proxy 路由任一提供者，OpenClaw 會讓
    `service_tier` 保持不變。
    </Warning>

  </Accordion>

  <Accordion title="伺服器端壓縮（Responses API）">
    對於直接 OpenAI Responses 模型（`openai/*` 位於 `api.openai.com`），
    OpenAI 外掛的 OpenClaw 串流包裝器會自動啟用伺服器端
    壓縮：

    - 強制使用 `store: true`（除非模型相容性設定了 `supportsStore: false`）
    - 注入 `context_management: [{ type: "compaction", compact_threshold: ... }]`
    - 預設 `compact_threshold`：`contextWindow` 的 70%（無法取得時則為
      `80000`）

    這適用於內建的 OpenClaw 執行階段路徑，以及嵌入式執行所使用的 OpenAI 提供者
    掛鉤。原生 Codex 應用程式伺服器測試框架會透過 Codex 管理
    自己的上下文，因此不受此設定影響。

    <Tabs>
      <Tab title="明確啟用">
        適用於 Azure OpenAI Responses 等相容端點：

        ```json5
        {
          agents: {
            defaults: {
              models: {
                "azure-openai-responses/gpt-5.5": {
                  params: { responsesServerCompaction: true },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="自訂閾值">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: {
                    responsesServerCompaction: true,
                    responsesCompactThreshold: 120000,
                  },
                },
              },
            },
          },
        }
        ```
      </Tab>
      <Tab title="停用">
        ```json5
        {
          agents: {
            defaults: {
              models: {
                "openai/gpt-5.5": {
                  params: { responsesServerCompaction: false },
                },
              },
            },
          },
        }
        ```
      </Tab>
    </Tabs>

    <Note>
    `responsesServerCompaction` 僅控制 `context_management` 的注入。
    直接 OpenAI Responses 模型仍會強制使用 `store: true`，除非相容性設定了
    `supportsStore: false`。
    </Note>

  </Accordion>

  <Accordion title="嚴格代理式 GPT 模式">
    對於透過 OpenClaw 嵌入式執行階段執行的 `openai` 提供者 GPT-5 系列模型，
    OpenClaw 已預設採用名為 `strict-agentic` 的較嚴格執行合約。
    只要解析後的提供者為 `openai`，且模型 ID 符合 GPT-5 系列，
    就會自動啟用，除非設定明確選擇退出：

    ```json5
    {
      agents: {
        defaults: {
          embeddedAgent: { executionContract: "default" },
        },
      },
    }
    ```

    在支援的路徑上明確設定 `"strict-agentic"` 不會產生任何效果（它
    已是預設值），而在不支援的提供者／模型組合上也不會起作用。

    啟用 `strict-agentic` 時，OpenClaw 會：
    - 針對大量工作自動啟用 `update_plan`
    - 針對結構上為空或僅含推理的回合，以提供可見答案的
      延續回合重試
    - 當所選測試框架提供明確的測試框架計畫事件時，
      使用這些事件

    OpenClaw 不會透過分類助理文字，判斷某個回合是
    計畫、進度更新或最終答案。

    <Note>
    此合約完全存在於 OpenClaw 的嵌入式代理執行器中。它不
    適用於原生 Codex 應用程式伺服器測試框架；該框架會自行管理
    回合與計畫行為。對於原生 Codex 執行而言，測試框架的選擇比
    執行合約設定更重要。
    </Note>

  </Accordion>

  <Accordion title="原生路由與 OpenAI 相容路由">
    OpenClaw 對待直接 OpenAI、Codex 與 Azure OpenAI 端點的方式，
    不同於通用的 OpenAI 相容 `/v1` Proxy：

    **原生路由**（`openai/*`、Azure OpenAI）：
    - 僅針對支援 OpenAI `none` 強度的模型保留
      `reasoning: { effort: "none" }`
    - 對於拒絕 `reasoning.effort: "none"` 的模型或 Proxy，
      省略已停用的推理
    - 工具結構描述預設採用嚴格模式
    - 僅在已驗證的原生主機上附加隱藏的歸屬標頭（Azure
      OpenAI 即使屬於原生路由，也不會取得這些標頭）
    - 保留僅限 OpenAI 的請求塑形（`service_tier`、`store`、
      推理相容性、提示快取提示）

    **Proxy／相容路由：**
    - 使用較寬鬆的相容行為
    - 從非原生 `openai-completions` 承載資料中移除 Completions `store`
    - 接受針對 OpenAI 相容 Completions Proxy 的進階
      `params.extra_body`/`params.extraBody` 直通 JSON
    - 接受 OpenAI 相容 Completions Proxy（例如 vLLM）的
      `params.chat_template_kwargs`
    - 不強制使用嚴格工具結構描述或僅限原生路由的標頭

  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="圖片生成" href="/zh-TW/tools/image-generation" icon="image">
    共用的圖片工具參數與提供者選擇。
  </Card>
  <Card title="影片生成" href="/zh-TW/tools/video-generation" icon="video">
    共用的影片工具參數與提供者選擇。
  </Card>
  <Card title="OAuth 與驗證" href="/zh-TW/gateway/authentication" icon="key">
    驗證詳細資訊與認證資訊重複使用規則。
  </Card>
</CardGroup>
