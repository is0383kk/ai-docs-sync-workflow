---
read_when:
    - 你想搭配 OpenClaw 使用本機 ComfyUI 工作流程
    - 你想要搭配影像、影片或音樂工作流程使用 Comfy Cloud
    - 你需要內建 comfy 外掛的設定鍵
summary: 在 OpenClaw 中設定使用 ComfyUI 工作流程生成圖片、影片與音樂
title: ComfyUI
x-i18n:
    generated_at: "2026-07-26T07:53:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74150d202a422de8e0f4b2b82d5d12bd42eb46991e8ef688832208e1a2ff7793
    source_path: providers/comfy.md
    workflow: 16
---

OpenClaw 隨附一個內建的 `comfy` 外掛，用於以工作流程驅動的 ComfyUI 執行。此
外掛完全由工作流程驅動：OpenClaw 不會將通用的 `size`、
`aspectRatio`、`resolution`、`durationSeconds` 或 TTS 類型控制項對應至
你的圖形。

| 屬性         | 詳細資訊                                                                         |
| ------------ | -------------------------------------------------------------------------------- |
| 提供者       | `comfy`                                                                          |
| 模型         | `comfy/workflow`                                                                 |
| 共用工具     | `image_generate`、`video_generate`、`music_generate`                             |
| 驗證         | 本機 ComfyUI 不需要；Comfy Cloud 使用 `COMFY_API_KEY` 或 `COMFY_CLOUD_API_KEY` |
| API          | ComfyUI `/prompt` / `/history` / `/view`；Comfy Cloud `/api/*`                   |

## 支援項目

- 透過工作流程 JSON 產生及編輯影像（編輯時需上傳 1 張參考影像）
- 透過工作流程 JSON 產生影片，支援文字轉影片或影像轉影片（1 張參考影像）
- 透過共用的 `music_generate` 工具產生音樂／音訊，並可選用 1 張參考影像
- 從已設定的節點下載輸出；若未設定節點，則從所有相符的輸出節點下載

## 開始使用

選擇在自己的機器上執行 ComfyUI，或使用 Comfy Cloud。

<Tabs>
  <Tab title="本機">
    **最適合：** 在你的機器或區域網路上執行自己的 ComfyUI 執行個體。

    <Steps>
      <Step title="在本機啟動 ComfyUI">
        確認本機 ComfyUI 執行個體正在執行（預設為 `http://127.0.0.1:8188`）。
      </Step>
      <Step title="準備工作流程 JSON">
        匯出或建立 ComfyUI 工作流程 JSON 檔案。記下提示輸入節點，以及你希望 OpenClaw 讀取之輸出節點的節點 ID。
      </Step>
      <Step title="設定提供者">
        設定 `mode: "local"` 並指向你的工作流程檔案。最小影像範例：

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```
      </Step>
      <Step title="設定預設模型">
        將 OpenClaw 指向你所設定功能的 `comfy/workflow` 模型：

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="驗證">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Comfy Cloud">
    **最適合：** 在 Comfy Cloud 上執行工作流程，而不必管理本機 GPU 資源。

    <Steps>
      <Step title="取得 API 金鑰">
        在 [comfy.org](https://comfy.org) 註冊，並從你的帳戶儀表板產生 API 金鑰。
      </Step>
      <Step title="設定 API 金鑰">
        透過下列任一方式提供金鑰：

        ```bash
        # 初始設定旗標
        openclaw onboard --comfy-api-key "your-key"

        # 環境變數（常駐程式建議使用）
        export COMFY_API_KEY="your-key"

        # 替代環境變數
        export COMFY_CLOUD_API_KEY="your-key"

        # 或直接寫入設定
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```
      </Step>
      <Step title="準備工作流程 JSON">
        匯出或建立 ComfyUI 工作流程 JSON 檔案。記下提示輸入節點與輸出節點的節點 ID。
      </Step>
      <Step title="設定提供者">
        設定 `mode: "cloud"` 並指向你的工作流程檔案：

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        <Tip>
        雲端模式會將 `baseUrl` 預設為 `https://cloud.comfy.org`。只有使用自訂雲端端點時，才設定 `baseUrl`。
        </Tip>
      </Step>
      <Step title="設定預設模型">
        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="驗證">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 設定

Comfy 支援共用的頂層連線設定，以及各功能的工作流程區段（`image`、`video`、`music`）：

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### 共用鍵

| 鍵                    | 類型                   | 說明                                                                                  |
| --------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`                | `"local"` 或 `"cloud"` | 連線模式。預設為 `"local"`。                                               |
| `baseUrl`             | 字串                   | 本機模式預設為 `http://127.0.0.1:8188`，雲端模式預設為 `https://cloud.comfy.org`。 |
| `apiKey`              | 字串                   | 選用的內嵌金鑰，可替代 `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY` 環境變數。 |
| `allowPrivateNetwork` | 布林值                 | 允許在雲端模式中使用私人／區域網路 `baseUrl`，或使用本機私人 DNS FQDN。              |

<Note>
在 `local` 模式中，迴路／私人 IP 常值與 `http://comfyui:8188` 之類的單一標籤服務名稱不需要 `allowPrivateNetwork` 即可運作。看似公開的私人 DNS FQDN（例如 `https://comfy.local.example.com`）需要 `allowPrivateNetwork: true`。私人來源信任仍僅限於已設定的配置、主機名稱與連接埠；本機重新導向不能離開已設定的主機名稱，而雲端重新導向至公開 CDN 時，則會使用預設 SSRF 政策進行檢查。
</Note>

### 各功能的鍵

這些鍵適用於 `image`、`video` 或 `music` 區段內：

| 鍵                           | 必填     | 預設值   | 說明                                                                         |
| ---------------------------- | -------- | -------- | ---------------------------------------------------------------------------- |
| `workflow` 或 `workflowPath` | 是       | --       | 內嵌工作流程 JSON，或 ComfyUI 工作流程 JSON 檔案的路徑。                     |
| `promptNodeId`               | 是       | --       | 接收文字提示的節點 ID。                                                      |
| `promptInputName`            | 否       | `"text"` | 提示節點上的輸入名稱。                                                       |
| `outputNodeId`               | 否       | --       | 要讀取輸出的節點 ID。若省略，則使用所有相符的輸出節點。                     |
| `pollIntervalMs`             | 否       | `1500`   | 輪詢工作完成狀態的間隔（毫秒）。                                             |
| `timeoutMs`                  | 否       | `300000` | 工作流程執行的逾時時間（毫秒）。                                             |

`image` 與 `video` 區段也支援參考影像輸入節點：

| 鍵                    | 必填                                 | 預設值    | 說明                                                |
| --------------------- | ------------------------------------ | --------- | --------------------------------------------------- |
| `inputImageNodeId`    | 是（傳遞參考影像時）                 | --        | 接收已上傳參考影像的節點 ID。                       |
| `inputImageInputName` | 否                                   | `"image"` | 影像節點上的輸入名稱。                              |

`apiKey` 接受常值字串或[祕密參照](/zh-TW/gateway/configuration-reference#secrets)物件。

## 工作流程詳細資訊

<AccordionGroup>
  <Accordion title="影像工作流程">
    將預設影像模型設為 `comfy/workflow`：

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **參考影像編輯範例：**

    若要使用已上傳的參考影像啟用影像編輯，請將 `inputImageNodeId` 新增至影像設定：

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              image: {
                workflowPath: "./workflows/edit-api.json",
                promptNodeId: "6",
                inputImageNodeId: "7",
                inputImageInputName: "image",
                outputNodeId: "9",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="影片工作流程">
    將預設影片模型設為 `comfy/workflow`：

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    Comfy 影片工作流程可透過已設定的圖形支援文字轉影片與影像轉影片。

    <Note>
    OpenClaw 不會將輸入影片傳入 Comfy 工作流程。僅支援將文字提示與單張參考影像作為輸入。
    </Note>

  </Accordion>

  <Accordion title="音樂工作流程">
    內建外掛會為工作流程所定義的音訊或音樂輸出註冊音樂產生提供者，並透過共用的 `music_generate` 工具公開。它可接受選用的參考影像（最多 1 張）：

    ```text
    /tool music_generate prompt="帶有柔和磁帶質感的溫暖環境合成器循環"
    ```

    使用 `music` 設定區段指向你的音訊工作流程 JSON 與輸出節點。

  </Accordion>

  <Accordion title="向下相容性">
    現有的頂層影像設定（不含巢狀 `image` 區段）仍可使用：

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              workflowPath: "./workflows/flux-api.json",
              promptNodeId: "6",
              outputNodeId: "9",
            },
          },
        },
      },
    }
    ```

    OpenClaw 會將該舊版結構視為圖片工作流程設定。你不需要立即遷移，但建議新設定使用巢狀的 `image` / `video` / `music` 區段。如果你只使用圖片生成，舊版的扁平設定與新的巢狀 `image` 區段在功能上等效。

  </Accordion>

  <Accordion title="即時測試">
    隨附的外掛提供可選用的即時測試涵蓋範圍：

    ```bash
    OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
    ```

    除非已設定相符的 Comfy 工作流程區段，否則即時測試會略過個別的圖片、影片或音樂測試案例。

  </Accordion>
</AccordionGroup>

## 相關內容

<CardGroup cols={2}>
  <Card title="圖片生成" href="/zh-TW/tools/image-generation" icon="image">
    圖片生成工具的設定與使用方式。
  </Card>
  <Card title="影片生成" href="/zh-TW/tools/video-generation" icon="video">
    影片生成工具的設定與使用方式。
  </Card>
  <Card title="音樂生成" href="/zh-TW/tools/music-generation" icon="music">
    音樂與音訊生成工具的設定方式。
  </Card>
  <Card title="提供者目錄" href="/zh-TW/providers/index" icon="layers">
    所有提供者與模型參照的概覽。
  </Card>
  <Card title="設定參考" href="/zh-TW/gateway/config-agents#agent-defaults" icon="gear">
    完整的設定參考，包括代理程式預設值。
  </Card>
</CardGroup>
