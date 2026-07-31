---
read_when:
    - 你想要從 OpenClaw 撥打語音電話
    - 你正在設定或開發語音通話外掛
    - 你需要在電話通訊中使用即時語音或串流語音轉錄功能
sidebarTitle: Voice call
summary: 透過 Twilio、Telnyx 或 Plivo 撥打及接聽語音通話，並可選用即時語音和串流轉錄功能
title: 語音通話外掛
x-i18n:
    generated_at: "2026-07-26T08:00:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79f09f7b5cb99aace0960e283723d4f4408afa5f5dacd71f3c527fa62859f56f
    source_path: plugins/voice-call.md
    workflow: 16
---

透過外掛為 OpenClaw 提供語音通話：撥出通知、多輪
對話、全雙工即時語音、串流轉錄，以及
具有允許清單政策的來電。

**供應商：** `mock`（開發用，無網路）、`plivo`（Voice API + XML 轉接 +
GetInput 語音）、`telnyx`（Call Control v2）、`twilio`（Programmable Voice +
Media Streams）。

<Note>
語音通話外掛會在**閘道程序內部**執行。如果你使用
遠端閘道，請在執行閘道的機器上安裝並設定此外掛，
然後重新啟動閘道以載入它。
</Note>

## 快速開始

<Steps>
  <Step title="安裝外掛">
    <Tabs>
      <Tab title="從 npm 安裝">
        ```bash
        openclaw plugins install @openclaw/voice-call
        ```
      </Tab>
      <Tab title="從本機資料夾安裝（開發用）">
        ```bash
        PLUGIN_SRC=./path/to/local/voice-call-plugin
        openclaw plugins install "$PLUGIN_SRC"
        cd "$PLUGIN_SRC" && pnpm install
        ```
      </Tab>
    </Tabs>

    使用未指定版本的套件以跟隨目前的發行標籤。只有在需要可重現的安裝時，
    才固定使用確切版本。之後請重新啟動閘道，
    讓外掛載入。

  </Step>
  <Step title="設定供應商和網路鉤子">
    在 `plugins.entries.voice-call.config` 下設定組態（請參閱下方的
    [組態](#configuration)）。至少需要：`provider`、供應商
    認證資訊、`fromNumber`，以及可從公網存取的網路鉤子 URL。
  </Step>
  <Step title="驗證設定">
    ```bash
    openclaw voicecall setup
    openclaw voicecall setup --json
    ```

    檢查外掛是否啟用、供應商認證資訊、網路鉤子是否公開，
    以及是否只啟用一種音訊模式（`streaming` 或 `realtime`）。

  </Step>
  <Step title="冒煙測試">
    ```bash
    openclaw voicecall smoke
    openclaw voicecall smoke --to "+15555550123"
    ```

    兩者預設皆為試執行。加入 `--yes` 以撥打一通簡短的撥出
    通知電話：

    ```bash
    openclaw voicecall smoke --to "+15555550123" --yes
    ```

  </Step>
</Steps>

<Warning>
對於 Twilio、Telnyx 和 Plivo，設定結果必須是**公開網路鉤子 URL**。
如果 `publicUrl`、通道 URL、Tailscale URL 或 serve 備援
解析為回送位址或私人網路空間，設定會失敗，而不會
啟動無法接收電信業者網路鉤子的供應商。
</Warning>

## 組態

如果 `enabled: true`，但所選供應商缺少認證資訊，閘道
啟動時會記錄設定未完成警告，列出缺少的金鑰，並略過
啟動執行階段。命令、RPC 呼叫和代理程式工具在使用時仍會傳回
確切缺少的組態。

<Note>
語音通話認證資訊接受 SecretRef。`plugins.entries.voice-call.config.twilio.authToken`、`plugins.entries.voice-call.config.realtime.providers.*.apiKey`、`plugins.entries.voice-call.config.streaming.providers.*.apiKey` 和 `plugins.entries.voice-call.config.tts.providers.*.apiKey` 會透過標準 SecretRef 介面解析；請參閱 [SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface)。
</Note>

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // 或 "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234", // Twilio 也可使用 TWILIO_FROM_NUMBER
          toNumber: "+15550005678",
          sessionScope: "per-phone", // per-phone | per-call
          numbers: {
            "+15550009999": {
              inboundGreeting: "Silver Fox Cards，請問需要什麼協助？",
              responseSystemPrompt: "你是一位言簡意賅的棒球卡專家。",
              tts: {
                providers: {
                  openai: { speakerVoice: "alloy" },
                },
              },
            },
          },

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
            // region: "ie1", // 選用：us1 | ie1 | au1；預設為 us1
          },
          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // 來自 Mission Control Portal 的 Telnyx 網路鉤子公開金鑰
            //（Base64；也可透過 TELNYX_PUBLIC_KEY 設定）。
            publicKey: "...",
          },
          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // 網路鉤子伺服器
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // 網路鉤子安全性（建議用於通道／Proxy）
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // 公開存取（擇一）
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" },

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: { enabled: true /* 僅限 Twilio；請參閱串流轉錄 */ },
          realtime: { enabled: false /* 請參閱即時語音對話 */ },
        },
      },
    },
  },
}
```

### 組態參考

以上未顯示的 `plugins.entries.voice-call.config` 頂層金鑰：

| 金鑰                             | 預設值      | 備註                                                                                              |
| ------------------------------- | ------------ | -------------------------------------------------------------------------------------------------- |
| `enabled`                       | `false`      | 總開關。                                                                              |
| `inboundPolicy`                 | `"disabled"` | `disabled` \| `allowlist` \| `pairing` \| `open`。請參閱[來電](#inbound-calls)。             |
| `allowFrom`                     | `[]`         | `inboundPolicy: "allowlist"` 的 E.164 允許清單。                                                  |
| `maxDurationSeconds`            | `300`        | 每通電話的硬性通話時間上限，無論是否接聽都會強制執行。                                 |
| `staleCallReaperSeconds`        | `120`        | 請參閱[過期通話清理器](#stale-call-reaper)。`0` 會將其停用。                                      |
| `silenceTimeoutMs`              | `800`        | 傳統（非即時）流程的語音結束靜音偵測。                               |
| `transcriptTimeoutMs`           | `180000`     | 放棄某一輪對話前，等待來電者轉錄文字的最長時間。                                       |
| `ringTimeoutMs`                 | `30000`      | 撥出電話的響鈴逾時。                                                                   |
| `maxConcurrentCalls`            | `1`          | 超過此限制的撥出電話會遭拒絕。                                                     |
| `outbound.notifyHangupDelaySec` | `3`          | 通知模式下，在 TTS 結束後等待多少秒再自動掛斷。                                       |
| `skipSignatureVerification`     | `false`      | 僅供本機測試；切勿在正式環境中啟用。                                                    |
| `store`                         | 未設定        | 覆寫預設的 `$OPENCLAW_STATE_DIR/voice-calls` 路徑（通常為 `~/.openclaw/voice-calls`）。 |
| `agentId`                       | `"main"`     | 用於產生回應和儲存工作階段的代理程式。                                            |
| `responseModel`                 | 未設定        | 覆寫傳統（非即時）回應的預設模型。                                  |
| `responseSystemPrompt`          | 已產生    | 傳統回應的自訂系統提示詞。                                                        |
| `responseTimeoutMs`             | `30000`      | 傳統回應產生的逾時時間（毫秒）。                                                      |

Twilio 預設使用其 US1 REST 端點。若要在支援的
非美國區域處理通話，請將 `twilio.region` 設為 `ie1` 或 `au1`，並使用來自
該區域的認證資訊。請參閱
[Twilio 的非美國 REST API 指南](https://www.twilio.com/docs/global-infrastructure/using-the-twilio-rest-api-in-a-non-us-region)。

<AccordionGroup>
  <Accordion title="供應商公開存取與安全性注意事項">
    - Twilio、Telnyx 和 Plivo 都需要**可從公網存取**的網路鉤子 URL。
    - `mock` 是本機開發供應商（不會進行網路呼叫）。
    - Telnyx 需要 `telnyx.publicKey`（或 `TELNYX_PUBLIC_KEY`），除非 `skipSignatureVerification` 為 true。
    - `skipSignatureVerification` 僅供本機測試使用。
    - 在 ngrok 免費方案中，請將 `publicUrl` 設為確切的 ngrok URL；一律會強制驗證簽章。
    - 僅當 `tunnel.provider="ngrok"` 且 `serve.bind` 為回送位址（ngrok 本機代理程式）時，`tunnel.allowNgrokFreeTierLoopbackBypass: true` 才允許簽章無效的 Twilio 網路鉤子。僅供本機開發使用。
    - ngrok 免費方案的 URL 可能會變更或加入插頁行為；如果 `publicUrl` 發生偏移，Twilio 簽章驗證會失敗。正式環境：建議使用穩定的網域或 Tailscale funnel。

  </Accordion>
  <Accordion title="串流連線上限">
    - `streaming.preStartTimeoutMs`（預設為 `5000`）會關閉從未傳送有效 `start` 畫面的通訊端。
    - `streaming.maxPendingConnections`（預設為 `32`）會限制未驗證身分且尚未啟動的通訊端總數。
    - `streaming.maxPendingConnectionsPerIp`（預設為 `4`）會限制每個來源 IP 未驗證身分且尚未啟動的通訊端數量。
    - `streaming.maxConnections`（預設為 `128`）會限制所有開啟的媒體串流通訊端（待處理 + 使用中）。

  </Accordion>
  <Accordion title="舊版組態遷移">
    組態剖析會自動正規化這些舊版金鑰，並記錄
    指明替代路徑的警告；此相容層將在未來的
    版本（`2026.6.0`）中移除，因此請執行 `openclaw doctor --fix`，將已提交的
    組態重寫為標準形態：

    - `provider: "log"` → `provider: "mock"`
    - `twilio.from` → `fromNumber`
    - `streaming.sttProvider` → `streaming.provider`
    - `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
    - `streaming.sttModel` → `streaming.providers.openai.model`
    - `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
    - `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`
    - `realtime.agentContext.includeSystemPrompt` 已移除（即時內容現在使用產生的代理程式提示詞）

  </Accordion>
</AccordionGroup>

## 工作階段範圍

語音通話預設使用 `sessionScope: "per-phone"`，因此同一來電者
重複來電時會保留對話記憶。當每通電信業者電話都應以全新內容開始時，
請設定 `sessionScope: "per-call"`，例如接待、預約、IVR 或 Google Meet
橋接流程，其中同一電話號碼可能代表不同的會議。

語音通話會將產生的工作階段金鑰儲存在已設定的代理程式命名空間下
（`agent:<agentId>:voice:*`）。原始的明確整合金鑰會解析至
相同的命名空間：標準 `agent:<configuredAgentId>:*` 金鑰會保留該
擁有者，並遵循核心 `session.mainKey`/全域範圍別名；外來或
格式錯誤的 `agent:*` 輸入會在已設定的
代理程式下限定為不透明金鑰；`global` 和 `unknown` 仍是全域哨兵值。

## 即時語音對話

`realtime` 會選取用於即時通話音訊的全雙工即時語音供應商。
它與 `streaming` 分開，後者只會將音訊轉送至即時
轉錄供應商。

<Warning>
`realtime.enabled` 無法與 `streaming.enabled` 結合使用。每通電話只能選擇一種
音訊模式。
</Warning>

目前的執行階段行為：

- `realtime.enabled` 支援 Twilio 和 Telnyx。
- `realtime.provider` 為選用設定。若未設定，Voice Call 會使用第一個已註冊的即時語音提供者。
- 隨附的即時語音提供者：Google Gemini Live（`google`）和 OpenAI（`openai`），由各自的提供者外掛註冊。
- 提供者擁有的原始設定位於 `realtime.providers.<providerId>` 下。
- Voice Call 預設公開共用的 `openclaw_agent_consult` 即時工具。當來電者要求更深入的推理、最新資訊或一般 OpenClaw 工具時，即時模型可以呼叫該工具。
- `realtime.consultPolicy` 可選擇性加入即時模型應在何時呼叫 `openclaw_agent_consult` 的指引。
- `realtime.agentContext.enabled` 預設關閉。啟用時，Voice Call 會在設定工作階段時，將有範圍限制的代理程式身分及所選工作區檔案的摘要資訊注入即時提供者的指示中。
- `realtime.fastContext.enabled` 預設關閉。啟用時，Voice Call 會先在已建立索引的記憶／工作階段內容中搜尋諮詢問題，並在 `realtime.fastContext.timeoutMs` 範圍內將這些片段傳回即時模型；只有當 `realtime.fastContext.fallbackToConsult` 為 true 時，才會改用完整的諮詢代理程式。
- 如果 `realtime.provider` 指向未註冊的提供者，或完全沒有註冊任何即時語音提供者，Voice Call 會記錄警告並略過即時媒體，而不會使整個外掛失敗。
- 當 `realtime.enabled` 為 true 時，`inboundPolicy` 不得為 `"disabled"`；`validateProviderConfig` 會拒絕此組合。
- 諮詢工作階段金鑰會在可用時重複使用已儲存的通話工作階段，否則改用已設定的 `sessionScope`（預設為 `per-phone`，隔離通話則為 `per-call`）。

### 工具政策

`realtime.toolPolicy` 控制諮詢執行：

| 政策           | 行為                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | 公開諮詢工具，並將一般代理程式限制為使用 `read`、`web_search`、`web_fetch`、`x_search`、`memory_search` 和 `memory_get`。 |
| `owner`          | 公開諮詢工具，並允許一般代理程式使用正常的代理程式工具政策。                                                      |
| `none`           | 不公開諮詢工具。自訂 `realtime.tools` 仍會傳遞給即時提供者。                               |

`realtime.consultPolicy` 僅控制即時模型的指示：

| 政策        | 指引                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------- |
| `auto`        | 保留預設提示，並由提供者決定何時呼叫諮詢工具。              |
| `substantive` | 直接回答簡單的對話銜接內容；在回答事實、使用記憶、工具或內容前先進行諮詢。 |
| `always`      | 在每個實質回答前進行諮詢。                                                        |

### 代理程式語音內容

當語音橋接應呈現已設定 OpenClaw 代理程式的說話風格，而一般對話不需負擔完整代理程式諮詢往返成本時，請啟用 `realtime.agentContext`。內容摘要會在建立即時工作階段時加入一次，因此不會增加每輪對話的延遲。呼叫 `openclaw_agent_consult` 時仍會執行完整的 OpenClaw 代理程式，並應用於工具作業、最新資訊、記憶查詢或工作區狀態。

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          agentId: "main",
          realtime: {
            enabled: true,
            provider: "google",
            toolPolicy: "safe-read-only",
            consultPolicy: "substantive",
            agentContext: {
              enabled: true,
              maxChars: 6000,
              includeIdentity: true,
              includeWorkspaceFiles: true,
              files: ["SOUL.md", "IDENTITY.md", "USER.md"],
            },
          },
        },
      },
    },
  },
}
```

### 即時提供者範例

<Tabs>
  <Tab title="Google Gemini Live">
    預設值：API 金鑰來自 `realtime.providers.google.apiKey`、`GEMINI_API_KEY`
    或 `GOOGLE_API_KEY`；模型為 `gemini-3.1-flash-live-preview`；
    語音為 `Kore`。`sessionResumption` 和 `contextWindowCompression` 預設開啟，
    適用於時間較長、可重新連線的通話。使用 `silenceDurationMs`、
    `startSensitivity` 和 `endSensitivity`，可針對電話音訊調整成更快速的輪流對話。

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              provider: "twilio",
              inboundPolicy: "allowlist",
              allowFrom: ["+15550005678"],
              realtime: {
                enabled: true,
                provider: "google",
                instructions: "簡短回應。在使用更深入的工具前呼叫 openclaw_agent_consult。",
                toolPolicy: "safe-read-only",
                consultPolicy: "substantive",
                consultThinkingLevel: "low",
                consultFastMode: true,
                agentContext: { enabled: true },
                providers: {
                  google: {
                    apiKey: "${GEMINI_API_KEY}",
                    model: "gemini-3.1-flash-live-preview",
                    speakerVoice: "Kore",
                    silenceDurationMs: 500,
                    startSensitivity: "high",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="OpenAI">
    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              realtime: {
                enabled: true,
                provider: "openai",
                providers: {
                  openai: { apiKey: "${OPENAI_API_KEY}" },
                },
              },
            },
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

如需提供者專屬的即時語音選項，請參閱 [Google 提供者](/zh-TW/providers/google)和
[OpenAI 提供者](/zh-TW/providers/openai)。

## 串流轉錄

`streaming` 將 Twilio Media Streams 連接至即時轉錄提供者。
傳統串流路徑需要 `provider: "twilio"`；使用 Telnyx、Plivo 或 mock 的設定會遭到拒絕。Telnyx 即時音訊會改用需要獨立驗證的 `realtime.enabled` 路徑。

目前的執行階段行為：

- `streaming.provider` 為選用設定。若未設定，Voice Call 會使用第一個已註冊的即時轉錄提供者。
- 隨附的即時轉錄提供者：Deepgram（`deepgram`）、ElevenLabs（`elevenlabs`）、Mistral（`mistral`）、OpenAI（`openai`）和 xAI（`xai`），由各自的提供者外掛註冊。
- 提供者擁有的原始設定位於 `streaming.providers.<providerId>` 下。
- Twilio 傳送已接受的串流 `start` 訊息後，Voice Call 會立即註冊該串流，在提供者連線期間，透過轉錄提供者將傳入媒體排入佇列，並只在即時轉錄就緒後才開始播放初始問候語。
- 如果 `streaming.provider` 指向未註冊的提供者，或未註冊任何提供者，Voice Call 會記錄警告並略過媒體串流，而不會使整個外掛失敗。

### 串流提供者範例

<Tabs>
  <Tab title="OpenAI">
    預設值：API 金鑰為 `streaming.providers.openai.apiKey` 或
    `OPENAI_API_KEY`；模型為 `gpt-4o-transcribe`；`silenceDurationMs: 800`；
    `vadThreshold: 0.5`。

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "openai",
                streamPath: "/voice/stream",
                providers: {
                  openai: {
                    apiKey: "sk-...", // 如果已設定 OPENAI_API_KEY，則為選用
                    model: "gpt-4o-transcribe",
                    silenceDurationMs: 800,
                    vadThreshold: 0.5,
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="xAI">
    預設值：API 金鑰為 `streaming.providers.xai.apiKey` 或 `XAI_API_KEY`（若兩者皆未設定，
    則改用 xAI OAuth 驗證設定檔）；端點為
    `wss://api.x.ai/v1/stt`；編碼為 `mulaw`；取樣率為 `8000`；
    `endpointingMs: 800`；`interimResults: true`。

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                streamPath: "/voice/stream",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}", // 如果已設定 XAI_API_KEY，則為選用
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## 通話使用的 TTS

Voice Call 使用核心 `tts` 設定，在通話中進行串流語音合成。你可以在外掛設定下使用**相同結構**覆寫它——該設定會與 `tts` 深度合併。

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

<Warning>
**語音通話會忽略 Microsoft speech。** 電話語音合成需要實作電話目標輸出的提供者；Microsoft speech 提供者未實作此功能，因此通話時會略過該提供者，改為嘗試備援鏈中的其他提供者。
</Warning>

行為說明：

- 外掛設定內的舊版 `tts.<provider>` 金鑰（`openai`、`elevenlabs`、`microsoft`、`edge`）會由 `openclaw doctor --fix` 修復；提交的設定應使用 `tts.providers.<provider>`。
- 啟用 Twilio 媒體串流時會使用核心 TTS；否則通話會改用提供者原生語音。
- 如果 Twilio 媒體串流已在使用中，Voice Call 不會改用 TwiML `<Say>`。如果該狀態下無法使用電話 TTS，播放要求會直接失敗，而不會混用兩種播放路徑。
- 當電話 TTS 改用次要提供者時，Voice Call 會記錄包含提供者鏈（`from`、`to`、`attempts`）的警告，以供偵錯。
- 當 Twilio 插話或串流終止清除待處理的 TTS 佇列時，已排入佇列的播放要求會正常結束，而不會讓等待播放完成的來電者持續卡住。

### TTS 範例

<Tabs>
  <Tab title="僅限核心 TTS">
```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "alloy" },
    },
  },
}
```
  </Tab>
  <Tab title="覆寫為 ElevenLabs（僅限通話）">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI 模型覆寫（深度合併）">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                speakerVoice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
</Tabs>

## 來電

來電政策預設為 `disabled`。若要啟用來電，請設定：

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "你好！我能如何協助你？",
}
```

<Warning>
`inboundPolicy: "allowlist"` 是低可信度的來電顯示篩選機制。外掛會將供應商提供的 `From` 值正規化，並與 `allowFrom` 比較。
網路鉤子驗證會確認傳送來源為供應商並保障承載資料的完整性，
但**無法**證明 PSTN/VoIP 來電號碼的所有權。請將
`allowFrom` 視為來電顯示篩選，而非可靠的來電者身分驗證。
</Warning>

自動回應使用代理程式系統。可透過 `responseModel`、
`responseSystemPrompt` 和 `responseTimeoutMs` 進行調整。

### 依號碼路由

當一個語音通話外掛接收多個電話號碼的來電，且每個號碼應如同不同線路般運作時，請使用 `numbers`。例如，
其中一個號碼可以使用輕鬆隨和的個人助理，另一個則使用商務
角色、不同的回應代理程式，以及不同的 TTS 語音。

路由會依據供應商提供的被撥 `To` 號碼選取。鍵必須
是 E.164 號碼。來電時，語音通話會解析一次相符的
路由，將相符路由儲存在通話記錄中，並在問候語、傳統自動回應路徑、即時
諮詢路徑和 TTS 播放中重複使用該生效設定。若沒有相符路由，則使用全域語音通話
設定。撥出通話不使用 `numbers`；發起通話時，請明確傳入撥出
目標、訊息和工作階段。

路由覆寫目前支援：

- `inboundGreeting`
- `tts`
- `agentId`
- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

`tts` 路由值會深度合併至全域語音通話 `tts` 設定之上，因此
通常只需覆寫供應商語音：

```json5
{
  inboundGreeting: "你好，這裡是總機。",
  responseSystemPrompt: "你是預設的語音助理。",
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "coral" },
    },
  },
  numbers: {
    "+15550001111": {
      inboundGreeting: "Silver Fox Cards，你好，請問需要什麼協助？",
      responseSystemPrompt: "你是一位回答簡潔的棒球卡專家。",
      tts: {
        providers: {
          openai: { speakerVoice: "alloy" },
        },
      },
    },
  },
}
```

### 語音輸出契約

對於自動回應，語音通話會在
系統提示詞附加嚴格的語音輸出契約，要求回覆 `{"spoken":"..."}` JSON。語音通話會
以防禦性方式擷取語音文字：

- 忽略標記為推理／錯誤內容的承載資料。
- 剖析直接 JSON、設有圍欄的 JSON，或行內 `"spoken"` 鍵。
- 退回純文字，並移除可能屬於規劃／中繼內容的開頭段落。

如此可讓語音播放聚焦於面向來電者的文字，並避免將
規劃文字洩漏至音訊中。

### 對話啟動行為

對於撥出的 `conversation` 通話，第一則訊息的處理會繫結至即時
播放狀態：

- 僅在初始問候語正在播放時，才會抑制插話佇列清除與自動回應。
- 若初始播放失敗，通話會返回 `listening`，且初始訊息會保留在佇列中以供重試。
- Twilio 串流的初始播放會在串流連線時啟動，不會額外延遲。
- 插話會中止進行中的播放，並清除已排入佇列但尚未播放的 Twilio TTS 項目。清除的項目會以已略過狀態完成，因此後續回應邏輯不必等待永遠不會播放的音訊，即可繼續執行。
- 即時語音對話使用即時串流本身的開場輪次。語音通話**不會**針對該初始訊息發送舊式 `<Say>` TwiML 更新，因此撥出的 `<Connect><Stream>` 工作階段會保持連接。

### Twilio 串流中斷寬限期

Twilio 媒體串流中斷時，語音通話會等待 **2000 ms**，再
自動結束通話：

- 若串流在此期間重新連線，將取消自動結束。
- 若寬限期結束後沒有串流重新註冊，則結束通話，以避免通話卡在進行中狀態。

## 過期通話清除器

使用 `staleCallReaperSeconds`（預設為 **120**）結束從未
接聽且從未進入即時對話狀態的通話，例如供應商始終未傳送終止網路鉤子的通知模式
通話。將其設為 `0` 即可
停用。

清除器每 30 秒執行一次，且只會結束沒有
`answeredAt` 時間戳記，也尚未處於終止或即時
（`speaking`/`listening`）狀態的通話，因此已接聽的對話絕不會被此計時器
清除；`maxDurationSeconds`（預設為 300）是另一個上限，用於
結束持續過久的已接聽通話。

對於電信業者可能延遲傳送響鈴／接聽
網路鉤子的通知型流程，請將 `staleCallReaperSeconds` 調高至超過預設值，以免正常但較慢的
通話過早遭到清除；`120`-`300` 秒是合理的正式環境
範圍。

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 120,
        },
      },
    },
  },
}
```

## 網路鉤子安全性

當閘道前方設有 Proxy 或通道時，外掛會重建
公開 URL 以進行簽章驗證。下列選項控制要信任哪些
轉送標頭：

<ParamField path="webhookSecurity.allowedHosts" type="string[]">
  允許清單中的轉送標頭主機。
</ParamField>
<ParamField path="webhookSecurity.trustForwardingHeaders" type="boolean">
  不使用允許清單，直接信任轉送標頭。
</ParamField>
<ParamField path="webhookSecurity.trustedProxyIPs" type="string[]">
  僅在請求的遠端 IP 符合清單時信任轉送標頭。
</ParamField>

其他保護措施：

- Twilio、Telnyx 和 Plivo 已啟用網路鉤子**重播防護**。重播的有效網路鉤子請求會收到確認，但會略過其副作用。
- Twilio 對話輪次的 `<Gather>` 回呼中包含每輪權杖，因此過期／重播的語音回呼無法滿足較新的待處理轉錄輪次。
- 當供應商要求的簽章標頭缺失時，未經驗證的網路鉤子請求會在讀取本文前遭到拒絕。
- 語音通話網路鉤子會使用共用的驗證前本文讀取設定檔（本文上限為 64 KB、讀取逾時為 5 秒），並在簽章驗證前套用每個鍵的進行中請求上限（預設每個鍵 8 個並行請求）。

使用穩定公開主機的範例：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## 命令列介面

```bash
openclaw voicecall call --to "+15555550123" --message "來自 OpenClaw 的問候"
openclaw voicecall start --to "+15555550123"   # call 的別名
openclaw voicecall continue --call-id <id> --message "有任何問題嗎？"
openclaw voicecall speak --call-id <id> --message "請稍候"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                      # 從日誌彙整輪次延遲
openclaw voicecall expose --mode funnel
```

當閘道已在執行時，操作型 `voicecall` 命令
會委派給閘道所擁有的語音通話執行階段，使命令列介面不會繫結
第二個網路鉤子伺服器。若無法連線至閘道，命令會退回至
獨立的命令列介面執行階段。

`latency` 會從預設的語音通話儲存路徑讀取 `calls.jsonl`。使用
`--file <path>` 可指向不同的日誌，使用 `--last <n>` 可將
分析限制為最後 N 筆記錄（預設為 200）。輸出包括輪次延遲與聆聽等待時間的最小值／最大值／平均值、
p50 和 p95。

## 代理程式工具

工具名稱：`voice_call`。

| 動作          | 引數                                       |
| --------------- | ------------------------------------------ |
| `initiate_call` | `message`, `to?`, `mode?`, `dtmfSequence?` |
| `continue_call` | `callId`, `message`                        |
| `speak_to_user` | `callId`, `message`                        |
| `send_dtmf`     | `callId`, `digits`                         |
| `end_call`      | `callId`                                   |
| `get_status`    | `callId`                                   |

語音通話外掛隨附相符的代理程式 skill。

## 閘道 RPC

| 方法                      | 引數                                                             | 備註                                                                     |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `voicecall.initiate`        | `to?`, `message`, `mode?`, `sessionKey?`, `requesterSessionKey?` | 省略 `to` 時，會回退使用 `toNumber` 設定。                     |
| `voicecall.start`           | `to`, `message?`, `mode?`, `dtmfSequence?`, `sessionKey?`        | 與 `initiate` 相同，但也接受連線前的 `dtmfSequence`。           |
| `voicecall.continue`        | `callId`, `message`                                              | 阻塞直到該回合完成；回傳逐字稿。                   |
| `voicecall.continue.start`  | `callId`, `message`                                              | 非同步變體：立即回傳 `operationId`。                      |
| `voicecall.continue.result` | `operationId`                                                    | 輪詢待處理的 `voicecall.continue.start` 作業以取得其結果。      |
| `voicecall.speak`           | `callId`, `message`                                              | 不等待就直接發話；當 `realtime.enabled` 時使用即時橋接。 |
| `voicecall.dtmf`            | `callId`, `digits`                                               |                                                                           |
| `voicecall.end`             | `callId`                                                         |                                                                           |
| `voicecall.status`          | `callId?`                                                        | 省略 `callId` 以列出所有進行中的通話。                                   |

`dtmfSequence` 僅可與 `mode: "conversation"` 搭配使用；通知模式的通話如需在連線後
傳送按鍵，應在通話建立後使用 `voicecall.dtmf`。

## 疑難排解

### 設定無法公開網路鉤子

請在執行閘道的同一環境中執行設定：

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

對於 `twilio`、`telnyx` 和 `plivo`，`webhook-exposure` 必須顯示綠色。已設定的
`publicUrl` 若指向本機或私人網路空間，仍會失敗，因為電信業者無法回呼至這些位址。
請勿將 `localhost`、`127.0.0.1`、`0.0.0.0`、`10.x`、`172.16.x`-`172.31.x`、
`192.168.x`、`169.254.x`、`fc00::/7`、`fd00::/8` 或其他電信業者級 NAT
範圍用作 `publicUrl`。

Twilio 通知模式的撥出通話會直接在建立通話的要求中傳送其初始 `<Say>` TwiML，
因此第一段語音訊息不依賴 Twilio 擷取網路鉤子 TwiML。狀態回呼、對話通話、
連線前 DTMF、即時串流及連線後通話控制仍需要公開網路鉤子。

請使用一種公開連線方式：

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          // 或
          tunnel: { provider: "ngrok" },
          // 或
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

變更設定後，請重新啟動或重新載入閘道，然後執行：

```bash
openclaw voicecall setup
openclaw voicecall smoke
```

除非傳入 `--yes`，否則 `voicecall smoke` 僅會進行試執行。

### 提供者認證資訊失敗

請檢查所選提供者與必要的認證資訊欄位：

- Twilio：`twilio.accountSid`、`twilio.authToken` 和 `fromNumber`，或
  `TWILIO_ACCOUNT_SID`、`TWILIO_AUTH_TOKEN` 和 `TWILIO_FROM_NUMBER`。
- Telnyx：`telnyx.apiKey`、`telnyx.connectionId`、`telnyx.publicKey` 和
  `fromNumber`，或 `TELNYX_API_KEY`、`TELNYX_CONNECTION_ID` 和
  `TELNYX_PUBLIC_KEY`。
- Plivo：`plivo.authId`、`plivo.authToken` 和 `fromNumber`，或
  `PLIVO_AUTH_ID` 和 `PLIVO_AUTH_TOKEN`。

認證資訊必須存在於閘道主機上。編輯本機殼層設定檔不會影響已在執行中的閘道，
必須重新啟動閘道或重新載入其環境才會生效。

### 通話已開始，但未收到提供者的網路鉤子

確認提供者主控台指向確切的公開網路鉤子 URL：

```text
https://voice.example.com/voice/webhook
```

接著檢查執行階段狀態：

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw logs --follow
```

常見原因：

- `publicUrl` 指向與 `serve.path` 不同的路徑。
- 通道 URL 在閘道啟動後發生變更。
- Proxy 轉送了要求，但移除或改寫了主機／通訊協定標頭。
- 防火牆或 DNS 將公開主機名稱路由至閘道以外的位置。
- 閘道重新啟動時未啟用語音通話外掛。

當閘道前方有反向 Proxy 或通道時，請將
`webhookSecurity.allowedHosts` 設為公開主機名稱，或針對已知的 Proxy 位址使用
`webhookSecurity.trustedProxyIPs`。僅在 Proxy 邊界由你控制時
使用 `webhookSecurity.trustForwardingHeaders`。

### 簽章驗證失敗

系統會根據 OpenClaw 從傳入要求重建的公開 URL 來驗證提供者簽章。
如果簽章驗證失敗：

- 確認提供者的網路鉤子 URL 與 `publicUrl` 完全相符，包括配置、主機和路徑。
- 對於 ngrok 免費方案 URL，通道主機名稱變更時請更新 `publicUrl`。
- 確保 Proxy 保留原始主機和通訊協定標頭，或設定 `webhookSecurity.allowedHosts`。
- 請勿在本機測試以外的環境啟用 `skipSignatureVerification`。

### Google Meet 的 Twilio 加入失敗

Google Meet 使用此外掛進行 Twilio 撥入加入。請先驗證語音通話：

```bash
openclaw voicecall setup
openclaw voicecall smoke --to "+15555550123"
```

接著明確驗證 Google Meet 傳輸方式：

```bash
openclaw googlemeet setup --transport twilio
```

如果語音通話顯示綠色，但 Meet 參與者始終未加入，請檢查 Meet
撥入號碼、PIN 和 `--dtmf-sequence`。電話通話可能運作正常，
但會議可能會拒絕或忽略錯誤的 DTMF 序列。

Google Meet 透過 `voicecall.start` 啟動 Twilio 電話端，
並附帶連線前 DTMF 序列。由 PIN 衍生的序列會將 Google Meet
外掛的 `voiceCall.dtmfDelayMs`（預設為 **12000 ms**）作為前置 Twilio
等待按鍵，因為 Meet 撥入提示可能較晚出現。接著，語音通話會在要求開場問候語之前，
重新導向回即時處理。

使用 `openclaw logs --follow` 查看即時階段追蹤。正常的 Twilio Meet
加入會依下列順序記錄：

- Google Meet 將 Twilio 加入委派給語音通話。
- 語音通話儲存連線前 DTMF TwiML。
- Twilio 初始 TwiML 會在即時處理前被取用並提供。
- 語音通話為 Twilio 通話提供即時 TwiML。
- Google Meet 在 DTMF 後延遲結束後，使用 `voicecall.speak` 要求播放開場語音。

`openclaw voicecall tail` 仍會顯示已持久化的通話記錄；這對通話狀態和逐字稿很有用，
但並非每次網路鉤子／即時轉換都會顯示於其中。

### 即時通話沒有語音

確認只啟用一種音訊模式：`realtime.enabled` 和
`streaming.enabled` 不可同時為 true。

對於即時 Twilio／Telnyx 通話，另請確認：

- 已載入並註冊即時提供者外掛。
- `realtime.provider` 未設定，或其名稱對應至已註冊的提供者。
- 閘道程序可取得提供者 API 金鑰。
- `openclaw logs --follow` 顯示已提供即時 TwiML、即時橋接已啟動，且初始問候語已排入佇列。

## 相關內容

- [對話模式](/zh-TW/nodes/talk)
- [文字轉語音](/zh-TW/tools/tts)
- [語音喚醒](/zh-TW/nodes/voicewake)
