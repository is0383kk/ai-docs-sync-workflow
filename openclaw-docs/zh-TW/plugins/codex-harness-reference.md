---
read_when:
    - 你需要每個 Codex 控制框架設定欄位
    - 你正在變更 app-server 的傳輸、驗證、探索或逾時行為
    - 你正在偵錯 Codex 控制框架啟動、模型探索或環境隔離問題
summary: Codex 控制框架的設定、驗證、探索與應用程式伺服器參考資料
title: Codex 執行框架參考資料
x-i18n:
    generated_at: "2026-07-26T08:39:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 149f065f5bef18d0f491c97facc4b5991afc3f7e1077abdc7a4b49f506eac3e0
    source_path: plugins/codex-harness-reference.md
    workflow: 16
---

此參考資料涵蓋官方 `codex` 外掛的詳細設定。
如需設定與路由決策，請先參閱
[Codex 控制框架](/zh-TW/plugins/codex-harness)。

## 外掛設定介面

所有 Codex 控制框架設定都位於 `plugins.entries.codex.config` 下。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
          appServer: {
            mode: "guardian",
          },
        },
      },
    },
  },
}
```

頂層欄位：

| 欄位                       | 預設值                    | 意義                                                                                                                                           |
| -------------------------- | ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `discovery`                | 已啟用                    | Codex app-server `model/list` 的模型探索設定。                                                                                    |
| `appServer`                | 受管理的 stdio app-server | 傳輸、命令、驗證、核准、沙箱與逾時設定。一般控制框架預設使用代理程式範圍的狀態。                        |
| `codexDynamicToolsLoading` | `"searchable"`           | 使用 `"direct"`，將 OpenClaw 動態工具直接放入初始 Codex 工具情境中。                                                       |
| `codexDynamicToolsExclude` | `[]`                     | 要從 Codex app-server 回合中省略的其他 OpenClaw 動態工具名稱。                                                                    |
| `codexPlugins`             | 已停用                    | 原生 Codex 外掛／應用程式支援，包括選擇啟用對已連線帳號應用程式的存取權。請參閱[原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)。 |
| `computerUse`              | 已停用                    | Codex Computer Use 設定。請參閱 [Codex Computer Use](/zh-TW/plugins/codex-computer-use)。                                                               |
| `sessionCatalog`           | 已啟用                    | 側邊欄的原生 Codex 工作階段探索。設定 `enabled: false` 可停用探索，而不停用供應商或控制框架。           |
| `supervision`              | 已停用                    | 面向代理程式的原生工作階段逐字稿與寫入控制原則。請參閱 [Codex 監督](/zh-TW/plugins/codex-supervision)。                          |

## 監督

原生工作階段探索預設會列出閘道電腦及已選擇啟用的配對節點上，未封存的 Codex 工作階段。若只要停用該目錄，請使用：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          sessionCatalog: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

`supervision` 會另外控制面向代理程式的工具：

| 欄位                  | 預設值                  | 意義                                                                                                                                                                                                                                      |
| --------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enabled`             | `false`                 | 啟用面向代理程式的 Codex 監督工具。這不會控制已驗證的操作員工作階段目錄。                                                                                                                            |
| `endpoints`           | 內建本機端點              | 為保留的 Codex 監督代理程式及獨立 MCP 工具提供相容性與進階端點目標。人員目錄與分支流程會忽略這些目標，並使用從 `appServer` 解析的監督 App Server。       |
| `allowRawTranscripts` | `false`                 | 啟用監督時，允許自主代理程式或獨立 MCP 讀取逐字稿及由逐字稿衍生的清單欄位。僅讀取 `codex_threads` 中繼資料仍然可用。不會控制已驗證的 Control UI 接續操作。     |
| `allowWriteControls`  | `false`                 | 啟用監督時，允許自主執行 `codex_threads` 分支、重新命名、封存與取消封存變更，以及獨立 MCP 的傳送、引導與中斷操作。不會略過其他繫結、主機、狀態或確認檢查。 |

端點項目接受下列欄位：

| 欄位           | 適用範圍       | 意義                                                                  |
| -------------- | ------------- | --------------------------------------------------------------------- |
| `id`           | 全部           | 穩定的端點 ID。                                                   |
| `label`        | 全部           | 選用的顯示標籤。                                               |
| `transport`    | 全部           | `"stdio-proxy"` 或 `"websocket"`。                                     |
| `command`      | `stdio-proxy` | 選用的 App Server 命令。                                          |
| `args`         | `stdio-proxy` | 選用的命令引數。                                           |
| `cwd`          | `stdio-proxy` | 選用的子程序工作目錄。                             |
| `url`          | `websocket`   | 必要的 WebSocket 或受支援的本機通訊端 URL。                     |
| `authTokenEnv` | `websocket`   | 選用的環境變數，其值用於驗證端點。 |

**Codex 工作階段**頁面使用外掛的監督 App Server，並且只顯示未封存的工作階段。若沒有明確的 `appServer` 連線設定，該連線會使用受管理的使用者家目錄 stdio。已儲存或閒置的本機資料列，可以透過截至最後一個終止且已持久儲存的來源回合為止、範圍受限的使用者與助理歷史記錄，建立模型鎖定的聊天。其私有繫結會讓快照分支、標準 `appServer` 來源分支、歷史記錄注入及後續回合都維持在該連線上。第一次標準啟動會使用分支所傳回的配對。後續繼續執行時會省略 OpenClaw 模型與供應商覆寫，讓 Codex 還原標準執行緒已持久儲存的配對；另外的原生變更可以更新該配對，但外層模型與備援鏈絕不會取代它。已儲存及閒置的資料列可在確認沒有其他執行器後封存，除非另一個作用中的 OpenClaw 繫結擁有完全相同的目標，或其某個尚未封存的衍生子項。OpenClaw 會遵循 Codex 的子項分頁，並在列舉錯誤、循環或安全限制耗盡時採取封閉式失敗。確認仍涵蓋未知的原生用戶端，以及從狀態變更到封存之間的競態條件。受監督且模型鎖定的聊天在保護原生繫結期間無法刪除。作用中的來源無法建立分支或封存，但仍可開啟現有的受監督聊天。每個配對節點資料列都維持唯讀；節點傳輸尚未提供控制框架所需的串流生命週期。

僅 `appServer.homeScope: "user"` 會變更受管理的控制框架程序所使用的 Codex 家目錄；它不會發布機群目錄。啟用監督不會變更控制框架預設值。相反地，當沒有明確的 `appServer` 連線設定時，獨立的監督連線預設使用受管理的使用者家目錄 stdio。該連線會採用明確設定。待處理及已提交的受監督繫結，會在每個回合保留該連線；監督停用或連線／生命週期偏移時會採取封閉式失敗，而不會退回代理程式家目錄控制框架。預設連線會與原生 Codex 用戶端共用已儲存的工作階段，但不共用其程序本機活動狀態。

舊版 `plugins.entries.codex-supervisor` 設定已淘汰。執行
`openclaw doctor --fix`，將舊項目、端點定義、原則旗標及外掛允許／拒絕參照移轉至此區塊。發生衝突時，以明確的標準 `codex.config.supervision` 值為準。

## App-server 傳輸

對於一般控制框架回合，OpenClaw 會啟動隨官方外掛提供的受管理 Codex 二進位檔（目前為 `@openai/codex` `0.145.0`）：

```bash
codex app-server --listen stdio://
```

這會讓 app-server 版本與官方 `codex` 外掛保持一致，而不是取決於本機剛好安裝的其他 Codex 命令列介面。只有在你有意使用不同的可執行檔時，才設定 `appServer.command`。使用預設隔離代理程式家目錄的一般受管理回合，即使已安裝 macOS 桌面應用程式套件，也會優先使用這個固定版本的套件。啟用 [Computer Use](/zh-TW/plugins/codex-computer-use) 時，或當 `homeScope` 為 `"user"` 且可以載入原生 Computer Use 狀態時，受管理啟動會改為優先使用擁有所需 macOS 權限的桌面應用程式二進位檔。當隔離代理程式家目錄的有效 Codex 設定啟用原生 Computer Use 時，也會套用相同的桌面優先規則。如果未安裝桌面應用程式套件，OpenClaw 會退回固定版本的套件二進位檔。

可執行檔交接與原生設定隔離機制會協調單一執行中閘道程序內的用戶端。當另一個程序變更原生 Codex 外掛設定後，請重新啟動閘道。

監督會解析獨立連線。若沒有明確的 `appServer` 連線設定，它會使用搭配 `homeScope: "user"` 的受管理 stdio；一般控制框架則維持搭配 `homeScope: "agent"` 的受管理 stdio。兩條路徑都會採用明確的連線設定。若一般控制框架應與原生用戶端共用 `$CODEX_HOME`（或 `~/.codex`），請明確設定 `homeScope: "user"`。無論一般控制框架的預設值為何，私有受監督繫結都會使用監督連線。各自獨立的 App Server 程序會保有不同的即時狀態與核准狀態。

若要在非正式環境中針對已執行的 app-server 進行測試，可使用 WebSocket 傳輸：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
            requestTimeoutMs: 60000,
          },
        },
      },
    },
  },
}
```

Codex 將 WebSocket 傳輸歸類為實驗性且不受支援。正式工作負載應優先使用受管理的 stdio 或本機 Unix 控制通訊端。

`appServer` 欄位：

| 欄位                                         | 預設值                                                | 說明                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` 會產生 Codex；明確指定 `"unix"` 會連線至本機控制通訊端；`"websocket"` 會連線至 `url`。                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` 會依各 OpenClaw 代理程式隔離一般測試框架狀態。`"user"` 是明確的選擇加入設定，會共用原生 `$CODEX_HOME` 或 `~/.codex`、使用原生驗證，並啟用僅限擁有者的執行緒管理。使用者範圍支援本機 stdio 或 Unix 傳輸。對於獨立的監督連線，未設定的值在 stdio 或 Unix 下會解析為 `"user"`，在 WebSocket 下則解析為 `"agent"`。     |
| `command`                                     | 受管理的 Codex 二進位檔                                   | stdio 傳輸使用的可執行檔。保持未設定即可使用受管理的二進位檔。                                                                                                                                                                                                                                                                                                                          |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio 傳輸的引數。                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | 未設定                                                  | WebSocket App Server URL 或 `unix://` URL。明確指定空白的 Unix 路徑會選取標準的使用者主目錄控制通訊端。                                                                                                                                                                                                                                                                          |
| `authToken`                                   | 未設定                                                  | WebSocket 傳輸的 Bearer 權杖。接受常值字串或 SecretInput，例如 `${CODEX_APP_SERVER_TOKEN}`。                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | 額外的 WebSocket 標頭。標頭值接受常值字串或 SecretInput 值，例如 `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`。                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | OpenClaw 建立繼承的環境後，從所產生的 stdio app-server 程序中移除的額外環境變數名稱。                                                                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | 未設定                                                  | 遠端 Codex app-server 工作區根目錄。設定後，OpenClaw 會從解析後的 OpenClaw 工作區推斷本機工作區根目錄、在此遠端根目錄下保留目前 cwd 的尾碼，並僅將最終的 app-server cwd 傳送至 Codex。若 cwd 位於解析後的 OpenClaw 工作區根目錄之外，OpenClaw 會採取失敗關閉，而不會將閘道本機路徑傳送至遠端 app-server。 |
| `loopDetectionPreToolUseRelay`                | `true`                                                 | 安裝 Codex `PreToolUse` 子程序，該程序僅用於 OpenClaw 迴圈偵測及其明確的無原則標記。設定 `false` 可減少每項工具的程序扇出。工具執行前的外掛掛鉤與受信任工具原則仍會安裝其必要的轉送器。                                                                                                                                         |
| `requestTimeoutMs`                            | `60000`                                                | app-server 控制平面呼叫的逾時時間。                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Codex 接受一輪互動後，或執行該輪範圍內的 app-server 要求後，OpenClaw 等待 `turn/completed` 時的靜默時窗。                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | 在最終／非評論的助理項目或工具執行前的原始助理完成訊號啟動助理輸出釋出後，OpenClaw 仍等待 `turn/completed` 時的靜默時窗。提高此值可讓 Codex 有更多時間發出 `turn/completed`，之後 OpenClaw 才會中斷並釋出工作階段通道。                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | 工具交接、原生工具完成、工具執行後的原始助理進度、原始推理完成或推理進度之後，OpenClaw 等待 `turn/completed` 時使用的完成閒置與進度防護。對於工具執行後的綜整可合理地維持較長靜默時間（超過最終助理釋出預算）的受信任或高負載工作，請使用此設定。                                |
| `mode`                                        | `"yolo"`，除非本機 Codex 要求不允許 YOLO | YOLO 或經守護者審查之執行的預設組態。                                                                                                                                                                                                                                                                                                                                                 |
| `approvalPolicy`                              | `"never"` 或允許的守護者核准原則       | 傳送至執行緒啟動、繼續和互動輪次的原生 Codex 核准原則。                                                                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` 或允許的守護者沙箱  | 傳送至執行緒啟動和繼續的原生 Codex 沙箱模式。作用中的 OpenClaw 沙箱會將 `danger-full-access` 互動輪次限縮為 Codex `workspace-write`；該輪的網路旗標會遵循 OpenClaw 沙箱的輸出規則。                                                                                                                                                                                       |
| `approvalsReviewer`                           | `"user"` 或允許的守護者審查者               | 在允許時，使用 `"auto_review"` 讓 Codex 審查原生核准提示。                                                                                                                                                                                                                                                                                                                   |
| `defaultWorkspaceDir`                         | 目前程序目錄                              | 省略 `--cwd` 時，`/codex bind` 使用的工作區。                                                                                                                                                                                                                                                                                                                                        |
| `serviceTier`                                 | 未設定                                                  | 選用的 Codex app-server 服務層級。`"priority"` 會啟用快速模式路由，`"flex"` 會要求彈性處理，而 `null` 會清除覆寫。舊版 `"fast"` 會視為 `"priority"` 接受。                                                                                                                                                                                                 |
| `networkProxy`                                | 已停用                                               | 選擇加入 Codex 權限設定檔網路功能，以供 app-server 命令使用。OpenClaw 會定義所選的 `permissions.<profile>.network` 設定，並透過 `default_permissions` 選取，而不是傳送 `sandbox`。                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | 預覽版選用功能，會向支援的 Codex 應用程式伺服器註冊由 OpenClaw 沙箱支援的 Codex 環境，讓原生 Codex 執行可在作用中的 OpenClaw 沙箱內運作。                                                                                                                                                                                                            |

`appServer.networkProxy` 是明確設定，因為它會變更 Codex 沙箱合約。啟用後，OpenClaw 也會在 Codex 執行緒設定中設定 `features.network_proxy.enabled` 和
`default_permissions`，讓產生的權限設定檔可以啟動由 Codex 管理的網路功能。OpenClaw 預設會根據設定檔內容產生具抗碰撞性的
`openclaw-network-<fingerprint>` 設定檔名稱；只有在需要穩定的本機名稱時，才使用 `profileName`。

```js
export default {
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
};
```

如果一般 app-server 執行階段原本會是 `danger-full-access`，啟用
`networkProxy` 後，產生的權限設定檔會改用工作區形式的檔案系統存取。由 Codex 管理的網路強制執行屬於沙箱化網路，因此完整存取權設定檔無法保護對外流量。

此外掛會封鎖較舊、較新但未經驗證、預發行、帶有建置後綴或未標示版本的 app-server 交握。Codex app-server 必須回報從 `0.143.0` 到隨附的 `0.145.0` 範圍內的穩定版本。

OpenClaw 會將非回送位址的 WebSocket app-server URL 視為遠端，並要求透過 `appServer.authToken` 或
`Authorization` 標頭提供帶有身分資訊的 WebSocket 驗證。`appServer.authToken` 和每個 `appServer.headers.*`
值都可以是 SecretInput；在 OpenClaw 建立 app-server 啟動選項之前，密鑰執行階段會解析 SecretRef 和環境變數簡寫，而未解析的結構化 SecretRef 會在傳送任何權杖或標頭前失敗。設定原生 Codex 外掛後，OpenClaw 會使用已連線 app-server 的外掛控制平面安裝或重新整理這些外掛，接著重新整理應用程式清單，讓外掛所擁有的應用程式可供 Codex 執行緒使用。`app/list` 仍是具權威性的清單和中繼資料來源，但即使 Codex 目前將某個列出且可存取的應用程式標示為停用，OpenClaw 政策仍會決定 `thread/start` 是否傳送 `config.apps[appId].enabled = true`。未知或缺少的應用程式 ID 仍會採取失敗關閉；此路徑只會透過 `plugin/install` 啟用市集外掛並重新整理清單。只有在信任遠端 app-server 能接受由 OpenClaw 管理的外掛安裝和應用程式清單重新整理時，才將 OpenClaw 連線至該遠端 app-server。

## 核准與沙箱模式

本機 stdio app-server 工作階段預設使用 YOLO 模式：
`approvalPolicy: "never"`、`approvalsReviewer: "user"` 和
`sandbox: "danger-full-access"`。這種受信任的本機操作人員模式，讓無人值守的 OpenClaw 回合和心跳偵測能持續進行，而不會顯示無人可回應的原生核准提示。

如果 Codex 的本機系統需求檔案不允許隱含的 YOLO 核准、審查者或沙箱值，OpenClaw 會改將隱含預設值視為 guardian，並選擇允許的 guardian 權限。`tools.exec.mode: "auto"`
也會強制使用由 guardian 審查的 Codex 核准，且不會保留不安全的舊版 `approvalPolicy: "never"` 或 `sandbox: "danger-full-access"` 覆寫；若有意採用不需核准的模式，請設定 `tools.exec.mode: "full"`。
同一需求檔案中與主機名稱相符的 `[[remote_sandbox_config]]` 項目，也會套用於沙箱預設值的決策。

若要使用由 Codex guardian 審查的核准，請設定 `appServer.mode: "guardian"`：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            mode: "guardian",
            serviceTier: "priority",
          },
        },
      },
    },
  },
}
```

當這些值獲允許時，`guardian` 預設集會展開為 `approvalPolicy: "on-request"`、
`approvalsReviewer: "auto_review"` 和 `sandbox: "workspace-write"`。個別政策欄位會覆寫 `mode`。較舊的
`guardian_subagent` 審查者值仍可作為相容性別名使用，但新設定應使用 `auto_review`。

啟用 OpenClaw 沙箱時，本機 Codex app-server 程序仍會在閘道主機上執行。因此，OpenClaw 會在該回合停用 Codex 原生 Code Mode、使用者 MCP 伺服器和由應用程式支援的外掛執行，而不會將 Codex 主機端沙箱視為等同於 OpenClaw 沙箱後端。當一般 exec/process 工具可用時，Shell 存取會透過由 OpenClaw 沙箱支援的動態工具公開，例如 `sandbox_exec` 和 `sandbox_process`。

<Note>
在以 Docker 為後端的 OpenClaw 沙箱主機上（`agents.defaults.sandbox.mode` 設為 Docker 後端），`openclaw doctor` 會探測主機是否允許非特權使用者命名空間，以及在停用 Docker 沙箱網路輸出時是否允許網路命名空間；巢狀 Codex `bwrap` 需要這些命名空間，才能在沙箱容器內執行 `workspace-write` Shell。探測失敗通常會在 Ubuntu/AppArmor 主機上顯示為 `bwrap: setting up uid map: Permission denied` 或
`bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`。請為 OpenClaw 服務使用者修正所回報的主機命名空間政策，然後重新啟動閘道；相較於主機全域的
`kernel.apparmor_restrict_unprivileged_userns=0` 備援方式，應優先為服務程序使用限定範圍的 AppArmor 設定檔，且不要只為了滿足巢狀 `bwrap` 而授予 Docker 容器更廣泛的權限。
</Note>

## 沙箱化原生執行

穩定的預設行為是失敗關閉：啟用 OpenClaw 沙箱會停用原本會從 Codex app-server 主機執行的原生 Codex 執行介面。只有在想要搭配 OpenClaw 沙箱後端試用 Codex 的遠端環境支援時，才使用 `appServer.experimental.sandboxExecServer: true`。此預覽路徑適用於所有支援的 Codex app-server 版本。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            experimental: {
              sandboxExecServer: true,
            },
          },
        },
      },
    },
  },
}
```

啟用此旗標且目前的 OpenClaw 工作階段位於沙箱中時，OpenClaw 會啟動由作用中沙箱支援的本機回送 exec-server，將其註冊至 Codex app-server，並使用該 OpenClaw 所擁有的環境啟動 Codex 執行緒和回合。如果 app-server 無法註冊該環境，執行會採取失敗關閉，而不會在未提示的情況下退回主機執行。

此預覽路徑僅限本機使用。遠端 WebSocket app-server 除非與回送 exec-server 在同一部主機上執行，否則無法連線至該伺服器，因此 OpenClaw 會拒絕此組合。

## 驗證與環境隔離

在預設的個別代理程式主目錄中，驗證會依下列順序選取：

1. 代理程式的明確 OpenClaw Codex 驗證設定檔。
2. 該代理程式 Codex 主目錄中 app-server 的現有帳號。
3. 僅限本機 stdio app-server 啟動：當 app-server 沒有帳號且仍需要 OpenAI 驗證時，先使用 `CODEX_API_KEY`，再使用
   `OPENAI_API_KEY`。

當 OpenClaw 偵測到 ChatGPT 訂閱形式的 Codex 驗證設定檔（OAuth 或權杖認證資訊類型）時，會從產生的 Codex 子程序移除 `CODEX_API_KEY` 和 `OPENAI_API_KEY`。如此可讓閘道層級的 API 金鑰繼續供嵌入或直接 OpenAI 模型使用，同時避免原生 Codex app-server 回合意外透過 API 計費。

明確的 Codex API 金鑰設定檔和本機 stdio 環境金鑰備援會使用 app-server 登入，而不是繼承子程序環境。WebSocket app-server 連線不會收到閘道環境中的 API 金鑰備援；請使用明確的驗證設定檔或遠端 app-server 自己的帳號。

stdio app-server 啟動預設會繼承 OpenClaw 的程序環境。OpenClaw 擁有 Codex app-server 帳號橋接，並將 `CODEX_HOME` 設為該代理程式 OpenClaw 狀態下的個別代理程式目錄。如此可將 Codex 設定、帳號、外掛快取／資料和執行緒狀態限定於 OpenClaw 代理程式，而不會從操作人員個人的 `~/.codex` 主目錄洩漏進來。

若要與 Codex Desktop 和命令列介面共用原生 Codex 狀態，請設定 `appServer.homeScope: "user"`。此本機使用者主目錄模式支援受管理的 stdio 和明確的 Unix 傳輸。設定 `$CODEX_HOME` 時會使用該值，否則使用 `~/.codex`，包括原生驗證、設定、外掛和執行緒。OpenClaw 會略過 app-server 的驗證設定檔橋接。經驗證的擁有者回合可使用 `codex_threads` 來列出（可搭配選用的 `search` 篩選器）、讀取、分支、重新命名、封存和取消封存這些執行緒。請先分支執行緒，再於 OpenClaw 中繼續執行；獨立的 Codex 程序不會協調同一執行緒的並行寫入者。

該 `homeScope` 選擇加入設定適用於一般測試框架工作階段。透過 Codex Sessions 建立的 Chat 會改用其私人監督連線，為標準分支和未來繼續執行保留原生連線的驗證與提供者設定。

在鎖定模型的受監督 Chat 中，`codex_threads` 無法附加不同的分支，也無法封存 Chat 所繫結的原生執行緒。仍可使用清單和僅限中繼資料的讀取。原始逐字稿讀取需要 `allowRawTranscripts`；停用時，也會拒絕清單搜尋，因為原生搜尋可能會比對逐字稿預覽。若要重新命名、取消封存、建立分離式分支，或封存不屬於其他 OpenClaw Chat 的無關執行緒，則需要
`allowWriteControls`。這兩個選項都無法略過鎖定的繫結。

OpenClaw 不會針對一般本機 app-server 啟動重寫 `HOME`。由 Codex 執行的子程序，例如 `openclaw`、`gh`、`git`、雲端命令列介面和 Shell 命令，都會看到一般程序主目錄，並能找到使用者主目錄中的設定和權杖。Codex 也可能探索到 `$HOME/.agents/skills` 和
`$HOME/.agents/plugins/marketplace.json`；此 `.agents` 探索會刻意與操作人員主目錄共用，並與隔離的 `~/.codex` 狀態分開。

在預設的代理程式範圍中，OpenClaw 外掛和 OpenClaw skill 快照仍會透過 OpenClaw 自己的外掛登錄和 skill 載入器流通；個人的 Codex `~/.codex` 資產則不會。如果 Codex 主目錄中有實用的 Codex 命令列介面 skills 或外掛，且應納入隔離的 OpenClaw 代理程式，請明確盤點：

```bash
openclaw migrate codex --dry-run
openclaw migrate apply codex --yes
```

如果部署需要額外的環境隔離，請將這些變數加入 `appServer.clearEnv`：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` 只會影響產生的 Codex app-server 子程序。OpenClaw 會在本機啟動正規化期間，從此清單移除 `CODEX_HOME` 和 `HOME`：`CODEX_HOME` 會繼續指向選取的代理程式或使用者範圍，而 `HOME` 會繼續繼承，讓子程序能使用一般的使用者主目錄狀態。

## 動態工具

Codex 動態工具預設使用 `searchable` 載入，並透過具有 `deferLoading: true` 的
`openclaw` 命名空間公開。OpenClaw 通常不會公開與 Codex 原生工作區操作或 Codex 自有工具搜尋介面重複的動態工具：

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`
- `tool_call`
- `tool_describe`
- `tool_search`
- `tool_search_code`

當有限的執行階段允許清單停用原生 Code Mode 時，OpenClaw 會傳送空的執行環境選項。在這種直接且未沙箱化的情況下，OpenClaw 會保留經政策篩選的 `exec` 和 `process` 工具，作為 Shell 備援。執行階段允許清單和 `codexDynamicToolsExclude` 仍然適用。

其餘大多數 OpenClaw 整合工具，例如訊息、媒體、排程、
瀏覽器、節點、閘道、`heartbeat_respond` 和 `web_search`，都可透過
該命名空間下的 Codex 工具搜尋使用。這可縮小模型的初始
上下文。無論 `codexDynamicToolsLoading` 為何，仍有一小組工具可直接
呼叫，因為 Codex 工具搜尋可能無法使用，或
解析為僅含連接器的工具集合：`agents_list`、`sessions_spawn` 和
`sessions_yield`。開發者指示仍會引導一般 Codex 子代理
針對 Codex 原生子代理工作使用原生 `spawn_agent`，而
`sessions_spawn` 則仍可用於明確的 OpenClaw 或 ACP 委派。
僅限訊息工具的來源回覆也會維持直接處理，因為這是一項
回合控制合約。

Codex Code Mode 會將一般 OpenClaw 動態工具結果呈現為文字。請先剖析
JSON 結果，再讀取欄位。巢狀動態呼叫會由
Codex 執行階段序列化，因此 `Promise.all` 不會並行提交這些呼叫；啟動收集器子項目時，請使用
有界的循序啟動迴圈。

標示為 `catalogMode: "direct-only"` 的工具（包括 OpenClaw `computer`
工具）會歸入 `openclaw_direct`。OpenClaw 會將該命名空間加入
Codex 的 `code_mode.direct_only_tool_namespaces` 清單，而不取代
操作員提供的項目。因此，Codex 會在一般執行緒和僅限程式碼模式的執行緒中，將這些工具公開為
`DirectModelOnly`，而非透過巢狀 Code Mode `tools.*` 呼叫
路由。此邊界對含有圖片的結果而言不可或缺：巢狀 Code Mode 序列化會將圖片輸出扁平化為
文字，導致下一個電腦操作所需的螢幕截圖遭到捨棄。

只有在連線至無法搜尋延後載入動態工具的自訂
Codex app-server，或偵錯完整工具承載資料時，才設定 `codexDynamicToolsLoading: "direct"`。

## 逾時

OpenClaw 擁有的動態工具呼叫會獨立於
`appServer.requestTimeoutMs` 受到限制。每個 Codex `item/tool/call` 請求會依下列
順序使用第一個可用的逾時值：

- 正值的單次呼叫 `timeoutMs` 引數。
- 若為 `image_generate`，則使用 `agents.defaults.mediaModels.image.timeoutMs`。
- 若為未設定逾時的 `image_generate`，則使用 120 秒的
  圖片生成預設值。
- 若為媒體理解 `image` 工具，則使用所選支援圖片的 `tools.media.models[]` 項目之 `timeoutSeconds`
  換算為毫秒的值，或 60 秒的媒體預設值。對於圖片
  理解，此值套用於請求本身，且不會因先前的
  準備工作而縮短。
- 若為 `message` 工具，則使用固定的 600 秒外層預算，涵蓋閘道傳遞與有界的同鍵協調。
- 90 秒的動態工具預設值。

此監視器是外層動態 `item/tool/call` 預算。供應商特定的
請求逾時會在該呼叫內執行，並保有各自的逾時語意。
動態工具預算上限為 600000 ms。`agents_wait` 會增加 30000 ms 的
外層完成寬限時間，而 app-server 用戶端允許 660000 ms，使該
結構化等待結果能夠送達 Codex。發生逾時時，OpenClaw 會在支援的情況下中止工具
訊號，並向 Codex 傳回失敗的動態工具回應，讓
回合得以繼續，而非使工作階段停留在 `processing`。

Codex 接受回合後，以及 OpenClaw 回應回合範圍的
app-server 請求後，執行框架會預期 Codex 在目前回合中取得進展，
並最終以 `turn/completed` 完成原生回合。如果
app-server 在 `appServer.turnCompletionIdleTimeoutMs` 期間沒有動靜，OpenClaw
會盡力中斷 Codex 回合、記錄診斷逾時，並
釋放 OpenClaw 工作階段通道，避免後續聊天訊息排在
過期的原生回合之後。

同一回合的大多數非終止通知都會解除該短期監視器，
因為 Codex 已證明該回合仍在運作。工具交接會使用較長的
工具後閒置預算：在 OpenClaw 傳回 `item/tool/call` 回應後、
在 `commandExecution` 等原生工具項目完成後、在原始
`custom_tool_call_output` 完成後，以及在工具後原始助理
進度、原始推理完成或推理進度之後。此防護機制在已設定時使用
`appServer.postToolRawAssistantCompletionIdleTimeoutMs`，否則預設為五分鐘。相同的工具後預算也會延長
進度監視器，以涵蓋 Codex 發出下一個目前回合事件前的靜默綜合期間。
推理完成、評論 `agentMessage`
完成，以及工具前原始推理或助理進度之後，可能會接著產生
自動最終回覆，因此它們會使用進度後回覆防護，而非立即釋放工作階段通道。
只有最終／非評論的已完成 `agentMessage` 項目，以及工具前原始助理完成，才會啟動
助理輸出釋放機制：若 Codex 隨後保持靜默且未出現 `turn/completed`，
OpenClaw 會盡力中斷原生回合並釋放工作階段
通道。可安全重播的 stdio app-server 失敗，包括沒有助理、
工具、作用中項目或副作用證據的回合完成閒置
逾時，會在新的 app-server 嘗試中重試一次。不安全的逾時仍會淘汰
卡住的 app-server 用戶端，並釋放 OpenClaw 工作階段通道。這些逾時也會
清除過期的原生執行緒繫結，而非自動
重播。完成監視逾時會顯示 Codex 特定的逾時文字：
可安全重播的情況會指出回應可能不完整，而不安全的情況會要求
使用者在重試前確認目前狀態。公開逾時診斷
包含結構化欄位，例如最後一個 app-server 通知方法、
原始助理回應項目的 id／類型／角色、作用中請求／項目數量，以及
已啟動的監視狀態。當最後一則通知是原始助理回應
項目時，也會包含有界的助理文字預覽。診斷不會
包含原始提示或工具內容。

## 模型探索

根據預設，Codex 外掛會向 app-server 查詢可用模型。模型
可用性由 Codex app-server 管理，因此當
OpenClaw 升級內附的 `@openai/codex` 版本，或部署將
`appServer.command` 指向不同的 Codex 二進位檔時，清單可能改變。可用性也可能
因帳戶而異。請在執行中的閘道上使用 `/codex models`，查看該執行框架與帳戶的即時
目錄。

如果探索失敗或逾時，OpenClaw 會使用內附的備援目錄：

| 模型 id       | 顯示名稱 | 推理強度        |
| -------------- | ------------ | ------------------------ |
| `gpt-5.5`      | gpt-5.5      | low, medium, high, xhigh |
| `gpt-5.4-mini` | GPT-5.4-Mini | low, medium, high, xhigh |

<Note>
目前內附的執行框架為 `@openai/codex` `0.145.0`。對該內附 app-server 執行的 `model/list` 探測
傳回下列公開選擇器資料列：

| 模型 id        | 輸入模態 | 推理強度                    |
| --------------- | ---------------- | ------------------------------------ |
| `gpt-5.6-sol`   | 文字、圖片      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-terra` | 文字、圖片      | low, medium, high, xhigh, max, ultra |
| `gpt-5.6-luna`  | 文字、圖片      | low, medium, high, xhigh, max        |
| `gpt-5.5`       | 文字、圖片      | low, medium, high, xhigh             |
| `gpt-5.2`       | 文字、圖片      | low, medium, high, xhigh             |

app-server 目錄可以回報 `ultra`；OpenClaw 推理控制項目前
公開至 `max` 的層級。

即時選擇器資料列會因帳戶而異，且可能隨帳戶、Codex
目錄或內附版本而改變；請執行 `/codex models` 取得目前清單，而非
依賴任何特定時間點的表格。隱藏模型也可能出現在
app-server 目錄中，以供內部或專門流程使用，但不屬於一般
模型選擇器選項。
</Note>

請在 `plugins.entries.codex.config.discovery` 下調整探索：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: true,
            timeoutMs: 2500,
          },
        },
      },
    },
  },
}
```

若希望啟動時避免探測 Codex，並只使用
備援目錄，請停用探索：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          discovery: {
            enabled: false,
          },
        },
      },
    },
  },
}
```

## 工作區啟動載入檔案

Codex 會透過原生專案文件探索自行處理 `AGENTS.md`。
OpenClaw 不會寫入合成的 Codex 專案文件，也不會依賴 Codex
備援檔名作為角色設定檔，因為 Codex 備援僅在
`AGENTS.md` 缺少時才會套用。

為了與 OpenClaw 工作區保持一致，Codex 執行框架會將其他
啟動載入檔案轉送為開發者指示，但方式並不完全相同：

- `TOOLS.md` 會轉送為**繼承的** Codex 開發者指示，因此
  在該回合中產生的原生 Codex 子代理也能看見它。
- `SOUL.md`、`IDENTITY.md` 和 `USER.md` 會轉送為**回合範圍**
  的協作指示。原生 Codex 子代理不會繼承它們，
  以免子代理回合取得父代理的角色設定與
  使用者設定檔。
- 精簡載入的 OpenClaw Skills 清單也會轉送為回合範圍的
  協作開發者指示，因此原生 Codex 子代理也不會
  繼承它。
- 不會注入 `HEARTBEAT.md` 內容；當檔案存在且
  非空白時，心跳偵測回合會收到協作模式指引，以讀取該檔案。
- 當該工作區可使用記憶工具時，設定的代理工作區中的 `MEMORY.md` 內容不會貼入
  原生 Codex 回合輸入；若檔案存在，執行框架會在回合範圍的協作開發者指示中加入簡短的工作區記憶
  指引，而當永久記憶相關時，Codex
  應使用 `memory_search` 或 `memory_get`。
  如果工具已停用、記憶搜尋不可用，或作用中
  工作區不同於代理記憶工作區，`MEMORY.md` 則會使用
  一般的有界回合上下文路徑。
- 若存在 `BOOTSTRAP.md`，則會轉送為 OpenClaw 回合輸入的參考
  上下文。

## 環境覆寫

環境覆寫仍可用於本機測試：

- `OPENCLAW_CODEX_APP_SERVER_BIN`
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

當 `appServer.command` 未設定時，
`OPENCLAW_CODEX_APP_SERVER_BIN` 會略過受管理的二進位檔。

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` 已移除。請改用
`plugins.entries.codex.config.appServer.mode: "guardian"`，或使用
`OPENCLAW_CODEX_APP_SERVER_MODE=guardian` 進行一次性本機測試。若要進行
可重複部署，建議使用設定，因為這能將外掛行為與
Codex 執行框架的其餘設定保存在同一個經過審查的檔案中。

## 相關內容

- [Codex 執行框架](/zh-TW/plugins/codex-harness)
- [Codex 執行框架執行階段](/zh-TW/plugins/codex-harness-runtime)
- [Codex 監督](/zh-TW/plugins/codex-supervision)
- [原生 Codex 外掛](/zh-TW/plugins/codex-native-plugins)
- [Codex 電腦操作](/zh-TW/plugins/codex-computer-use)
- [OpenAI 供應商](/zh-TW/providers/openai)
- [設定參考](/zh-TW/gateway/configuration-reference)
