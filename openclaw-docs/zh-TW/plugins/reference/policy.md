---
read_when:
    - 你正在安裝、設定或稽核政策外掛
summary: 新增以政策為依據的 doctor 檢查，以確認工作區符合規範。
title: 政策外掛
x-i18n:
    generated_at: "2026-07-26T08:06:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 440f2f46e4149fdd5e65bf0140d4981c6d840e8e8c8a85d05eeb23a0839a61ac
    source_path: plugins/reference/policy.md
    workflow: 16
---

# 政策外掛

新增由政策支援的 doctor 檢查，以確認工作區符合規範。

## 發佈

- 套件：`@openclaw/policy`
- 安裝途徑：隨附於 OpenClaw

## 介面

外掛

<!-- openclaw-plugin-reference:manual-start -->

## 行為

政策外掛會提供 doctor 健全狀況檢查，用於檢查由政策管理的 OpenClaw
設定及受治理的工作區宣告。政策目前涵蓋頻道規範符合性、
受治理的工具中繼資料、MCP 伺服器態勢、模型供應商態勢、
私人網路存取態勢、閘道暴露態勢、代理程式工作區／工具
態勢、已設定的全域／各代理程式工具態勢、已設定的沙箱執行階段
態勢、輸入／頻道存取態勢、資料處理態勢，以及 OpenClaw 設定秘密
供應商／認證設定檔態勢。

政策會將編寫的要求儲存在 `policy.jsonc`，觀察現有的
OpenClaw 設定和工作區宣告並將其作為證據，且透過
`openclaw policy check` 和 `openclaw doctor --lint` 回報偏移。通過政策
檢查時，會輸出政策、證據、發現項目與證明雜湊，供操作人員
記錄以進行稽核。

`openclaw policy compare --baseline <file>` 會將一個政策檔案與另一個
政策檔案進行比較。這僅檢查設定層級的規範符合性：它會使用政策規則中繼資料，
驗證受檢查的政策是否遺漏內容或弱於編寫的
基準，而且不會檢查執行階段狀態、認證資訊或秘密值。

工具態勢規則可以要求使用核准的設定檔、僅限工作區的檔案系統
工具、具有限制範圍的 exec 安全性／詢問／主機設定、停用提升權限模式、完全相符的
`alsoAllow` 項目，以及必要的工具拒絕項目。證據記錄會
加入附加的 `alsoAllow` 項目，因為這些項目可能會擴大有效的工具態勢。
這些檢查僅觀察設定是否符合規範；不會讀取執行階段核准
狀態或新增執行階段強制措施。

沙箱態勢規則可以要求使用核准的沙箱模式／後端、禁止主機
容器網路、禁止加入容器命名空間、要求容器以唯讀方式
掛載、禁止掛載容器執行階段通訊端與使用不受限的容器設定檔，
並要求沙箱瀏覽器 CDP 來源範圍。
這些檢查僅觀察設定是否符合規範；不會讀取執行階段核准
狀態、檢查執行中的容器或新增執行階段強制措施。

資料處理規則可以要求遮蔽記錄中的敏感資訊、禁止遙測
內容擷取、要求維護工作階段保留機制，以及禁止為工作階段
逐字稿建立記憶索引。這些檢查僅觀察設定是否符合規範；不會
檢查原始記錄、遙測匯出資料、逐字稿、記憶檔案、秘密
或個人資料。

`scopes.<scopeName>` 下的具名政策範圍可針對其列出的選取器，
新增更嚴格的一般政策區段。`agentIds` 支援 `tools`、
`agents.workspace`、`sandbox` 和 `dataHandling.memory`；`channelIds` 支援
`ingress.channels`。
未明確列於 `agents.entries.*` 中的執行階段代理程式 ID，會依據
繼承的全域／預設態勢接受檢查，而不會在沒有證據的情況下直接通過。
`policy.jsonc` 中的每個範圍都必須對其選取器有效且可強制執行。
覆疊規則是額外的宣告，因此不會削弱
頂層政策；當相同的已觀察設定同時違反兩個範圍時，也可能產生各自的發現項目。

<!-- openclaw-plugin-reference:manual-end -->

## 相關文件

- [政策](/zh-TW/cli/policy)
