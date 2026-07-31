---
read_when:
    - 你需要知道會載入哪些環境變數，以及載入順序。
    - 你正在偵錯閘道中缺少 API 金鑰的問題
    - 你正在記錄供應商驗證或部署環境的相關資訊
summary: OpenClaw 載入環境變數的位置與優先順序
title: 環境變數
x-i18n:
    generated_at: "2026-07-26T07:53:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw 會從多個來源載入環境變數。規則是**絕不覆寫現有值**。
工作區 `.env` 檔案是信任程度較低的來源：OpenClaw 會先忽略工作區 `.env` 中的供應商認證資訊與受保護的執行階段控制項，再套用優先順序。

## 優先順序（由高至低）

1. **程序環境**（閘道程序已從父層 Shell／常駐程式取得的內容）。
2. **目前工作目錄中的 `.env`**（dotenv 預設值；不覆寫；忽略供應商認證資訊與受保護的執行階段控制項）。
3. 位於 `~/.openclaw/.env` 的**全域 `.env`**（亦稱 `$OPENCLAW_STATE_DIR/.env`；建議用於供應商 API 金鑰；不覆寫）。
4. `~/.openclaw/openclaw.json` 中的**設定 `env` 區塊**（僅在缺少值時套用）。
5. **選用的登入 Shell 匯入**（`env.shellEnv.enabled` 或 `OPENCLAW_LOAD_SHELL_ENV=1`），僅針對缺少的預期金鑰套用。

在使用預設狀態目錄的全新 Ubuntu 安裝中，OpenClaw 也會在全域 `.env` 之後，將 `~/.config/openclaw/gateway.env` 視為相容性備援。如果兩個檔案都存在且內容不一致，OpenClaw 會保留 `~/.openclaw/.env` 並顯示警告。

如果設定檔完全不存在，會略過步驟 4；若已啟用 Shell 匯入，仍會執行。

## 支援的操作者環境變數

以下變數是為操作者提供的受支援環境合約。未記載於文件的 `OPENCLAW_*` 變數屬於內部實作細節，可能不經通知即移除。

### 路徑與執行個體

| 變數                     | 用途                                                              |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | 覆寫 OpenClaw 路徑預設值所使用的家目錄。                           |
| `OPENCLAW_STATE_DIR`     | 覆寫可變更的狀態目錄。                                             |
| `OPENCLAW_CONFIG_PATH`   | 覆寫作用中的設定檔路徑。                                           |
| `OPENCLAW_WORKSPACE_DIR` | 覆寫預設代理程式工作區。                                           |
| `OPENCLAW_PROFILE`       | 選取具隔離預設值的具名設定檔。                                     |
| `OPENCLAW_GIT_DIR`       | 覆寫開發頻道更新所使用的原始碼簽出。                               |
| `OPENCLAW_INCLUDE_ROOTS` | 允許從其他根目錄解析 `$include`。                          |

### 閘道與驗證

| 變數                        | 用途                                                            |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | 覆寫用戶端使用的遠端閘道 URL。                                  |
| `OPENCLAW_GATEWAY_PORT`     | 覆寫本機閘道連接埠。                                            |
| `OPENCLAW_GATEWAY_TOKEN`    | 為閘道伺服器與用戶端提供權杖驗證。                              |
| `OPENCLAW_GATEWAY_PASSWORD` | 為閘道伺服器與用戶端提供密碼驗證。                              |

### 供應商認證資訊

核心與隨附的供應商外掛會辨識下列認證資訊及供應商選取變數。如果需要限定範圍的認證資訊，而不是整個程序共用的單一值，請優先使用各供應商的設定或 SecretRef 欄位。

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY`，以及 `Z_AI_API_KEY`。

已安裝的第三方外掛可在其外掛資訊清單中宣告額外的認證資訊變數；這些變數是宣告它們之外掛的合約，而不是 OpenClaw 核心變數。

### 記錄與診斷

| 變數                                 | 用途                                                            |
| ------------------------------------ | --------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | 覆寫檔案與主控台記錄層級。                                      |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | 啟用模型傳輸計時診斷。                                          |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | 選取經遮蔽處理的模型承載資料診斷。                              |
| `OPENCLAW_DEBUG_SSE`                 | 選取 SSE 計時或事件預覽診斷。                                   |
| `OPENCLAW_DEBUG_CODE_MODE`           | 啟用程式碼模式介面診斷。                                        |
| `OPENCLAW_DIAGNOSTICS`               | 啟用具名診斷旗標，或使用 `0` 停用所有旗標。       |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | 選取時間軸診斷的 JSONL 路徑。                                   |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | 將事件迴圈樣本新增至時間軸診斷。                                |

### 功能與執行階段切換

| 變數                                 | 用途                                                                         |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | 從登入 Shell 匯入缺少的預期變數。                                             |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | 設定登入 Shell 匯入逾時。                                                     |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | 使用 `0` 停用 exec Shell 快照。                                |
| `OPENCLAW_OFFLINE`                   | 防止下載固定版本的代理程式輔助二進位檔。                                      |
| `OPENCLAW_BROWSER_HEADLESS`          | 強制受管理的瀏覽器以顯示介面（`0`）或無介面（`1`）模式啟動。 |
| `OPENCLAW_DISABLE_BONJOUR`           | 強制開啟（`0`）或關閉（`1`）Bonjour 廣播。       |
| `OPENCLAW_NO_AUTO_UPDATE`            | 停用自動套用更新。                                                            |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | 允許受信任的私人 DNS `ws://` 連線，作為緊急例外覆寫。               |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | 允許多個閘道程序，同時保留各狀態的擁有權鎖定。                                |
| `OPENCLAW_SKIP_CHANNELS`             | 啟動不含頻道傳輸的閘道，以進行疑難排解。                                      |
| `OPENCLAW_THEME`                     | 強制將終端介面色盤設為 `light` 或 `dark`。              |

## 供應商認證資訊與工作區 `.env`

請勿只將供應商 API 金鑰保存在工作區 `.env` 中。OpenClaw 會封鎖工作區 `.env` 檔案中的大量供應商認證資訊與端點重新導向金鑰，包括每個已知的供應商驗證環境變數（例如 `GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、`GROQ_API_KEY`、`DEEPSEEK_API_KEY`、`PERPLEXITY_API_KEY`、`BRAVE_API_KEY`、`TAVILY_API_KEY`、`EXA_API_KEY`、`FIRECRAWL_API_KEY`），以及任何以 `_API_HOST`、`_BASE_URL`、`_ENDPOINT` 或 `_HOMESERVER` 結尾的金鑰，還有完整的 `OPENCLAW_*`、`CLAWHUB_*`、`ANTHROPIC_API_KEY_*` 與 `OPENAI_API_KEY_*` 命名空間。

請改用下列任一受信任來源保存供應商認證資訊：

- 閘道程序環境，例如 Shell、launchd/systemd 單元、容器密鑰或 CI 密鑰。
- 位於 `~/.openclaw/.env` 或 `$OPENCLAW_STATE_DIR/.env` 的全域執行階段 dotenv 檔案。
- `~/.openclaw/openclaw.json` 中的設定 `env` 區塊。
- 啟用 `env.shellEnv.enabled` 或 `OPENCLAW_LOAD_SHELL_ENV=1` 時的選用登入 Shell 匯入。

如果你先前只將供應商金鑰或端點路由值儲存在工作區 `.env` 中，請將它們移至上述任一受信任來源。工作區 `.env` 仍可提供不屬於認證資訊、端點重新導向、主機覆寫或 `OPENCLAW_*` 執行階段控制項的一般專案變數。

如需瞭解安全性理由，請參閱[工作區 `.env` 檔案](/zh-TW/gateway/security#workspace-env-files)。

## 設定 `env` 區塊

有兩種對等方式可設定內嵌環境變數（兩者都不會覆寫）：

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

設定 `env` 區塊僅接受常值字串。它不會展開
`file:...` 值；例如，`XAI_API_KEY: "file:secrets/xai-api-key.txt"`
會以該確切字串傳遞給供應商。

對於以檔案為來源的供應商金鑰，請在支援 SecretRef 的認證資訊欄位上使用 SecretRef：

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

如需瞭解支援的欄位，請參閱[密鑰管理](/zh-TW/gateway/secrets)與
[SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface)。

## Shell 環境匯入

`env.shellEnv` 會執行你的登入 Shell，且僅匯入**缺少的**預期金鑰：

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

對應的環境變數：

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`（預設為 `15000`）

## Exec Shell 快照

在非 Windows 的閘道主機上，bash 與 zsh 的 `exec` 命令預設使用啟動快照。
在閘道程序環境中設定 `OPENCLAW_EXEC_SHELL_SNAPSHOT=0`，即可停用此路徑。
值 `false`、`no` 與 `off` 也會停用此功能。每次呼叫的 `exec.env` 值無法切換
快照或重新導向快照快取。

## 執行階段注入的環境變數

OpenClaw 也會將內容標記注入產生的子程序中：

- `OPENCLAW_SHELL=exec`：針對透過 `exec` 工具執行的命令設定。
- `OPENCLAW_SHELL=acp-client`：當 `openclaw acp client` 產生 ACP 橋接處理程序時設定。
- `OPENCLAW_SHELL=tui-local`：針對本機終端介面 `!` shell 命令設定。
- `OPENCLAW_CLI=1`：針對命令列介面進入點產生的子處理程序設定。

這些是執行階段標記（不是必要的使用者設定）。可在 shell／設定檔邏輯中使用，
以套用情境特定規則。

## UI 環境變數

- `OPENCLAW_THEME=light`：當終端機使用淺色背景時，強制採用淺色終端介面調色盤。
- `OPENCLAW_THEME=dark`：強制採用深色終端介面調色盤。
- `COLORFGBG`：如果終端機匯出此變數，OpenClaw 會使用背景色提示自動選擇終端介面調色盤。

## 設定中的環境變數替換

你可以使用 `${VAR_NAME}` 語法，直接在設定字串值中參照環境變數：

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

完整詳細資訊請參閱[設定：環境變數替換](/zh-TW/gateway/configuration-reference#env-var-substitution)。

## 密鑰參照與 `${ENV}` 字串

OpenClaw 支援兩種由環境驅動的模式：

- 設定值中的 `${VAR}` 字串替換。
- 針對支援密鑰參照的欄位使用 SecretRef 物件（`{ source: "env", provider: "default", id: "VAR" }`）。

兩者都會在啟用時從處理程序環境解析。SecretRef 的詳細資訊記載於[密鑰管理](/zh-TW/gateway/secrets)。
設定的 `env` 區塊本身不會解析 SecretRef 或 `file:...`
簡寫值。

## 路徑相關環境變數

| 變數                 | 用途                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | 覆寫用於 OpenClaw 內部路徑預設值的家目錄（`~/.openclaw/`、代理程式目錄、工作階段、認證資訊、安裝程式初始設定，以及預設開發簽出）。以專用服務使用者身分執行 OpenClaw 時很實用。 |
| `OPENCLAW_STATE_DIR`     | 覆寫狀態目錄（預設為 `~/.openclaw`）。                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | 覆寫設定檔路徑（預設為 `~/.openclaw/openclaw.json`）。                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | 目錄的路徑清單，`$include` 指令可從中解析設定目錄以外的檔案（預設：無；`$include` 僅限於設定目錄）。會展開波浪號。                                                         |

## 代理程式輔助工具下載

設定 `OPENCLAW_OFFLINE=1` 可防止 OpenClaw 下載其固定版本的 `fd`
與 `ripgrep` 輔助二進位檔。OpenClaw 工具目錄下現有的輔助工具
及可運作的系統二進位檔仍可使用；缺少的輔助工具會維持
不可用，而不會觸發網路要求。

## 記錄

| 變數                         | 用途                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | 覆寫檔案與主控台的記錄層級（例如 `debug`、`trace`）。優先於設定中的 `logging.level` 與 `logging.consoleLevel`。無效值會被忽略並顯示警告。 |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | 在 `info` 層級發出特定模型要求／回應的計時診斷，而不啟用全域偵錯記錄。                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | 模型承載資料診斷：`summary`、`tools` 或 `full-redacted`。`full-redacted` 會受限並經過遮蔽處理，但可能包含提示／訊息文字。                                               |
| `OPENCLAW_DEBUG_SSE`             | 串流診斷：`events` 用於首次／完成計時，`peek` 用於包含前五個經遮蔽處理的 SSE 事件。                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | 程式碼模式的模型介面診斷，包括隱藏供應商工具及精簡控制／直接強制執行。                                                                                  |

### `OPENCLAW_HOME`

設定後，`OPENCLAW_HOME` 會取代系統家目錄（`$HOME`／`os.homedir()`），作為 OpenClaw 內部路徑預設值。這包括預設狀態目錄、設定路徑、代理程式目錄、認證資訊、安裝程式初始設定工作區，以及 `openclaw update --channel dev` 使用的預設開發簽出。

**優先順序：** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Android 上的 Termux `PREFIX` 家目錄備援 > `os.homedir()`

**範例**（macOS LaunchDaemon）：

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` 也可設為波浪號路徑（例如 `~/svc`），使用前會透過相同的作業系統家目錄備援鏈展開。

`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH` 和 `OPENCLAW_GIT_DIR` 等明確路徑變數仍具有較高優先順序。偵測 shell 啟動檔、設定套件管理員，以及展開主機 `~` 等作業系統帳號工作，仍可能使用真正的系統家目錄。

## nvm 使用者：web_fetch TLS 失敗

如果 Node.js 是透過 **nvm**（而非系統套件管理員）安裝，內建的 `fetch()` 會使用
nvm 隨附的 CA 存放區，其中可能缺少現代根 CA（Let's Encrypt 的 ISRG Root X1/X2、
DigiCert Global Root G2 等）。這會導致 `web_fetch` 在大多數 HTTPS 網站上以 `"fetch failed"` 失敗。

在 Linux 上，OpenClaw 會自動偵測 nvm，並在實際啟動環境中套用修正：

- `openclaw gateway install` 會將 `NODE_EXTRA_CA_CERTS` 寫入 systemd 服務環境
- `openclaw` 命令列介面進入點會在 Node 啟動前設定 `NODE_EXTRA_CA_CERTS`，並重新執行自身

**手動修正（適用於舊版或直接啟動 `node ...`）：**

啟動 OpenClaw 前匯出此變數：

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

不要只依賴將此變數寫入 `~/.openclaw/.env`；Node 會在處理程序啟動時讀取
`NODE_EXTRA_CA_CERTS`。

## 舊版環境變數

OpenClaw 只會讀取 `OPENCLAW_*` 環境變數。早期版本中的舊版
`CLAWDBOT_*` 與 `MOLTBOT_*` 前綴會被無聲
忽略。

如果閘道處理程序啟動時仍設定了任何此類變數，OpenClaw 會發出
一則 Node 棄用警告（`OPENCLAW_LEGACY_ENV_VARS`），列出
偵測到的前綴與總數。請將舊版前綴替換為
`OPENCLAW_`，以重新命名每個值（例如將 `CLAWDBOT_GATEWAY_TOKEN` 改為
`OPENCLAW_GATEWAY_TOKEN`）；舊名稱不會生效。

## 相關內容

- [閘道設定](/zh-TW/gateway/configuration)
- [常見問題：環境變數與 .env 載入](/zh-TW/help/faq#env-vars-and-env-loading)
- [模型概覽](/zh-TW/concepts/models)
