---
read_when:
    - 你想將 API 金鑰從 openclaw.json 移出並存放在 1Password 中
    - 你以無介面模式執行閘道，並需要 op 的服務帳號驗證。
    - 你想讓代理程式使用 `op` 命令列介面讀取或注入密鑰
summary: 使用 1Password 命令列介面解析閘道密鑰，並讓代理程式使用內建的 1password Skill
title: 1Password
x-i18n:
    generated_at: "2026-07-26T07:50:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw 能以三種彼此獨立的方式搭配 **1Password**：

- **設定秘密：** `openclaw.json` 中的任何 [SecretRef](/zh-TW/gateway/secrets) 欄位都能在執行階段透過 `op` 命令列介面解析，因此 API 金鑰絕不會存放在設定檔中。
- **代理工作流程：** 內建的 `1password` skill 會教導代理登入，並使用 `op` 讀取或注入秘密，以完成自身的任務。
- **瀏覽器登入：** `claude-cli` 後端可以搭配 [1Password for Claude](https://support.1password.com/1password-claude/) 使用 Claude Code 的 Chrome 整合，讓代理登入網站，而密碼絕不會傳送至模型或 OpenClaw。

## 需求

- 在閘道主機上安裝 [1Password CLI](https://developer.1password.com/docs/cli/get-started/)（`op`；在 macOS 上為 `brew install 1password-cli`）。
- 為 `op` 設定一種驗證模式：
  - **服務帳戶**（建議用於無頭閘道）：在閘道服務環境中匯出 `OP_SERVICE_ACCOUNT_TOKEN`。不需要桌面應用程式，也不需要互動式登入。
  - **桌面應用程式整合**：1Password 應用程式需在同一台機器上執行，並啟用命令列介面整合。首次呼叫可能會觸發 Touch ID 或系統驗證。
  - **獨立登入**：`op signin` 會在每個工作階段提示登入。代理可透過該 skill 使用此方式，但不適合在無頭閘道上解析設定秘密。

## 使用 op 解析設定秘密

宣告一個執行 `op read` 並使用 `op://vault/item/field` 參照的 exec 秘密提供者，接著讓任何支援 SecretRef 的欄位指向它：

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // Homebrew 符號連結二進位檔需要此設定
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

各部分的運作方式：

- `command` 必須是絕對路徑；`trustedDirs` 會將其目錄標示為受信任，而由於 Homebrew 會將 `op` 安裝為符號連結，因此需要 `allowSymlinkCommand`。
- `args` 會原封不動地傳遞 `op://vault/item/field` 參照。OpenClaw 本身不會剖析 `op://` 配置；該參照由 `op` 二進位檔解析。
- `passEnv` 會轉送閘道環境中列出的變數。桌面應用程式整合需要 `HOME`；服務帳戶也需要閘道服務環境中存在 `OP_SERVICE_ACCOUNT_TOKEN`（將它加入 `passEnv`；只有在你接受該權杖可從設定檔中讀取時，才透過 `env` 設定）。
- 若輸出為單一值，請保留 `id: "value"`。若使用 `jsonOnly: true` 與 JSON 承載資料，則改用 JSON 指標 ID 指定欄位。
- 每項秘密使用一個提供者項目，可確保參照便於稽核；請依其使用者命名提供者（`onepassword_openai`、`onepassword_telegram`）。

如需瞭解析順序、快取與失敗語意，請參閱[閘道秘密](/zh-TW/gateway/secrets)；如需查看所有接受 SecretRef 的欄位，請參閱 [SecretRef 認證資訊介面](/zh-TW/reference/secretref-credential-surface)。

## 無頭閘道的服務帳戶設定

1. 在你的 1Password 帳戶中建立服務帳戶，並僅授予其讀取閘道所需保存庫項目的權限。
2. 將 `OP_SERVICE_ACCOUNT_TOKEN` 提供給閘道服務（launchd plist、systemd unit 或容器環境）。
3. 將 `"OP_SERVICE_ACCOUNT_TOKEN"` 加入提供者的 `passEnv` 清單。
4. 從閘道主機環境驗證：`op whoami` 應在不提示登入的情況下輸出服務帳戶。

服務帳戶讀取要求在 `op://` 參照中明確指定保存庫名稱。請嚴格限制帳戶範圍；它是持有者認證資訊。

## 代理使用的 1password skill

OpenClaw 內建一個 `1password` skill，可讓代理熟練操作 `op`：它會偵測可用的驗證模式（服務帳戶、桌面應用程式整合或獨立登入），在讀取任何內容前使用 `op whoami` 驗證存取權，並優先使用 `op run` / `op inject`，而不是將秘密值寫入磁碟。此 skill 需要 `op` 二進位檔，若缺少該檔案，則會提供 Homebrew 安裝選項。

代理會將它用於自身工作流程，例如在任務進行期間讀取部署權杖，或將環境變數注入命令。它與設定秘密解析彼此獨立；閘道解析 SecretRef 時不會使用任何 skill。

## 使用 1Password for Claude 進行瀏覽器登入

[1Password for Claude](https://support.1password.com/1password-claude/) 可讓 Claude 要求登入，而 1Password 瀏覽器擴充功能會透過加密通道，將認證資訊直接填入頁面。秘密絕不會進入模型上下文、逐字稿或 OpenClaw。當 OpenClaw 執行 [`claude-cli` 後端](/zh-TW/gateway/cli-backends#claude-cli-specifics)，且已啟用 Claude Code 的 Chrome 整合時，代理任務便可針對需要實際登入工作階段的網站使用此流程。

除了後端本身，還需要：

- 一台配備 Chrome 的 macOS 閘道主機、已連線的 [Claude in Chrome extension](https://code.claude.com/docs/en/chrome)、1Password 桌面應用程式，以及 1Password 瀏覽器擴充功能（兩者皆須為 8.12.28 或更新版本）。
- Claude Code 已登入直接的 Anthropic 方案（Pro、Max、Team 或 Enterprise）。Amazon Bedrock、Google Cloud 或其他第三方提供者不支援 Chrome 整合。
- 在 Anthropic 端完成一次性的 1Password 連線：透過 [1Password 指南](https://support.1password.com/1password-claude/)中所述的 Claude 桌面應用程式或擴充功能流程設定 1Password for Claude；目前此功能是 macOS 測試版。使用 1Password Business 時，管理員必須先在 Policies 下啟用 "Allow AI agents to autofill for users"；Anthropic Team/Enterprise 方案的此整合預設也是關閉的，必須由 Owner 啟用。
- 一個[命令列介面後端外掛](/zh-TW/plugins/cli-backend-plugins)，用於將 `--chrome` 加入 Claude 啟動引數；內建後端不會啟用 Chrome。
- 閘道主機旁需要有人操作：每次使用認證資訊時，都會顯示 1Password 提示，並需在該主機上確認（例如使用 Touch ID）。在限制嚴格的 exec 政策下，瀏覽器工具呼叫本身也會先轉送至你的頻道，作為 OpenClaw 核准要求。

在將此功能接入 OpenClaw 前，請先於閘道主機上的互動式工作階段中驗證各部分：執行 `claude --chrome`、確認擴充功能已連線，並檢查 `claude-in-chrome` 工具是否包含認證資訊工具。若這些工具未在該處出現，也不會透過 OpenClaw 出現。

一次性密碼會由 1Password 填入同一頁面；絕不要透過聊天轉送驗證碼或密碼。目前無頭或遠端閘道無法使用此流程，因為核准操作與瀏覽器都位於閘道主機上。

## 安全性注意事項

- 透過 exec 提供者解析的秘密值會保留在閘道記憶體中；設定快照與 `config.get` 回應會遮蔽 SecretRef 欄位。
- 絕不要將秘密值放入 `openclaw.json`、記錄或聊天中。設定檔只保留項目名稱，值則存放在 1Password 中。
- 1Password 稽核軌跡會顯示每次服務帳戶讀取，讓金鑰輪替與事件審查更容易執行。

## 疑難排解

- `command not found` 或衍生程序錯誤：使用 `op` 的絕對路徑，並將其目錄加入 `trustedDirs`。
- `op` 可解析，但讀取因符號連結錯誤而失敗：若使用 Homebrew 安裝，請設定 `allowSymlinkCommand: true`。
- `account is not signed in`：若使用服務帳戶，請確認 `OP_SERVICE_ACCOUNT_TOKEN` 已傳入閘道服務，且列於 `passEnv` 中；若使用桌面整合，請確認應用程式正在執行且已解鎖。
- 首次讀取緩慢：提高提供者的 `timeoutMs`；在繁忙的主機上，`op` 冷啟動可能超過嚴格的逾時限制。
