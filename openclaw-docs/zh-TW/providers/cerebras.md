---
read_when:
    - 你想在 OpenClaw 中使用 Cerebras
    - 你需要 Cerebras API 金鑰環境變數或命令列介面驗證選項
summary: Cerebras 設定（驗證 + 模型選擇）
title: Cerebras
x-i18n:
    generated_at: "2026-07-26T08:08:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 716eef83155ef80d9aa61bd55ed83e3e38ad22720ae055bce7eb9c2cbfb6cf41
    source_path: providers/cerebras.md
    workflow: 16
---

[Cerebras](https://www.cerebras.ai) 在自訂推論硬體上提供高速、與 OpenAI 相容的推論服務。此外掛隨附靜態的雙模型目錄（不支援即時探索）。

| 屬性            | 值                                                        |
| --------------- | --------------------------------------------------------- |
| 提供者 ID       | `cerebras`                                        |
| 外掛            | 官方外部套件（`@openclaw/cerebras-provider`）                        |
| 驗證環境變數    | `CEREBRAS_API_KEY`                                        |
| 初始設定旗標    | `--auth-choice cerebras-api-key`                                        |
| 直接命令列旗標  | `--cerebras-api-key <key>`                                        |
| API             | 與 OpenAI 相容（`openai-completions`）                      |
| 基礎 URL        | `https://api.cerebras.ai/v1`                                        |
| 預設模型        | `cerebras/zai-glm-4.7`                                        |

## 安裝外掛

```bash
openclaw plugins install @openclaw/cerebras-provider
openclaw gateway restart
```

## 開始使用

<Steps>
  <Step title="取得 API 金鑰">
    在 [Cerebras Cloud Console](https://cloud.cerebras.ai) 中建立 API 金鑰。
  </Step>
  <Step title="執行初始設定">
    <CodeGroup>

```bash 初始設定
openclaw onboard --auth-choice cerebras-api-key
```

```bash 直接旗標
openclaw onboard --non-interactive \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

```bash 僅使用環境變數
export CEREBRAS_API_KEY=csk-...
```

    </CodeGroup>

  </Step>
  <Step title="確認模型可用">
    ```bash
    openclaw models list --provider cerebras
    ```

    此命令會列出兩個靜態模型。如果 `CEREBRAS_API_KEY` 無法解析，`openclaw models status --json` 會在 `auth.unusableProfiles` 下回報缺少的認證資訊。

  </Step>
</Steps>

## 非互動式設定

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

## 內建目錄

兩個模型都共用 128k 上下文視窗，且輸出 Token 上限為 8,192。

| 模型參照                | 名稱         | 推理 | 備註                     |
| ----------------------- | ------------ | ---- | ------------------------ |
| `cerebras/zai-glm-4.7`      | Z.ai GLM 4.7 | 是   | 預設模型；預覽版推理模型 |
| `cerebras/gpt-oss-120b`      | GPT OSS 120B | 是   | 正式環境推理模型         |

## 手動設定

大多數設定只需要 API 金鑰。若要覆寫模型中繼資料，或在 `mode: "merge"` 中使用靜態目錄執行，請使用明確的 `models.providers.cerebras` 設定：

```json5
{
  env: { CEREBRAS_API_KEY: "csk-..." },
  agents: {
    defaults: {
      model: { primary: "cerebras/zai-glm-4.7" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "Z.ai GLM 4.7" },
          { id: "gpt-oss-120b", name: "GPT OSS 120B" },
        ],
      },
    },
  },
}
```

<Note>
如果閘道以常駐程式執行（launchd、systemd、Docker），請確保該程序可使用 `CEREBRAS_API_KEY`，例如將其設於 `~/.openclaw/.env` 中，或透過 `env.shellEnv` 提供。除非另行匯入環境變數，否則僅在互動式 Shell 中匯出的金鑰無法供受管理的服務使用。
</Note>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型提供者" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照與容錯移轉行為。
  </Card>
  <Card title="思考模式" href="/zh-TW/tools/thinking" icon="brain">
    兩個具備推理能力的 Cerebras 模型所支援的推理強度層級。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/config-agents#agent-defaults" icon="gear">
    代理程式預設值與模型設定。
  </Card>
  <Card title="模型常見問題" href="/zh-TW/help/faq-models" icon="circle-question">
    驗證設定檔、切換模型，以及解決「無設定檔」錯誤。
  </Card>
</CardGroup>
