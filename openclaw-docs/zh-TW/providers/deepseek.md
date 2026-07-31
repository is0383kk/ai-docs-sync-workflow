---
read_when:
    - 你想在 OpenClaw 中使用 DeepSeek
    - 你需要 API 金鑰環境變數或命令列介面驗證選項
summary: DeepSeek 設定（驗證 + 模型選擇）
title: DeepSeek
x-i18n:
    generated_at: "2026-07-26T08:33:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e074756d593205d7d05f499da93b9bd3c63acdce7092b42fb5562023577925
    source_path: providers/deepseek.md
    workflow: 16
---

[DeepSeek](https://www.deepseek.com) 提供功能強大的 AI 模型，並支援與 OpenAI 相容的 API。

| 屬性     | 值                         |
| -------- | -------------------------- |
| 提供者   | `deepseek`         |
| 驗證     | `DEEPSEEK_API_KEY`         |
| API      | 與 OpenAI 相容             |
| 基礎 URL | `https://api.deepseek.com`         |

## 安裝外掛

安裝官方外掛，然後重新啟動閘道：

```bash
openclaw plugins install @openclaw/deepseek-provider
openclaw gateway restart
```

## 開始使用

<Steps>
  <Step title="取得 API 金鑰">
    在 [platform.deepseek.com](https://platform.deepseek.com/api_keys) 建立 API 金鑰。
  </Step>
  <Step title="執行初始設定">
    ```bash
    openclaw onboard --auth-choice deepseek-api-key
    ```

    系統會提示輸入 API 金鑰，並將 `deepseek/deepseek-v4-flash` 設為預設模型。

  </Step>
  <Step title="確認模型可用">
    ```bash
    openclaw models list --provider deepseek
    ```

    若要在未執行閘道的情況下檢視外掛的靜態目錄：

    ```bash
    openclaw models list --all --provider deepseek
    ```

  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="非互動式設定">
    對於指令碼或無頭安裝，請直接傳入所有旗標：

    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice deepseek-api-key \
      --deepseek-api-key "$DEEPSEEK_API_KEY" \
      --skip-health \
      --accept-risk
    ```

  </Accordion>
</AccordionGroup>

<Warning>
如果閘道以常駐程式（launchd/systemd）執行，請確保該程序可以使用
`DEEPSEEK_API_KEY`（例如，放在 `~/.openclaw/.env` 中或透過
`env.shellEnv` 提供）。
</Warning>

## 內建目錄

| 模型參照                     | 名稱              | 輸入 | 上下文    | 最大輸出 | 備註                                             |
| ---------------------------- | ----------------- | ---- | --------- | -------- | ------------------------------------------------ |
| `deepseek/deepseek-v4-flash` | DeepSeek V4 Flash | 文字 | 1,000,000 | 384,000  | 預設模型；支援 V4 思考功能的介面                |
| `deepseek/deepseek-v4-pro`   | DeepSeek V4 Pro   | 文字 | 1,000,000 | 384,000  | 支援 V4 思考功能的介面                          |
| `deepseek/deepseek-chat`     | DeepSeek Chat     | 文字 | 1,000,000 | 384,000  | 已淘汰的 V4 Flash 非思考模式相容名稱             |
| `deepseek/deepseek-reasoner` | DeepSeek Reasoner | 文字 | 1,000,000 | 384,000  | 已淘汰的 V4 Flash 思考模式相容名稱               |

<Warning>
DeepSeek 已於 2026 年 7 月 24 日 15:59 UTC 停用
`deepseek-chat` 和 `deepseek-reasoner`。它們目前分別路由至非思考模式和
思考模式的 DeepSeek V4 Flash。請在截止時間前將已設定的模型參照改為
`deepseek/deepseek-v4-flash` 或 `deepseek/deepseek-v4-pro`。
</Warning>

OpenClaw 的本機成本估算會依照 DeepSeek 公布的快取命中、快取未命中及輸出費率。
DeepSeek 可能變更這些費率；其
[模型與定價](https://api-docs.deepseek.com/quick_start/pricing/)頁面是計費的
權威依據。

<Tip>
V4 模型支援 DeepSeek 的 `thinking` 控制項。OpenClaw 也會在後續回合中
重新傳送 DeepSeek `reasoning_content`，讓包含工具呼叫的思考工作階段能夠繼續。
對 DeepSeek V4 模型使用 `/think xhigh` 或 `/think max`，即可要求
DeepSeek 的最高 `reasoning_effort`；兩者都會對應至 `"max"`。
</Tip>

## 思考與工具

DeepSeek V4 思考工作階段要求：在後續請求中，從啟用思考功能的回合重新傳送的
助理訊息必須包含 `reasoning_content`。OpenClaw 的 DeepSeek 外掛會自動補上該
欄位，因此即使歷史記錄來自另一個與 OpenAI 相容的提供者（沒有原生
`reasoning_content`）或純助理訊息，一般的多回合工具使用仍可在
`deepseek/deepseek-v4-flash` 和 `deepseek/deepseek-v4-pro` 上正常運作。在工作階段中途切換
提供者後，不需要 `/new`。

停用思考功能時（包括 UI 中的 **None** 選項），OpenClaw 會傳送
`thinking: { type: "disabled" }`，並從傳出的歷史記錄中移除重新傳送的
`reasoning_content`，使工作階段維持在 DeepSeek 的非思考路徑。

使用 `deepseek/deepseek-v4-flash` 可採用預設的快速路徑。若可接受較高的成本或延遲，
請使用 `deepseek/deepseek-v4-pro` 以採用功能更強的模型。

## 即時測試

若只要從新版模型即時測試套件執行 DeepSeek V4 直接模型檢查：

```bash
OPENCLAW_LIVE_PROVIDERS=deepseek \
OPENCLAW_LIVE_MODELS="deepseek/deepseek-v4-flash,deepseek/deepseek-v4-pro" \
pnpm test:live src/agents/models.profiles.live.test.ts
```

驗證兩個 V4 模型都能完成執行，且思考／工具的後續回合會保留 DeepSeek 所需的
重新傳送承載資料。

## 設定範例

```json5
{
  env: { DEEPSEEK_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "deepseek/deepseek-v4-flash" },
    },
  },
}
```

## 相關內容

<CardGroup cols={2}>
  <Card title="模型選擇" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/configuration-reference" icon="gear">
    代理程式、模型及提供者的完整設定參考。
  </Card>
</CardGroup>
