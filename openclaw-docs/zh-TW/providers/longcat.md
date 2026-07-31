---
read_when:
    - 你想要搭配 OpenClaw 使用 LongCat-2.0
    - 你需要 LongCat API 金鑰或模型限制
summary: LongCat-2.0 的 LongCat API 設定
title: LongCat
x-i18n:
    generated_at: "2026-07-26T07:54:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c447f9c42e6547a69d2124debcb685c32fe59de29bfc551e18e791d9f280584
    source_path: providers/longcat.md
    workflow: 16
---

[LongCat](https://longcat.ai) 提供 LongCat-2.0 的託管 API；LongCat-2.0 是一款專為程式設計與代理型工作負載打造的推理模型。OpenClaw 為 LongCat 的 OpenAI 相容端點提供官方 `longcat` 外掛。

| 屬性       | 值                                 |
| ---------- | ---------------------------------- |
| 提供者     | `longcat`                 |
| 驗證       | `LONGCAT_API_KEY`                 |
| API        | OpenAI 相容的 Chat Completions     |
| 基礎 URL   | `https://api.longcat.chat/openai`                 |
| 模型       | `longcat/LongCat-2.0`                 |
| 上下文     | 1,048,576 個權杖                   |
| 最大輸出   | 131,072 個權杖                     |
| 輸入       | 文字                               |

## 安裝外掛

安裝官方套件，然後重新啟動閘道：

```bash
openclaw plugins install @openclaw/longcat-provider
openclaw gateway restart
```

## 開始使用

<Steps>
  <Step title="建立 API 金鑰">
    登入 [LongCat API 平台](https://longcat.chat/platform/)，並在
    [API Keys](https://longcat.chat/platform/api_keys) 頁面建立金鑰。
  </Step>
  <Step title="執行引導設定">
    ```bash
    openclaw onboard --auth-choice longcat-api-key
    ```
  </Step>
  <Step title="驗證模型">
    ```bash
    openclaw models list --provider longcat
    ```
  </Step>
</Steps>

若尚未設定主要模型，引導設定會新增託管目錄並選取 `longcat/LongCat-2.0`。

### 非互動式設定

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice longcat-api-key \
  --longcat-api-key "$LONGCAT_API_KEY"
```

## 推理行為

LongCat 提供二元思考控制。OpenClaw 會將已啟用的思考層級對應至
`thinking: { type: "enabled" }`，並將 `/think off` 對應至
`thinking: { type: "disabled" }`。LongCat 目前未記載
`reasoning_effort`，因此 OpenClaw 不會傳送該值。

LongCat 會在 `reasoning_content` 中傳回推理內容。OpenClaw 在重播助理的工具呼叫回合時會保留該欄位，讓多回合代理工作階段維持提供者預期的訊息結構。

## 定價

內建目錄採用 LongCat 的隨用隨付牌價，單位為每百萬個權杖的美元價格：未快取輸入 $0.75、已快取輸入 $0.015，以及輸出 $2.95。LongCat 可能提供限時折扣；請以[定價頁面](https://longcat.chat/platform/docs/Pricing/LongCat-2.0.html)和你的帳單記錄為準。

## 自行託管 LongCat-2.0

`longcat` 提供者以 LongCat 的託管 API 為目標。若要使用
[Hugging Face](https://huggingface.co/meituan-longcat/LongCat-2.0) 上的開放權重，請透過 OpenAI 相容執行環境提供模型服務，並改用 OpenClaw 現有的
[vLLM](/zh-TW/providers/vllm) 或 [SGLang](/zh-TW/providers/sglang) 提供者。

請在自行託管的提供者目錄中保留執行環境使用的確切模型識別碼；不要透過 `longcat/LongCat-2.0` 路由本機部署。

## 疑難排解

<AccordionGroup>
  <Accordion title="金鑰可在 Shell 中使用，但無法在閘道中使用">
    由常駐程式管理的閘道處理程序不會繼承互動式 Shell 中的所有變數。請將 `LONGCAT_API_KEY` 放入 `~/.openclaw/.env`、透過引導設定加以配置，或使用已核准的密鑰參照。
  </Accordion>

  <Accordion title="請求失敗並傳回 402 或 429">
    `402` 表示帳戶的權杖配額不足。`429` 表示 API 金鑰已達速率限制。請檢查 [LongCat 使用量](https://longcat.chat/platform/usage)，並在提供者的退避時段結束後重試受速率限制的請求。
  </Accordion>

  <Accordion title="模型未顯示">
    執行 `openclaw plugins list` 並確認 `longcat` 外掛已啟用，然後執行 `openclaw models list --provider longcat`。
  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型提供者" href="/zh-TW/concepts/model-providers" icon="layers">
    提供者配置、模型參照與容錯移轉行為。
  </Card>
  <Card title="LongCat API 文件" href="https://longcat.chat/platform/docs/" icon="arrow-up-right-from-square">
    託管 API 端點、驗證、限制與範例。
  </Card>
  <Card title="LongCat-2.0 模型卡" href="https://huggingface.co/meituan-longcat/LongCat-2.0" icon="arrow-up-right-from-square">
    架構、部署指南與模型詳細資訊。
  </Card>
  <Card title="密鑰" href="/zh-TW/gateway/secrets" icon="key">
    儲存提供者認證資訊，無須將純文字嵌入配置中。
  </Card>
</CardGroup>
