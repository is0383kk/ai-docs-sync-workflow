---
read_when:
    - 你想要搭配 OpenClaw 使用 Tencent hy3
    - 你需要設定 TokenHub 或 TokenPlan API 金鑰
summary: 騰訊雲 TokenHub 與 TokenPlan 的 hy3 設定
title: 騰訊雲端（TokenHub / TokenPlan）
x-i18n:
    generated_at: "2026-07-26T08:04:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c2ffb8ab824539c7765d38e4332c30a6dd371fdc19be825f2ad9af0197fa256
    source_path: providers/tencent.md
    workflow: 16
---

安裝官方 Tencent Cloud 提供者外掛，以透過兩個端點 — TokenHub（`tencent-tokenhub`）和 TokenPlan（`tencent-tokenplan`）— 使用 OpenAI 相容 API 存取 Tencent Hy3。

| 屬性                      | 值                                                    |
| ------------------------- | ----------------------------------------------------- |
| 提供者 ID                 | `tencent-tokenhub`、`tencent-tokenplan`               |
| 套件                      | `@openclaw/tencent-provider`                                    |
| TokenHub 驗證環境變數     | `TOKENHUB_API_KEY`                                    |
| TokenPlan 驗證環境變數    | `TOKENPLAN_API_KEY`                                    |
| TokenHub 初始設定旗標     | `--auth-choice tokenhub-api-key`                                    |
| TokenPlan 初始設定旗標    | `--auth-choice tokenplan-api-key`                                    |
| TokenHub 直接命令列旗標   | `--tokenhub-api-key <key>`                                    |
| TokenPlan 直接命令列旗標  | `--tokenplan-api-key <key>`                                    |
| API                       | OpenAI 相容（`openai-completions`）                     |
| TokenHub 基礎 URL         | `https://tokenhub.tencentmaas.com/v1`                                    |
| TokenHub 全域基礎 URL     | `https://tokenhub-intl.tencentmaas.com/v1`（覆寫）                            |
| TokenPlan 基礎 URL        | `https://api.lkeap.cloud.tencent.com/plan/v3`                                    |
| 預設模型                  | `tencent-tokenhub/hy3`                                    |

## 快速開始

<Steps>
  <Step title="建立 Tencent API 金鑰">
    為 Tencent Cloud TokenHub 和 TokenPlan 建立 API 金鑰。如果為金鑰選擇受限的存取範圍，請將 **hy3**（若計畫在 TokenHub 上使用 **hy3 preview**，也請一併加入）納入允許的模型。
  </Step>
  <Step title="執行初始設定">
    <CodeGroup>

```bash TokenHub 初始設定
openclaw onboard --auth-choice tokenhub-api-key
```

```bash TokenHub 直接旗標
openclaw onboard --non-interactive \
  --auth-choice tokenhub-api-key \
  --tokenhub-api-key "$TOKENHUB_API_KEY"
```

```bash TokenPlan 初始設定
openclaw onboard --auth-choice tokenplan-api-key
```

```bash TokenPlan 直接旗標
openclaw onboard --non-interactive \
  --auth-choice tokenplan-api-key \
  --tokenplan-api-key "$TOKENPLAN_API_KEY"
```

```bash 僅使用環境變數
export TOKENHUB_API_KEY=...
export TOKENPLAN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="驗證模型">
    ```bash
    openclaw models list --provider tencent-tokenhub
    openclaw models list --provider tencent-tokenplan
    ```
  </Step>
</Steps>

## 非互動式設定

```bash
# TokenHub
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice tokenhub-api-key \
  --tokenhub-api-key "$TOKENHUB_API_KEY" \
  --skip-health \
  --accept-risk

# TokenPlan
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice tokenplan-api-key \
  --tokenplan-api-key "$TOKENPLAN_API_KEY" \
  --skip-health \
  --accept-risk
```

<Note>
除了 `--non-interactive` 之外，也必須提供 `--accept-risk`。
</Note>

## 內建目錄

| 模型參照                       | 名稱                   | 輸入 | 上下文  | 最大輸出 | 備註       |
| ------------------------------ | ---------------------- | ---- | ------- | -------- | ---------- |
| `tencent-tokenhub/hy3-preview` | hy3 preview（TokenHub） | 文字 | 256,000 | 64,000   | 支援推理   |
| `tencent-tokenhub/hy3`         | hy3（TokenHub）         | 文字 | 256,000 | 64,000   | 支援推理   |
| `tencent-tokenplan/hy3`        | hy3（TokenPlan）        | 文字 | 256,000 | 64,000   | 支援推理   |

hy3 是 Tencent Hunyuan 的大型 MoE 語言模型，適用於推理、長上下文指令遵循、程式碼及代理工作流程。Tencent 的 OpenAI 相容範例使用 `hy3` 作為模型 ID，並支援標準聊天補全工具呼叫以及 `reasoning_effort`。

<Tip>
  模型 ID 是 `hy3`。請勿與 Tencent 的 `HY-3D-*` 模型混淆；後者是 3D 生成 API，並非此提供者所設定的 OpenClaw 聊天模型。
</Tip>

## 進階設定

<AccordionGroup>
  <Accordion title="覆寫端點">
    OpenClaw 的內建目錄使用 Tencent Cloud 的 `https://tokenhub.tencentmaas.com/v1` 端點。只有在你的 TokenHub 帳戶或區域需要不同端點時才覆寫：

    ```bash
    openclaw config set models.providers.tencent-tokenhub.baseUrl "https://your-endpoint/v1"
    ```

  </Accordion>

  <Accordion title="守護程序的環境變數可用性">
    如果閘道以受管理服務執行（launchd、systemd、Docker），該程序必須能存取 `TOKENHUB_API_KEY` 和 `TOKENPLAN_API_KEY`。請在 `~/.openclaw/.env` 中設定，或透過 `env.shellEnv` 設定，讓 launchd、systemd 或 Docker exec 環境可以讀取它們。

    <Warning>
      僅在互動式 shell 中匯出的金鑰，受管理的閘道程序無法存取。請使用環境變數檔案或設定介面，確保持續可用。
    </Warning>

  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="模型提供者" href="/zh-TW/concepts/model-providers" icon="layers">
    選擇提供者、模型參照及容錯移轉行為。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/configuration-reference" icon="gear">
    包含提供者設定的完整設定結構描述。
  </Card>
  <Card title="Tencent TokenHub" href="https://cloud.tencent.com/product/tokenhub" icon="arrow-up-right-from-square">
    Tencent Cloud 的 TokenHub 產品頁面。
  </Card>
  <Card title="Hy3 preview 模型卡" href="https://huggingface.co/tencent/Hy3-preview" icon="square-poll-horizontal">
    Tencent Hunyuan Hy3 preview 的詳細資訊與基準測試。
  </Card>
</CardGroup>
