---
read_when:
    - 偵錯缺少操作員範圍的錯誤
    - 審查裝置或節點的配對核准事項
    - 新增或分類閘道 RPC 方法
summary: 閘道用戶端的操作者角色、範圍與核准時檢查
title: 操作員範圍
x-i18n:
    generated_at: "2026-07-26T07:53:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 40053793bb5a80afab28fdfcdcac6565abde6bca988389b03a407272c70043e2
    source_path: gateway/operator-scopes.md
    workflow: 16
---

操作員範圍會限制閘道用戶端在完成驗證後可執行的操作。
它們是單一受信任閘道操作員網域內的控制平面防護機制，
而非用於抵禦惡意行為的多租戶隔離。若要在人員、
團隊或機器之間實現強隔離，請以不同的作業系統使用者或主機執行個別閘道。

相關資訊：[安全性](/zh-TW/gateway/security)、[閘道通訊協定](/zh-TW/gateway/protocol)、
[閘道配對](/zh-TW/gateway/pairing)、[裝置命令列介面](/zh-TW/cli/devices)。

## 角色

每個閘道 WebSocket 用戶端都會以一種角色連線：

- `operator`：控制平面用戶端，例如命令列介面、控制介面、自動化，以及
  受信任的輔助程序。
- `node`：透過 `node.invoke` 公開命令的
  功能主機（macOS、iOS、Android、無頭環境）。

操作員 RPC 方法需要 `operator` 角色；由節點發起的方法
則需要 `node` 角色。

## 範圍層級

| 範圍                    | 意義                                                                                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `operator.read`         | 唯讀狀態、清單、目錄、日誌、工作階段讀取，以及其他不會變更狀態的呼叫。                                                                                      |
| `operator.write`        | 會變更狀態的操作員動作：傳送訊息、叫用工具、更新對話／語音設定、轉送節點命令。同時滿足 `operator.read`。                                                   |
| `operator.admin`        | 管理存取權。滿足所有 `operator.*` 範圍。變更設定、更新、原生掛鉤、保留命名空間及高風險核准均需要此範圍。                                                |
| `operator.pairing`      | 裝置與節點配對管理：列出、核准、拒絕、移除、輪替、撤銷。                                                                                                    |
| `operator.approvals`    | 執行與外掛核准 API。                                                                                                                                          |
| `operator.questions`    | 列出、讀取、回答及解決互動式問題。                                                                                                                           |
| `operator.talk.secrets` | 讀取包含秘密的對話設定。                                                                                                                                      |

未知的未來 `operator.*` 範圍必須完全相符，除非呼叫者
已持有 `operator.admin`。

## 方法範圍只是第一道關卡

每個閘道 RPC 都有一個最小權限方法範圍，用來判斷
要求是否可進入其處理常式。會依參數判斷的方法會在
分派前推導該範圍，讓授權失敗採用單一標準的結構化回應：

- `agent` 的一般回合需要 `operator.write`，而
  `/new` 或 `/reset` 工作階段生命週期命令則需要 `operator.admin`。
- `node.invoke` 的一般轉送命令需要 `operator.write`，而
  `browser.proxy`、`fs.listDir` 及 `terminal.upload` 則需要
  `operator.admin`。
- `talk.config` 需要 `operator.read`；`includeSecrets: true` 還需要
  `operator.talk.secrets`。

部分處理常式之後會根據實際核准或變更的項目
套用更嚴格的檢查：

- `device.pair.approve` 可透過 `operator.pairing` 存取，但核准
  操作員裝置時，只能發給或保留呼叫者已持有的範圍。
- `node.pair.approve` 可透過 `operator.pairing` 存取，之後會從
  待處理節點宣告的命令清單推導額外的核准範圍。
- `chat.send` 是寫入範圍的方法，但 `/config set` 與
  `/config unset` 聊天命令還需要額外的 `operator.admin`，
  無論呼叫者具有何種聊天傳送範圍皆是如此。

這讓較低範圍的操作員可以執行低風險配對動作，
而無須將所有配對核准都限制為僅限管理員。

工作階段變更 RPC 是依據協商後的操作員範圍授權，
與連線用戶端的 `client.id` 或 `client.mode` 無關。用戶端
身分仍可能影響連線與裝置驗證政策，但既不會
授予也不會移除工作階段變更權限。

## 裝置配對核准

裝置配對記錄是已核准角色與範圍的持久性資料來源。
已配對的裝置不會在未告知的情況下取得更廣泛的存取權：若重新連線時
要求更廣泛的角色或範圍，就會建立新的待處理升級
要求。

核准裝置要求：

- 不含操作員角色的要求不需要操作員範圍核准。
- 非操作員裝置角色（例如 `node`）的要求需要
  `operator.admin`，即使 `device.pair.approve` 本身只需要
  `operator.pairing`。
- 要求 `operator.read`、`operator.write`、`operator.approvals`、
  `operator.questions`、`operator.pairing` 或 `operator.talk.secrets` 時，
  呼叫者必須已持有該範圍或 `operator.admin`。
- 要求 `operator.admin` 時需要 `operator.admin`。
- 不含明確範圍的修復要求可以繼承現有操作員
  權杖的範圍；若該權杖具有管理員範圍，核准仍需要
  `operator.admin`。

非管理員的共用秘密與受信任 Proxy 工作階段，只能在其自行宣告的操作員範圍內
核准操作員裝置要求；即使這些工作階段能以其他方式使用
`operator.pairing`，核准非操作員角色仍僅限管理員。

對於已配對裝置的權杖工作階段，除非呼叫者
具有 `operator.admin`，否則管理範圍僅限自身：非管理員呼叫者只能看到自己的配對項目，
也只能核准、拒絕、輪替、撤銷或移除自己的裝置項目。

## 節點配對核准

舊版 `node.pair.*` 方法使用另一個由閘道擁有的節點配對儲存區。
WS 節點改用裝置配對（`role: node`），但仍採用相同的核准
詞彙。請參閱[閘道配對](/zh-TW/gateway/pairing)，瞭解這兩個
儲存區之間的關係。

`node.pair.approve` 會從待處理要求的
命令清單推導額外的必要範圍：

| 宣告的命令                                                                                                           | 必要範圍                              |
| -------------------------------------------------------------------------------------------------------------------- | ------------------------------------- |
| 無                                                                                                                   | `operator.pairing`                    |
| 一般節點命令                                                                                                         | `operator.pairing` + `operator.write` |
| `system.run`、`system.run.prepare`、`system.which`、`browser.proxy`、`fs.listDir` 或 `system.execApprovals.get/set` | `operator.pairing` + `operator.admin` |

核准節點宣告不會啟用具有獨立
執行階段允許清單關卡的命令。例如，核准宣告
`computer.act` 的節點需要配對與寫入範圍，但只會記錄該介面。
管理員或擁有者仍必須啟用 `computer.act`。在其維持
啟用期間，透過 `node.invoke` 叫用它需要寫入範圍，但不需要
每次動作都具有管理員範圍。

節點配對會建立身分與信任；它不會取代節點本身的
`system.run` 執行核准政策。

## 共用秘密驗證

共用閘道權杖／密碼驗證會被視為該閘道的受信任操作員存取。
與 OpenAI 相容的 HTTP 介面、`/tools/invoke` 及 HTTP
工作階段歷程記錄端點，會為共用秘密持有人驗證還原完整的預設操作員範圍集合，
即使呼叫者傳送較窄的宣告範圍亦然。

帶有身分的模式（例如受信任 Proxy 驗證或私人入口 `none`）
仍可遵循明確宣告的範圍。若要實現真正的信任
邊界隔離，請使用個別閘道。
