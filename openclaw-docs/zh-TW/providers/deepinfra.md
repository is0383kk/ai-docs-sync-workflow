---
read_when:
    - 你想用單一 API 金鑰存取頂尖的開放原始碼大型語言模型
    - 你想要在 OpenClaw 中透過 DeepInfra 的 API 執行模型
summary: 使用 DeepInfra 的統一 API，在 OpenClaw 中存取最受歡迎的開放原始碼與前沿模型
title: DeepInfra
x-i18n:
    generated_at: "2026-07-26T08:02:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra 透過單一相容於 OpenAI 的端點和 API 金鑰，將請求路由至熱門的開放原始碼與前沿模型。大多數 OpenAI SDK 只要切換基礎 URL 即可與其搭配使用。

## 安裝外掛

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## 取得 API 金鑰

1. 在 [deepinfra.com](https://deepinfra.com/) 登入
2. 前往 Dashboard / Keys 並產生金鑰，或使用自動建立的金鑰

## 命令列介面設定

```bash
openclaw onboard --deepinfra-api-key <key>
```

或設定環境變數：

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## 設定片段

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## 支援的介面

設定 `DEEPINFRA_API_KEY` 後，聊天、圖片生成與影片生成會即時從 `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta`
重新整理其模型目錄。即時探索會擴充可選模型清單；各介面的預設模型仍維持下方的靜態值。其他介面在移至相同的即時目錄前，會繼續使用靜態目錄。

| 介面                     | 預設模型                                                                       | OpenClaw 設定／工具                                    |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| 聊天／模型提供者         | `deepseek-ai/DeepSeek-V4-Flash`（即時目錄會加入更多聊天模型）                               | `agents.defaults.model`                                    |
| 圖片生成／編輯           | `black-forest-labs/FLUX-1-schnell`（即時目錄會加入更多 `image-gen` 模型）               | `image_generate`、`agents.defaults.mediaModels.image`                |
| 媒體理解                 | 圖片使用 `moonshotai/Kimi-K2.5`                                                    | 傳入圖片理解                                          |
| 語音轉文字               | `openai/whisper-large-v3-turbo`                                                             | 傳入音訊轉錄                                          |
| 文字轉語音               | `hexgrad/Kokoro-82M`                                                             | `tts.provider: "deepinfra"`                                    |
| 影片生成                 | `Pixverse/Pixverse-T2V`（即時目錄會加入更多 `video-gen` 模型）               | `video_generate`、`agents.defaults.mediaModels.video`                |
| 記憶嵌入                 | `BAAI/bge-m3`                                                             | `memory.search.provider: "deepinfra"`                                    |

DeepInfra 也提供重新排序、分類、物件偵測及其他原生模型類型。OpenClaw 尚未針對這些類別提供供應商合約，因此此外掛不會註冊它們。

## 可用模型

設定金鑰後，OpenClaw 會動態探索 DeepInfra 模型。使用 `/models deepinfra` 或 `openclaw models list --provider deepinfra` 查看目前清單。

[deepinfra.com](https://deepinfra.com/) 上的任何模型皆可搭配 `deepinfra/` 前綴使用：

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
……以及更多模型
```

## 注意事項

- 模型參照為 `deepinfra/<provider>/<model>`（例如 `deepinfra/Qwen/Qwen3-Max`）。
- 預設聊天模型：`deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- 基礎 URL：`https://api.deepinfra.com/v1/openai`
- 影片生成使用相容於 OpenAI 的非同步端點 `https://api.deepinfra.com/v1/openai/videos`（先提交，再輪詢）。設定的 `baseUrl` 會予以採用。`openclaw doctor --fix` 會在 `api.deepinfra.com` 上自動將舊版 `nativeBaseUrl` 或 `/v1/inference` 值移轉至 `baseUrl`；自訂原生端點已停用，doctor 會顯示通知，且需要手動設定相容於 OpenAI 的 `baseUrl`。只要 `baseUrl` 仍指向已停用的 `/v1/inference` 介面，影片生成就會在傳送任何請求前失敗，並顯示可採取行動的錯誤訊息。

## 相關內容

- [模型提供者](/zh-TW/concepts/model-providers)
- [所有提供者](/zh-TW/providers/index)
