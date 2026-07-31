---
read_when:
    - 你希望 Codex 模式的 OpenClaw 代理程式使用 Codex Computer Use
    - 你正在 Codex Computer Use、PeekabooBridge 與直接使用 cua-driver MCP 之間做選擇
    - 你正在為內建的 Codex 外掛設定 computerUse
    - 你正在疑難排解 /codex 電腦操作狀態或安裝問題
summary: 為 Codex 模式的 OpenClaw 代理程式設定 Codex 電腦操作功能
title: Codex 電腦操作
x-i18n:
    generated_at: "2026-07-26T08:32:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use 是用於控制本機桌面的 Codex 原生 MCP 外掛。OpenClaw
不會內建桌面應用程式、不會自行執行桌面操作，也不會繞過
Codex 權限。隨附的 `codex` 外掛只會準備 Codex app-server：
它會啟用 Codex 外掛支援、尋找或安裝已設定的 Computer Use
外掛、檢查 `computer-use` MCP 伺服器是否可用，然後在 Codex 模式的回合期間，
讓 Codex 負責原生 MCP 工具呼叫。

當 OpenClaw 已使用原生 Codex 執行框架時，請參閱本頁。關於
執行階段本身的設定，請參閱 [Codex 執行框架](/zh-TW/plugins/codex-harness)。

這與 OpenClaw 內建的[節點後端電腦工具](/zh-TW/nodes/computer-use)不同。如果相同的代理程式合約應控制已配對的 Mac，無論代理程式是在閘道或其他節點上執行，都請使用內建工具。如果應由 Codex app-server 負責本機 MCP 安裝、權限及原生工具呼叫，請使用 Codex Computer Use。

## OpenClaw.app 與 Peekaboo

OpenClaw.app 的 Peekaboo 整合與 Codex Computer Use 分開運作。
macOS 應用程式可託管 PeekabooBridge 通訊端，讓 `peekaboo` 命令列介面重複使用
應用程式在本機取得的「輔助使用」與「螢幕錄製」權限，以供 Peekaboo 自有的
自動化工具使用。該橋接器不會安裝或代理 Codex Computer Use，而
Codex Computer Use 也不會透過 PeekabooBridge 通訊端呼叫。

如果你希望 OpenClaw.app 成為可感知權限的 Peekaboo 命令列介面自動化主機，
請使用 [Peekaboo 橋接器](/zh-TW/platforms/mac/peekaboo)。如果 Codex 模式的
OpenClaw 代理程式應在回合開始前備妥 Codex 原生的 `computer-use` MCP 外掛，
請參閱本頁。

## iOS 應用程式

iOS 應用程式與 Codex Computer Use 分開運作。它不會安裝或代理
Codex `computer-use` MCP 伺服器，也不是桌面控制後端。
iOS 應用程式會改以 OpenClaw 節點身分連線，並透過 `canvas.*`、`camera.*`、`screen.*`、
`location.*` 和 `talk.*` 等節點命令提供行動裝置功能。

如果你希望代理程式透過閘道操控 iPhone 節點，請使用 [iOS](/zh-TW/platforms/ios)。
如果 Codex 模式代理程式應透過 Codex 原生的 Computer Use 外掛控制
本機 macOS 桌面，請參閱本頁。

## 直接使用 cua-driver MCP

Codex Computer Use 並非提供桌面控制的唯一方式。如果你希望
由 OpenClaw 管理的執行階段直接呼叫 TryCua 的驅動程式，請透過 OpenClaw 的
MCP 登錄檔使用上游 `cua-driver mcp` 伺服器，而非
Codex 專用的市集流程。

安裝 `cua-driver` 後，可要求它提供 OpenClaw 命令：

```bash
cua-driver mcp-config --client openclaw
```

或直接註冊 stdio 伺服器：

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

此路徑會完整保留上游 MCP 工具介面，包括驅動程式
結構描述與結構化 MCP 回應。如果你希望 CUA 驅動程式
作為一般 OpenClaw MCP 伺服器使用，請選擇此方式。如果應由 Codex app-server
在 Codex 模式的回合內負責外掛安裝、MCP 重新載入及原生工具呼叫，
請使用本頁的 Codex Computer Use 設定。

CUA 驅動程式提供適用於 macOS、Windows（x64 與 ARM64）以及
Linux（x64 與 ARM64，預覽層級）的預發行版本。它仍需要
其應用程式提示取得的本機作業系統權限，例如 macOS 上的「輔助使用」與「螢幕錄製」。
OpenClaw 不會安裝 `cua-driver`、授予這些權限，或
繞過上游驅動程式的安全模型。

## 快速設定

如果 Codex 模式的回合必須在對話串開始前備妥
Computer Use，請設定 `plugins.entries.codex.config.computerUse`。`autoInstall: true` 會選用
Computer Use，並允許 OpenClaw 在回合開始前安裝或重新啟用它：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

使用此設定時，OpenClaw 會在每個 Codex 模式
回合前檢查 Codex app-server。如果缺少 Computer Use，但 Codex app-server 已探索到
可安裝的市集，OpenClaw 會要求 Codex app-server 安裝或
重新啟用此外掛，並重新載入 MCP 伺服器。在 macOS 上啟動隔離的
Codex app-server 前，如果缺少原生用戶端，自動安裝也會將所選桌面應用程式套件中
官方簽署的 Computer Use 服務應用程式複製至該
Codex 主目錄的 `computer-use` 目錄。
在 macOS 上，如果尚未註冊相符的市集，且存在標準桌面應用程式套件，OpenClaw
也會嘗試從 `/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled`
註冊隨附的 Codex 市集，並保留
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled`
作為舊版獨立安裝的備援。如果設定後仍無法讓
MCP 伺服器可用，回合會在對話串開始前失敗。
嚴格的就緒失敗屬於執行框架預檢失敗，因此模型備援
不會對每個模型候選項目重複相同的本機就緒程序。

變更 Computer Use 設定後，如果現有 Codex 對話串已經開始，
請先在受影響的聊天中使用 `/new` 或 `/reset`，再進行測試。

在 macOS 上，Computer Use 的受管理啟動會優先使用
`/Applications/ChatGPT.app/Contents/Resources/codex` 的桌面應用程式二進位檔，然後
針對舊版獨立安裝，備援至 `/Applications/Codex.app/Contents/Resources/codex`。
這也適用於會自行啟動用戶端的一次性 Computer Use 狀態與
安裝命令。如此可讓桌面控制持續由擁有本機 macOS 權限的
應用程式套件負責。如果未安裝桌面應用程式，OpenClaw 會改用安裝在
外掛旁的受管理 Codex 二進位檔。使用預設隔離代理程式主目錄的一般受管理 Codex 回合，
會優先使用該固定版本套件，避免較舊的桌面應用程式遮蔽目前的模型
支援。使用者範圍的主目錄仍會優先使用桌面應用程式，因為它們可以載入原生
Computer Use 狀態。有效 Codex 設定已啟用
Computer Use 的隔離代理程式主目錄，也會繼續優先使用桌面應用程式。明確的
`appServer.command` 設定或 `OPENCLAW_CODEX_APP_SERVER_BIN` 仍會覆寫
此受管理的選擇。

OpenClaw 會在同一個執行中的閘道內，依序執行原生 Codex 設定讀取與 Computer Use 安裝。
獨立的 Codex 程序或另一個閘道不在此互斥範圍內。在
閘道外變更原生 Codex 外掛設定後，請重新啟動閘道並開始新的聊天，再依賴新的
選擇。

## 命令

可在任何提供 `codex` 外掛命令介面的聊天介面中，
使用 `/codex computer-use` 命令。這些是 OpenClaw 聊天／執行階段
命令，不是 `openclaw codex ...` 命令列介面子命令：

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` 是預設動作，且為唯讀：它不會新增市集
來源、安裝外掛或啟用 Codex 外掛支援。如果沒有設定選用
Computer Use，即使執行過一次性安裝命令，`status` 仍可能回報已停用。

`install` 會啟用 Codex app-server 外掛支援、選擇性新增
已設定的市集來源、透過 Codex app-server 安裝或重新啟用已設定的外掛、
重新載入 MCP 伺服器，並驗證 MCP
伺服器是否提供工具。由於安裝會變更受信任的主機資源，
只有擁有者或 `operator.admin` 閘道用戶端可以執行 `install`。其他
已授權的傳送者仍可繼續使用唯讀的 `status` 命令，
包括搭配覆寫選項使用。

較舊的版本接受一次性的 `--plugin`、`--server` 和 `--mcp-server`
身分覆寫。請改為永久設定 `computerUse.pluginName` 和
`computerUse.mcpServerName`。使用舊版身分旗標時，
命令會指出要永久保留的確切設定，並在遷移指引中重述
要求的動作及所有支援的市集旗標。

## 市集選項

OpenClaw 使用 Codex 本身提供的相同 app-server API。
市集欄位會選擇 Codex 應從何處尋找 `computer-use`。

| 欄位                 | 適用情境                                                        | 安裝支援                                                   |
| -------------------- | --------------------------------------------------------------- | ---------------------------------------------------------- |
| 無市集欄位           | 你希望 Codex app-server 使用它已知的市集。                      | 可以，前提是 app-server 傳回本機市集。                     |
| `marketplaceSource`  | 你有可供 app-server 新增的 Codex 市集來源。                     | 可以，用於明確指定的 `/codex computer-use install`。         |
| `marketplacePath`    | 你已知道主機上的本機市集檔案路徑。                              | 可以，用於明確安裝及回合開始時的自動安裝。                 |
| `marketplaceName`    | 你希望依名稱選擇一個已註冊的市集。                              | 僅當所選市集具有本機路徑時才可以。                         |

新的 Codex 主目錄可能需要短暫時間來植入官方
市集。安裝期間，OpenClaw 會輪詢 `plugin/list`，最長
`marketplaceDiscoveryTimeoutMs` 毫秒（預設 60 秒）。

如果有多個已知市集包含 Computer Use，OpenClaw 會依序優先選擇
`openai-bundled`、`openai-curated`，再選擇 `local`。不明且有歧義的
相符項目會以封閉方式失敗，並要求你設定 `marketplaceName` 或
`marketplacePath`。

## 隨附的 macOS 市集

目前的 ChatGPT 桌面版會在此處隨附 Computer Use；舊版獨立
Codex 桌面版則在 `Codex.app` 下使用相同配置：

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

當 `computerUse.autoInstall` 為 true，且尚未註冊任何包含
`computer-use` 的市集時，OpenClaw 會嘗試新增第一個存在的標準
隨附市集根目錄：

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

你也可以從殼層使用 Codex 明確註冊它：

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

如果你使用非標準的 Codex 應用程式路徑，請執行一次 `/codex computer-use install
--source <marketplace-root>`，或將 `computerUse.marketplacePath` 設為
本機市集檔案路徑。只有在你持有
市集 JSON 檔案路徑時才使用 `--marketplace-path`，不要用於隨附的市集根目錄。

### 共用外掛快取

預設的 `pluginCacheMode: "independent"` 不會管理各個 Codex 主目錄及其
外掛快取。設定 `pluginCacheMode: "shared"`，即可在 app-server 啟動前，
將隨附的 Computer Use 外掛複製到作用中 Codex 主目錄內可探索的外掛快取。
共用模式會保留較舊的快取版本，因為執行中的 Codex 用戶端
可能仍參照其具有版本資訊的外掛目錄；替換複製失敗時，也會保留作用中的快取。明確設定
`marketplaceName` 或 `marketplacePath` 會停用此
協調程序，讓 OpenClaw 不會覆寫該選擇。

## 遠端目錄限制

Codex app-server 可以列出及讀取僅存在於遠端的目錄項目，但目前
不支援遠端 `plugin/install`。這表示 `marketplaceName`
可以選擇僅存在於遠端的市集來進行狀態檢查，但安裝與
重新啟用仍需要透過 `marketplaceSource` 或
`marketplacePath` 使用本機市集。

如果狀態顯示該外掛可從遠端 Codex 市集取得，但
不支援遠端安裝，請搭配本機來源或路徑執行安裝：

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## 設定參考

| 欄位                            | 預設值         | 含義                                                                                 |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------------ |
| `enabled`                       | 推斷           | 要求使用 Computer Use。設定其他 Computer Use 欄位時，預設為 true。                   |
| `autoInstall`                   | false          | 在回合開始時佈建原生用戶端，並安裝或重新啟用外掛。                                   |
| `marketplaceDiscoveryTimeoutMs` | 60000          | 安裝程序等待 Codex app-server 探索市集的時間。                                       |
| `liveTestTimeoutMs`             | 60000          | 暫時就緒執行緒及其清理要求的逾時時間。                                               |
| `toolCallTimeoutMs`             | 60000          | Computer Use `list_apps` 就緒工具呼叫的逾時時間。                             |
| `healthCheckEnabled`            | false          | 在所屬的 app-server 用戶端運作期間，定期執行就緒探測。                               |
| `healthCheckIntervalMinutes`    | 60             | 探測頻率；接受的值為 30、60、120 或 240 分鐘。                                       |
| `pluginCacheMode`               | `independent`  | 使用 `shared`，從隨附的桌面外掛重新整理 Codex 主目錄快取。                 |
| `strictReadiness`               | false          | 即時探測失敗時停止啟動，而非顯示警告後繼續。                                         |
| `autoRepair`                    | false          | 終止過時且限定範圍的 Computer Use MCP 子程序，並重試失敗的探測一次。                 |
| `marketplaceSource`             | 未設定         | 傳遞至 Codex app-server `marketplace/add` 的來源字串。                              |
| `marketplacePath`               | 未設定         | 包含此外掛的本機 Codex 市集檔案路徑。                                                 |
| `marketplaceName`               | 未設定         | 要選取的已註冊 Codex 市集名稱。                                                       |
| `pluginName`                    | `computer-use` | Codex 市集外掛名稱。                                                                 |
| `mcpServerName`                 | `computer-use` | 已安裝外掛所公開的 MCP 伺服器名稱。                                                   |

回合開始時的自動安裝會刻意拒絕已設定的 `marketplaceSource`
值。新增來源是明確的設定操作，因此請先使用一次
`/codex computer-use install --source <marketplace-source>`，之後再讓
`autoInstall` 從探索到的本機市集處理後續的重新啟用。
回合開始時的自動安裝可以使用已設定的 `marketplacePath`，因為它
已是主機上的本機路徑。

每個欄位也接受環境變數覆寫，並會在對應的設定鍵未設定時檢查：

| 欄位                            | 環境變數                                                       |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## OpenClaw 檢查的項目

OpenClaw 會在內部回報穩定的設定原因，並為聊天訊息格式化
使用者可見的狀態：

| 原因                         | 含義                                                   | 後續步驟                                              |
| ---------------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| `disabled`                   | `computerUse.enabled` 解析為 false。                   | 設定 `enabled` 或其他 Computer Use 欄位。    |
| `marketplace_missing`        | 沒有可用的相符市集。                                   | 設定來源、路徑或市集名稱。                            |
| `plugin_not_installed`       | 市集存在，但未安裝此外掛。                             | 執行安裝或啟用 `autoInstall`。                   |
| `plugin_disabled`            | 外掛已安裝，但在 Codex 設定中停用。                    | 執行安裝以重新啟用。                                  |
| `remote_install_unsupported` | 所選市集僅限遠端。                                     | 使用 `marketplaceSource` 或 `marketplacePath`。       |
| `mcp_missing`                | 外掛已啟用，但 MCP 伺服器無法使用。                    | 檢查 Codex Computer Use 與作業系統權限。              |
| `ready`                      | 外掛與 MCP 工具皆可使用。                              | 開始 Codex 模式回合。                                 |
| `check_failed`               | 狀態檢查期間有 Codex app-server 要求失敗。             | 檢查 app-server 連線與記錄。                          |
| `auto_install_blocked`       | 回合開始時的設定需要新增來源。                         | 先明確執行安裝。                                      |

聊天輸出包含外掛狀態、MCP 伺服器狀態、市集、
可用的工具，以及失敗設定步驟的特定訊息。

## macOS 權限

這個由 Codex 擁有的 Computer Use 路徑在 macOS 上執行；MCP 伺服器可能需要
本機作業系統權限，才能檢查或控制應用程式。（如需在 Windows 與 Linux 節點主機上
進行跨平台桌面控制，請參閱
[cua-computer 執行器](/zh-TW/nodes/computer-use#windows-and-linux-experimental-via-cua-driver)。）
如果 OpenClaw 表示 Computer Use 已安裝，但 MCP 伺服器無法使用，
請先確認 Codex 端的 Computer Use 設定：

- Codex app-server 正在應執行桌面控制的同一部主機上運作。
- Computer Use 外掛已在 Codex 設定中啟用。
- `computer-use` MCP 伺服器會出現在 Codex app-server MCP 狀態中。
- macOS 已授予桌面控制應用程式所需的權限。
- 目前的主機工作階段可以存取受控制的桌面。

當 `computerUse.enabled` 為 true 時，OpenClaw 會刻意採取失敗關閉。一個
Codex 模式回合不應在缺少設定所要求的原生桌面工具時，無聲地繼續執行。

## 疑難排解

**狀態顯示尚未安裝。**執行 `/codex computer-use install`。如果
未探索到市集，請傳入 `--source` 或 `--marketplace-path`。

**狀態顯示已安裝但已停用。**再次執行 `/codex computer-use install`。
Codex app-server 安裝程序會將外掛設定寫回已啟用狀態。

**狀態顯示不支援遠端安裝。**使用本機市集
來源或路徑。你可以檢查僅限遠端的目錄項目，但無法透過
目前的 app-server API 安裝。

**狀態顯示 MCP 伺服器無法使用。**重新執行一次安裝，讓 MCP
伺服器重新載入。如果仍無法使用，請修正 Codex Computer Use 應用程式、
Codex app-server MCP 狀態或 macOS 權限。

**狀態或探測在 `computer-use.list_apps` 上逾時。**外掛與
MCP 伺服器均已存在，但本機 Computer Use 橋接器沒有回應。
結束或重新啟動 Codex Computer Use，必要時重新啟動 Codex Desktop，然後
在新的 OpenClaw 工作階段中重試。如果主機先前曾透過較舊的受管理
Codex app-server 執行 Computer Use，請從桌面應用程式隨附的市集重新整理
已安裝的外掛（獨立 Codex 桌面安裝請使用 `Codex.app` 路徑）：

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**Computer Use 工具顯示 `Native hook relay unavailable`。**
Codex 原生工具掛鉤無法透過本機橋接器或閘道備援連線至運作中的 OpenClaw 中繼。
請使用 `/new` 或 `/reset` 啟動新的 OpenClaw 工作階段。
如果第一次可以運作，但之後的工具呼叫再次失敗，
`/new` 只會清除目前的嘗試；請重新啟動 Codex app-server 或
OpenClaw 閘道，以移除舊執行緒與掛鉤註冊，然後
在新的工作階段中重試。

**回合開始時的自動安裝拒絕來源。**這是刻意的行為。請先使用明確的
`/codex computer-use install --source
<marketplace-source>` 新增來源，之後回合開始時的自動安裝便可使用
探索到的本機市集。

## 相關內容

- [Codex 控制框架](/zh-TW/plugins/codex-harness)
- [Peekaboo 橋接器](/zh-TW/platforms/mac/peekaboo)
- [iOS 應用程式](/zh-TW/platforms/ios)
