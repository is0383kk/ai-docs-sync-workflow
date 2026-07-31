---
read_when:
    - 你想要對設定／狀態執行快速安全稽核
    - 你想要套用安全的「修正」建議（權限、收緊預設值）
summary: '`openclaw security` 的命令列介面參考（稽核並修正常見的安全性陷阱）'
title: 安全性
x-i18n:
    generated_at: "2026-07-26T07:15:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b5f9ea5cb746bfd29ff4d096062e81595abe99a883fc3b1113b45a3527d42d9
    source_path: cli/security.md
    workflow: 16
---

# `openclaw security`

安全工具：稽核及選用的安全修正。相關內容：[安全性](/zh-TW/gateway/security)。

```bash
openclaw security audit
openclaw security audit --deep
openclaw security audit --deep --password <password>
openclaw security audit --deep --token <token>
openclaw security audit --auth password --password <password>
openclaw security audit --fix
openclaw security audit --json
```

## 稽核模式

一般的 `security audit` 會維持在冷態設定／檔案系統／唯讀路徑：它不會探索外掛執行階段的安全性收集器，因此例行稽核不會載入每個已安裝外掛的執行階段。`--deep` 會加入盡力而為的即時閘道探測，以及由外掛擁有的安全性稽核收集器（明確的內部呼叫端若已具備適當的執行階段範圍，也可選擇啟用這些收集器）。

如果只在啟動時提供閘道密碼驗證，請透過 `--auth password --password <password>` 傳入相同的值，讓稽核能將其與 `hooks.token` 核對。

## 檢查項目

**私訊／信任模型**

- 當多個私訊傳送者共用主要工作階段時發出警告，並針對共用收件匣建議使用安全私訊模式：`session.dmScope="per-channel-peer"`（多帳號頻道則使用 `per-account-channel-peer`）。這是協作式／共用收件匣的強化措施，並非針對互不信任操作者的隔離；若要分隔信任邊界，請使用不同的閘道（或不同的作業系統使用者／主機）。
- 當設定顯示可能有多位共用使用者的輸入流量時（例如開放的私訊／群組政策、已設定的群組目標或萬用字元傳送者規則），會產生 `security.trust_model.multi_user_heuristic`——OpenClaw 的預設信任模型是個人助理（單一操作者），而非敵對的多租戶隔離。對於有意採用的多使用者共用設定：將所有工作階段置於沙箱中、將檔案系統存取限制於工作區範圍，並避免在該執行階段放置個人／私人身分或認證資訊。
- 使用小型模型（`<=300B` 個參數）、未啟用沙箱，且已啟用網頁／瀏覽器工具時發出警告。

**網路鉤子／鉤子**

啟動日誌會記錄非致命的安全性警告，而稽核會標記 `hooks.token` 重複使用有效中的閘道共用密鑰驗證值（`gateway.auth.token`／`OPENCLAW_GATEWAY_TOKEN`、`gateway.auth.password`／`OPENCLAW_GATEWAY_PASSWORD`）。下列情況也會發出警告：

- `hooks.token` 過短
- `hooks.path="/"`
- 未設定 `hooks.defaultSessionKey`
- `hooks.allowedAgentIds` 未受限制
- 已啟用要求的 `sessionKey` 覆寫
- 已啟用覆寫，但未設定 `hooks.allowedSessionKeyPrefixes`

執行 `openclaw doctor --fix` 以輪替已持久儲存且重複使用的 `hooks.token`，然後更新外部鉤子傳送端以使用新的權杖。

**沙箱／工具**

- 沙箱模式關閉，但已設定沙箱 Docker 設定時發出警告。
- `gateway.nodes.commands.deny` 使用無效的類模式／未知項目時發出警告（比對僅依節點命令名稱進行精確比對，不會篩選 Shell 文字）。
- `gateway.nodes.commands.allow` 明確啟用危險的節點命令時發出警告。
- 全域 `tools.profile="minimal"` 被代理程式工具設定檔覆寫時發出警告。
- 寫入／編輯工具已停用，但在缺乏具有限制作用的沙箱檔案系統邊界時，`exec` 仍可使用，則發出警告。
- 開放的私訊或群組在沒有沙箱／工作區防護的情況下暴露執行階段／檔案系統工具時發出警告。
- 已安裝外掛的工具可能在寬鬆的工具政策下可供存取時發出警告。

**沙箱瀏覽器**

- 沙箱瀏覽器使用 Docker `bridge` 網路，但未設定 `sandbox.browser.cdpSourceRange` 時發出警告。
- 標記危險的沙箱 Docker 網路模式，包括 `host` 和 `container:*` 命名空間加入。
- 現有沙箱瀏覽器 Docker 容器缺少雜湊標籤或標籤已過期時（例如遷移前的容器缺少 `openclaw.browserConfigEpoch`）發出警告，並建議使用 `openclaw sandbox recreate --browser --all`。

**網路／探索**

- 標記 `gateway.allowRealIpFallback=true`（若 Proxy 設定錯誤，可能有標頭偽造風險）。
- 標記 `discovery.mdns.mode="full"`（透過 mDNS TXT 記錄洩漏中繼資料）。
- `gateway.auth.mode="none"` 使閘道 HTTP API 在沒有共用密鑰的情況下仍可存取時（`/tools/invoke` 加上任何已啟用的 `/v1/*` 端點）發出警告。

**外掛／頻道**

- 以 npm 為基礎的外掛／鉤子安裝記錄未鎖定版本、缺少完整性中繼資料，或與目前已安裝的套件版本不一致時發出警告。
- 頻道允許清單依賴可變更的名稱／電子郵件／標籤，而非穩定 ID 時發出警告（適用時包括 Discord、Slack、Google Chat、Microsoft Teams、Mattermost、IRC 範圍）。

以 `dangerous`／`dangerously` 為前綴的設定，是操作者明確用於緊急解鎖的覆寫；僅啟用其中一項本身並不構成安全漏洞報告。如需完整的危險參數清單，請參閱[安全性](/zh-TW/gateway/security)中的「不安全或危險旗標摘要」。

## SecretRef 行為

`security audit` 會以唯讀模式解析其目標路徑支援的 SecretRef。如果目前命令路徑無法使用某個 SecretRef，稽核會繼續執行並回報 `secretDiagnostics`，而非當機。`--token` 和 `--password` 僅會覆寫該次命令執行的深度探測驗證；它們不會重寫設定或 SecretRef 對應。

## 抑制項目

使用 `security.audit.suppressions` 接受刻意保留的持續性發現。每個抑制項目會精確比對 `checkId`，並可透過不區分大小寫的 `titleIncludes` 和／或 `detailIncludes` 子字串進一步縮小範圍：

```json
{
  "security": {
    "audit": {
      "suppressions": [
        {
          "checkId": "plugins.tools_reachable_permissive_policy",
          "detailIncludes": "Enabled extension plugins: gbrain",
          "reason": "trusted local operator plugin"
        }
      ]
    }
  }
}
```

被抑制的發現會從有效的 `summary` 和 `findings` 清單中移除。JSON 輸出會將它們保留在 `suppressedFindings` 下，以供稽核追溯。設定抑制項目後，有效輸出也會保留一項不可抑制的 `security.audit.suppressions.active` 資訊發現，讓讀者知道稽核結果已經過篩選。危險設定旗標會以每個旗標一項發現的方式產生，因此接受一個危險旗標，不會隱藏共用相同 `config.insecure_or_dangerous_flags` checkId 的其他已啟用旗標。

由於抑制項目可能隱藏持續性風險，透過代理程式執行的 Shell 命令新增或移除它們時，需要 exec 核准；除非 exec 已使用 `security="full"` 和 `ask="off"` 執行受信任的本機自動化。

## JSON 輸出

```bash
openclaw security audit --json | jq '.summary'
openclaw security audit --deep --json | jq '.findings[] | select(.severity=="critical") | .checkId'
```

使用 `--fix --json` 時，輸出會同時包含修正動作和最終報告：

```bash
openclaw security audit --fix --json | jq '{fix: .fix.ok, summary: .report.summary}'
```

## `--fix` 會變更的項目

套用安全且具決定性的修正措施：

- 將常見的 `groupPolicy="open"` 切換為 `groupPolicy="allowlist"`（包括支援頻道中的帳號變體）
- 當 WhatsApp 群組政策切換為 `allowlist` 時，若已儲存的 `allowFrom` 檔案中存在清單，且設定尚未定義 `allowFrom`，則使用該檔案為 `groupAllowFrom` 填入初始值
- 將 `logging.redactSensitive` 從 `"off"` 設為 `"tools"`
- 收緊狀態／設定及常見敏感檔案的權限（`credentials/*.json`、`auth-profiles.json`、`openclaw-agent.sqlite`，以及舊版工作階段成品）
- 也會收緊從 `openclaw.json` 參照的設定引入檔案權限
- 在 POSIX 主機上使用 `chmod`，在 Windows 上使用 `icacls` 重設

`--fix` **不會**：

- 輪替權杖／密碼／API 金鑰
- 停用工具（`gateway`、`cron`、`exec` 等）
- 變更閘道繫結／驗證／網路暴露選項
- 移除或重寫外掛／Skills

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [安全性稽核](/zh-TW/gateway/security)
