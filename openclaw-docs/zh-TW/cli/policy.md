---
read_when:
    - 你想要依據編寫的 policy.jsonc 檢查 OpenClaw 設定
    - 你希望在 doctor lint 中看到原則檢查結果
    - 你需要政策證明雜湊值作為稽核證據
summary: '`openclaw policy` 一致性檢查的命令列介面參考資料'
title: 政策
x-i18n:
    generated_at: "2026-07-26T07:15:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` 由隨附的 Policy 外掛提供。它是在現有 OpenClaw 設定之上的企業合規層，而非第二套設定系統。你在 `policy.jsonc` 中撰寫要求；OpenClaw 將作用中的工作區觀察結果視為證據；Policy 透過 `doctor --lint` 回報偏離情形。Policy 不會強制執行工具呼叫，也不會在請求時重寫執行階段行為，且不會證明個別代理程式的認證資訊儲存區，例如 `auth-profiles.json`。

Policy 會檢查已設定的頻道、MCP 伺服器、模型供應商、網路 SSRF 防護態勢、輸入流量／頻道存取、閘道暴露情形與節點命令態勢、已撰寫的訊息路由探測、代理程式工作區存取、沙箱態勢、資料處理態勢、秘密供應商／驗證設定檔態勢，以及受治理的工具中繼資料（`TOOLS.md`）。當工作區需要持久且可檢查的聲明時，請使用它，例如「不得啟用 Telegram」或「受治理的工具必須宣告風險與擁有者中繼資料」。如果只需要本機行為，而不需要證明或偏離偵測，使用一般設定即可。

## 快速開始

```bash
openclaw plugins enable policy
```

即使缺少 `policy.jsonc`，此外掛仍會保持啟用，讓 doctor 能回報缺少的成品，而不是默默略過檢查。

請手動撰寫 `policy.jsonc`；它不會根據目前設定產生。每個頂層區段都是規則命名空間：只有在其中存在具體規則時才會執行檢查（不支援的區段或鍵會以 `policy/policy-jsonc-invalid` 失敗，而不是被默默忽略）。涵蓋所有支援區段的最小範例：

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "此工作區未核准使用 Telegram。",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

下方規則表中不易看出的跨領域注意事項：

- 在拒絕非回送繫結時省略 `gateway.bind`，表示你接受執行階段預設值；若要嚴格符合規範，請設定 `gateway.bind: "loopback"`。
- 若代理程式為唯讀，請在適用的預設值／代理程式上，將沙箱 `mode` 設為 `all` 或 `non-main`，並將 `workspaceAccess` 設為 `none` 或 `ro`。缺少沙箱模式或將其設為 `off`，均不符合唯讀 Policy。
- `agents.workspace.denyTools` 接受 `exec`、`process`、`write`、`edit`、`apply_patch`。設定中的工具拒絕群組 `group:fs`（檔案異動）與 `group:runtime`（Shell／程序）可滿足同等態勢。
- 只有存在 `execApprovals` 規則時，執行核准檢查才會讀取即時的 `exec-approvals.json` 成品；缺少或無效的成品屬於無法觀察的證據，而非模擬的通過結果。
- 秘密與驗證設定檔證據只會記錄供應商／來源態勢及 SecretRef 中繼資料，絕不記錄原始值。Policy 不會讀取或證明個別代理程式的認證資訊儲存區，例如 `auth-profiles.json`。
- 資料處理證據僅代表設定層級的態勢（遮蔽模式、遙測擷取切換、工作階段維護模式、逐字稿索引設定）。它不會檢查記錄、遙測匯出、逐字稿或記憶檔案，而乾淨的結果也無法證明其中不存在個人資料或秘密。
- 路由探測會重複使用 OpenClaw 的執行階段繫結解析器。路由證據只會記錄探測 ID、解析出的代理程式、比對種類，以及已遮蔽的繫結中繼資料。它絕不記錄對等方、帳號、社群、團隊或角色識別碼。新增路由區段會刻意變更 Policy 與證明雜湊；未包含路由的 Policy 會保留現有證據形態。

### Policy 規則參考

下方每條規則皆為選用；只有在規則存在時才會執行檢查。觀察到的狀態來自現有 OpenClaw 設定或工作區中繼資料。

#### 範圍覆寫

當特定代理程式或頻道所需的 Policy 比頂層基準更嚴格時，請使用 `scopes.<scopeName>`。範圍名稱只是標籤；比對會使用範圍內的選取器。覆寫採累加方式：全域規則仍會執行，而範圍規則可以針對相同證據新增自己的發現。

| 選取器     | 支援的區段                                                             | 使用時機                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`、`agents.workspace`、`sandbox`、`dataHandling.memory`、`execApprovals` | 一或多個執行階段代理程式需要更嚴格的規則。   |
| `channelIds` | `ingress.channels`                                                             | 一或多個頻道需要更嚴格的輸入流量規則。 |

如果 `agentIds` 項目不存在於 `agents.entries.*` 中，OpenClaw 會針對該執行階段代理程式 ID 所繼承的全域／預設態勢評估範圍規則，而不會略過。

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

如上所示，如果每個範圍治理不同欄位，同一個代理程式可以出現在多個範圍中。同一代理程式若重複宣告相同的範圍欄位，其限制程度必須相同或更嚴格；較寬鬆的重複宣告會遭拒絕（允許清單必須是子集、拒絕清單必須是超集、必要布林值固定不變）。

容器態勢規則（`sandbox.containers.*`）只會根據相符代理程式的沙箱後端可公開的證據進行檢查。如果後端無法觀察你為其啟用的規則，Policy 會回報 `policy/sandbox-container-posture-unobservable`，而非判定通過；請將容器規則限定於使用可公開相關證據之後端的代理程式群組。

頂層 `ingress.session.requireDmScope` 維持全域；`session.dmScope` 並非可歸因於頻道的證據，因此無法依 `channelIds` 限定範圍。

`policy.jsonc` 中存在的每個範圍都必須有效且可強制執行。

#### 頻道

| Policy 欄位                         | 觀察到的狀態                          | 使用時機                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | `channels.*` 供應商與啟用狀態 | 拒絕來自 `telegram` 等供應商的已設定頻道。 |
| `channels.denyRules[].reason`        | 發現訊息與修復提示內容 | 說明拒絕該供應商的原因。                          |

#### MCP 伺服器

| Policy 欄位        | 觀察到的狀態      | 使用時機                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | `mcp.servers.*` ID | 要求每個已設定的 MCP 伺服器都必須位於允許清單中。 |
| `mcp.servers.deny`  | `mcp.servers.*` ID | 拒絕特定的已設定 MCP 伺服器 ID。                   |

#### 模型供應商

| Policy 欄位             | 觀察到的狀態                                   | 使用時機                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | `models.providers.*` ID 與所選模型參照 | 要求已設定的供應商與所選模型參照使用已核准的供應商。 |
| `models.providers.deny`  | `models.providers.*` ID 與所選模型參照 | 依供應商 ID 拒絕已設定的供應商與所選模型參照。               |

#### 網路

| 原則欄位                   | 觀察到的狀態                      | 適用情況                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | 私人網路 SSRF 逃逸機制 | 設為 `false`，以要求私人網路存取保持停用。 |

#### 訊息路由

| 原則欄位                        | 觀察到的狀態                                      | 適用情況                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | 頻道路由繫結，不包括 ACP 繫結      | 要求至少一個訊息路由繫結。                          |
| `routing.requireConfiguredChannels` | 繫結頻道 ID 與已設定的 `channels.*` ID | 偵測過期或拼寫錯誤的繫結頻道 ID。                        |
| `routing.probes[].route`            | 公開的 OpenClaw 路由解析器                  | 描述代表性的傳入路由，而不傳送訊息。     |
| `routing.probes[].expect.agentId`   | 已解析的代理程式 ID                                   | 要求路由抵達經審查的代理程式。                         |
| `routing.probes[].expect.matchedBy` | 解析器比對種類                                 | 要求對等端、帳號、頻道或其他經審查的繫結明確程度。 |

探測 ID 必須唯一。路由支援 `channel`、選用的 `accountId`、
`peer`、`parentPeer`、`guildId`、`teamId` 和 `memberRoleIds`。對等端種類包括
`direct`、`group` 和 `channel`。`matchedBy` 可包含一或多個執行階段
比對種類，包括 `binding.peer`、`binding.account`、`binding.channel`
或 `default`。

路由檢查僅為符合性檢查。它們不會變更啟動、
訊息傳遞、繫結優先順序或後援行為。發現的問題需要
由操作人員審查，因為自動變更繫結可能會重新導向
私人訊息。

#### 輸入與頻道存取

| 原則欄位                              | 觀察到的狀態                                                 | 適用情況                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | 要求經審查的私人訊息隔離範圍。                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` 和舊版頻道私人訊息原則欄位      | 僅允許經審查的私人訊息頻道原則。               |
| `ingress.channels.denyOpenGroups`         | 頻道、帳號和群組輸入原則                     | 拒絕已設定頻道和帳號的開放群組輸入。      |
| `ingress.channels.requireMentionInGroups` | 頻道、帳號、群組、伺服器和巢狀提及閘門設定 | 當群組輸入為開放或受提及限制時，要求使用提及閘門。 |

#### 閘道

| 原則欄位                            | 觀察到的狀態                                 | 適用情況                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | 設為 `false`，以要求繫結至回送介面的閘道。                                  |
| `gateway.exposure.allowTailscaleFunnel` | Tailscale serve/funnel 閘道安全態勢         | 設為 `false`，以拒絕 Tailscale Funnel 暴露。                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | 設為 `true`，以拒絕停用閘道驗證。                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | 設為 `true`，以要求明確的驗證速率限制設定。                            |
| `gateway.controlUi.allowInsecure`       | 控制介面的不安全驗證／裝置／來源切換選項 | 設為 `false`，以拒絕不安全的控制介面暴露切換選項。                         |
| `gateway.remote.allow`                  | 遠端閘道模式／設定                     | 設為 `false`，以拒絕遠端閘道模式。                                          |
| `gateway.http.denyEndpoints`            | 閘道 HTTP API 端點                     | 拒絕 `chatCompletions` 或 `responses` 等端點 ID。                          |
| `gateway.http.requireUrlAllowlists`     | 閘道 HTTP URL 擷取輸入                  | 設為 `true`，以要求 URL 擷取輸入使用 URL 允許清單。                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | 要求 OpenClaw 設定明確拒絕 `system.run` 等節點命令 ID。 |

`gateway.nodes.denyCommands` 是精確且區分大小寫的原則拒絕超集合規則。
當原則必須證明 OpenClaw 設定明確
拒絕特殊權限節點命令時，請使用此規則。有意允許特殊權限
節點命令的部署應在審查後更新 `policy.jsonc`，而非僅依賴
`gateway.nodes.commands.allow`。

#### 代理程式工作區

| 原則欄位                     | 觀察到的狀態                                                                           | 適用情況                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` 和 `agents.entries.*.sandbox.workspaceAccess` | 僅允許 `none` 或 `ro` 等沙箱工作區存取值。                       |
| `agents.workspace.denyTools`     | 全域和各代理程式的工具拒絕設定                                                    | 要求拒絕變更工具（`exec`、`process`、`write`、`edit`、`apply_patch`）。 |

#### 沙箱安全態勢

| 原則欄位                                          | 觀察到的狀態                                          | 適用情況                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` 和各代理程式模式       | 僅允許 `all` 或 `non-main` 等經審查的沙箱模式。 |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` 和各代理程式後端 | 僅允許 `docker` 等經審查的沙箱後端。         |
| `sandbox.containers.denyHostNetwork`                  | 容器式沙箱／瀏覽器網路模式           | 拒絕主機網路模式。                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | 容器式沙箱／瀏覽器網路模式           | 拒絕加入其他容器的網路命名空間。              |
| `sandbox.containers.requireReadOnlyMounts`            | 容器式沙箱／瀏覽器掛載模式             | 要求掛載為唯讀。                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | 容器式沙箱／瀏覽器掛載目標          | 拒絕掛載容器執行階段通訊端。                          |
| `sandbox.containers.denyUnconfinedProfiles`           | 容器安全設定檔態勢                      | 拒絕未受限制的容器安全設定檔。                   |
| `sandbox.browser.requireCdpSourceRange`               | 沙箱瀏覽器 CDP 來源範圍                        | 要求瀏覽器 CDP 暴露宣告來源範圍。        |

原則會將缺少的 `sandbox.mode` 視為其隱含預設值 `off`，因此
`sandbox.requireMode` 會將全新或未設定的沙箱回報為不在
`["all"]` 等允許清單內。

#### 資料處理

| 原則欄位                                        | 觀察到的狀態                                                                                     | 適用情況                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | 設為 `true`，以拒絕 `logging.redactSensitive: "off"`。              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | 設為 `true`，以拒絕遙測內容擷取。                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | 設為 `true`，以要求有效的工作階段維護模式為 `enforce`。 |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`、`memory.search.experimental.sessionMemory` 和各代理程式覆寫 | 設為 `true`，以拒絕將工作階段轉錄內容索引至記憶體。       |

#### 密鑰

| 原則欄位                      | 觀察到的狀態                                           | 適用情況                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | 設定 SecretRef 和 `secrets.providers.*` 宣告 | 設為 `true`，以要求 SecretRef 指向已宣告的提供者。     |
| `secrets.denySources`             | 密鑰提供者來源和 SecretRef 來源            | 拒絕 `exec`、`file` 或其他已設定的來源名稱。 |
| `secrets.allowInsecureProviders`  | 不安全的密鑰提供者態勢旗標                   | 設為 `false`，以拒絕選擇不安全態勢的提供者。      |

#### 執行核准

執行核准檢查會讀取執行階段 `exec-approvals.json` 成品：
預設為 `~/.openclaw/exec-approvals.json`，或在設定 `OPENCLAW_STATE_DIR` 時使用
`$OPENCLAW_STATE_DIR/exec-approvals.json`。
`execApprovals.defaults.*` 或 `execApprovals.agents.*` 下的安全態勢規則
要求可讀取的成品證據；缺少或無效的成品會回報為
無法觀察的證據，而非以盡力而為方式判定通過。可讀取後，省略的
欄位會繼承執行階段預設值：缺少的 `defaults.security` 為 `full`，而
缺少的代理程式安全性會繼承該預設值。證據包括 `defaults`、
`agents.*`、`agents.*.allowlist[].pattern`、選用的 `argPattern`、有效的
`autoAllowSkills` 安全態勢及項目來源，但絕不包括通訊端路徑／權杖、
`commandText`、`lastUsedCommand`、已解析路徑或時間戳記。

| 原則欄位                                | 觀察到的狀態                                                                         | 適用情境                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | 使用中的執行階段 `exec-approvals.json` 路徑                                              | 設為 `true`，要求核准成品必須存在且可剖析。                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`，預設為 `full`                                              | 僅允許已核准的預設核准安全模式。                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`，繼承預設值                                               | 僅允許已核准的個別代理程式有效核准安全模式。                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` 和 `agents.*.autoAllowSkills`，繼承執行階段預設值 | 設為 `false`，要求使用嚴格的手動允許清單，不隱含核准技能命令列介面。 |
| `execApprovals.agents.allowlist.expected`   | 彙總的 `agents.*.allowlist[]` 模式和選用的 argPattern 項目               | 要求核准允許清單符合已審查的模式集。                      |

範例：要求核准成品、拒絕寬鬆的預設值，並且僅允許
所選代理程式採用已審查的執行核准安全態勢。

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // 安全模式："deny"、"allowlist" 或 "full"。
      // 此預設值僅允許鎖定的拒絕態勢。
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // 所選代理程式可使用已審查的允許清單態勢，但不可使用 "full"。
          "allowSecurity": ["allowlist"],
          // false 表示技能命令列介面必須出現在已審查的允許清單中，而非
          // 由 autoAllowSkills 隱含核准。
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // 簡單項目：完全相符的已審查可執行檔模式，且不含 argPattern。
              "travel-hub",
              // 受限項目：模式加上已審查的引數規則運算式。
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### 驗證設定檔

| 原則欄位                    | 觀察到的狀態                               | 適用情境                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | `auth.profiles.*` 提供者和模式中繼資料 | 要求設定驗證設定檔具有 `provider` 和 `mode` 等中繼資料鍵。               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | 僅允許支援的驗證設定檔模式，例如 `api_key`、`aws-sdk`、`oauth` 或 `token`。 |

#### 工具中繼資料

| 原則欄位            | 觀察到的狀態                   | 適用情境                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | 受治理的 `TOOLS.md` 宣告 | 要求受治理的工具宣告 `risk`、`sensitivity` 或 `owner` 等中繼資料鍵。 |

#### 工具安全態勢

| 原則欄位                    | 觀察到的狀態                                              | 適用情境                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` 和 `agents.entries.*.tools.profile`        | 僅允許 `minimal`、`messaging` 或 `coding` 等工具設定檔 ID。                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` 和個別代理程式的 `tools.fs` 覆寫值 | 設為 `true`，要求檔案系統工具採用僅限工作區的安全態勢。                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` 和個別代理程式的執行安全性           | 僅允許 `deny` 或 `allowlist` 等執行安全模式。                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` 和個別代理程式的執行詢問模式                | 要求採用 `always` 等核准安全態勢。                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` 和個別代理程式的執行主機路由           | 僅允許 `sandbox` 等執行主機路由模式。                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` 和個別代理程式的提升權限安全態勢     | 設為 `false`，要求提升權限工具模式維持停用。                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` 和個別代理程式的 `tools.alsoAllow`           | 要求完全符合 `alsoAllow` 項目，並回報缺少或非預期的附加工具授權。                 |
| `tools.denyTools`               | `tools.deny` 和 `agents.entries.*.tools.deny`              | 要求已設定的工具拒絕清單包含 `group:runtime` 和 `group:fs` 等工具 ID 或群組。 |

## 執行檢查

撰寫期間執行僅限原則的檢查：

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` 僅執行原則檢查集，並輸出證據、發現項目
和證明雜湊。啟用原則外掛後，相同的發現項目也會出現在
`openclaw doctor --lint` 中。

將操作者原則檔案與撰寫的基準進行比較：

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` 會依據原則檔案語法檢查原則檔案語法；它
不會檢查執行階段狀態、證據、認證資訊或祕密。它使用管理具範圍覆疊的相同
規則中繼資料：允許清單必須維持相同或更窄，拒絕清單必須維持
相同或更廣，必要的布林值必須保持其值，已排序的字串只能朝
所設定順序中更嚴格的一端移動，而完全符合清單必須相符。基準可以是
組織撰寫的原則；受檢查的原則可以加入更嚴格的值或
額外規則。當頂層受檢查規則具有相同或更嚴格的限制時，
可以滿足具範圍的基準規則。檔案之間的範圍名稱不必相符；
比較是以選取器（`agentIds`/`channelIds`）和欄位為鍵。
對於路由探查，每個基準探查 ID 都必須保留相同的路由
和預期代理程式。受檢查的原則可以加入探查或縮窄 `matchedBy`，但
移除探查、變更其路由或代理程式，或放寬其接受的比對
種類，都會減弱限制。

乾淨的比較（`--json`）：

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

乾淨的 `policy check --json` 輸出包含操作者或
監督者可記錄的穩定雜湊：

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## 設定原則

原則設定位於 `plugins.entries.policy.config` 之下。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| 設定                   | 用途                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | 即使 `policy.jsonc` 尚不存在，也啟用原則檢查。         |
| `workspaceRepairs`        | 允許 `doctor --fix` 編輯由原則管理的工作區設定。 |
| `expectedHash`            | 已核准原則成品的選用雜湊鎖定。            |
| `expectedAttestationHash` | 上次接受之乾淨原則檢查的選用雜湊鎖定。    |
| `path`                    | 原則成品相對於工作區的位置。             |

將 `plugins.entries.policy.config.enabled` 設為 `false`，可在保留外掛安裝的同時，
停用工作區的原則檢查。

## 接受原則狀態

JSON 輸出範例：

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` 識別編寫的規則成品。`evidence`
記錄檢查所使用的已觀察 OpenClaw 狀態，而
`workspace.hash` 識別該證據酬載。`findingsHash` 識別
確切的發現結果集。`checkedAt` 記錄檢查的執行時間。
`attestationHash` 識別穩定宣告（政策雜湊、證據雜湊、
發現結果雜湊及乾淨／非乾淨狀態），並刻意排除 `checkedAt`，
因此相同的政策狀態一律會產生相同的證明雜湊。這
四個值共同構成一次政策檢查的稽核四元組。

如果閘道或監督程式使用政策來封鎖、核准或註記
執行階段動作，應記錄上一次乾淨檢查的證明雜湊。
`checkedAt` 會保留在 JSON 輸出中供稽核日誌使用，但不屬於
穩定雜湊的一部分。

接受政策狀態的生命週期：

1. 編寫或審查 `policy.jsonc`。
2. 執行 `openclaw policy check --json`。
3. 若檢查乾淨，將 `attestation.policy.hash` 記錄為 `expectedHash`。
4. 將 `attestation.attestationHash` 記錄為 `expectedAttestationHash`。
5. 在 CI 或發布閘門中重新執行 `openclaw doctor --lint`。

如果有意變更政策規則，請使用一次乾淨檢查的結果更新兩個
已接受的雜湊。如果只有工作區設定變更（政策維持不變），
通常只有 `expectedAttestationHash` 會變更。

啟用或升級 `agents.workspace` 規則會將 `agentWorkspace` 證據
加入工作區雜湊和證明雜湊；啟用後請審查新證據，並
重新整理已接受的證明雜湊。啟用或升級
工具態勢規則也會以相同方式加入 `toolPosture` 證據。

`openclaw policy watch` 會重新執行檢查，並在目前證據不再
符合 `expectedAttestationHash` 時回報：

```bash
openclaw policy watch --json
```

在需要單次漂移評估的 CI 或指令碼中使用 `--once`。若未指定
`--once`，預設每兩秒輪詢一次；使用 `--interval-ms` 變更
間隔。

## 發現結果

| 檢查 ID                                                 | 發現事項                                                                           |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | 原則已啟用，但缺少 `policy.jsonc`。                                  |
| `policy/policy-jsonc-invalid`                            | 無法剖析原則，或其中包含格式錯誤的規則項目。                       |
| `policy/policy-hash-mismatch`                            | 原則與已設定的 `expectedHash` 不相符。                                  |
| `policy/attestation-hash-mismatch`                       | 目前的原則證據已不再與已接受的證明相符。               |
| `policy/policy-conformance-invalid`                      | 基準或受檢原則檔案的比較語法無效。                  |
| `policy/policy-conformance-missing`                      | 受檢原則檔案缺少基準原則檔案所要求的規則。     |
| `policy/policy-conformance-weaker`                       | 受檢原則檔案中的值弱於基準原則檔案。           |
| `policy/channels-denied-provider`                        | 已啟用的頻道符合頻道拒絕規則。                                   |
| `policy/mcp-denied-server`                               | 已設定的 MCP 伺服器遭原則拒絕。                                      |
| `policy/mcp-unapproved-server`                           | 已設定的 MCP 伺服器不在允許清單中。                                 |
| `policy/models-denied-provider`                          | 已設定的模型提供者或模型參照使用遭拒絕的提供者。                  |
| `policy/models-unapproved-provider`                      | 已設定的模型提供者或模型參照不在允許清單中。                |
| `policy/network-private-access-enabled`                  | 原則拒絕私人網路 SSRF 逃生機制時，該機制仍處於啟用狀態。             |
| `policy/routing-bindings-required`                       | 原則要求頻道路由繫結，但未設定任何繫結。                  |
| `policy/routing-binding-channel-unconfigured`            | 路由繫結指定了 `channels.*` 中不存在的頻道。                         |
| `policy/routing-agent-mismatch`                          | 編寫的路由解析至不同的代理程式。                                  |
| `policy/routing-match-kind-mismatch`                     | 編寫的路由以非預期的繫結明確程度相符。                   |
| `policy/ingress-dm-policy-unapproved`                    | 頻道私訊原則不在原則允許清單中。                              |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` 與原則要求的私訊隔離範圍不相符。          |
| `policy/ingress-open-groups-denied`                      | 頻道群組原則為 `open`，但原則拒絕開放群組輸入。          |
| `policy/ingress-group-mention-required`                  | 頻道或群組項目停用了提及閘門，但原則要求啟用。       |
| `policy/gateway-non-loopback-bind`                       | 原則拒絕非回送介面暴露時，閘道繫結態勢卻允許此類暴露。         |
| `policy/gateway-auth-disabled`                           | 原則要求驗證時，閘道驗證卻處於停用狀態。                     |
| `policy/gateway-rate-limit-missing`                      | 原則要求明確設定閘道驗證速率限制態勢時，該態勢並未明確設定。          |
| `policy/gateway-control-ui-insecure`                     | 閘道控制介面的不安全暴露切換項已啟用。                         |
| `policy/gateway-tailscale-funnel`                        | 原則拒絕閘道 Tailscale Funnel 暴露時，該暴露仍處於啟用狀態。               |
| `policy/gateway-remote-enabled`                          | 原則拒絕閘道遠端模式時，該模式仍處於作用中。                              |
| `policy/gateway-http-endpoint-enabled`                   | 原則拒絕某個閘道 HTTP API 端點，但該端點仍處於啟用狀態。                    |
| `policy/gateway-http-url-fetch-unrestricted`             | 閘道 HTTP URL 擷取輸入缺少必要的 URL 允許清單。                      |
| `policy/gateway-node-command-denied`                     | 原則拒絕的節點命令未遭 OpenClaw 設定拒絕。                 |
| `policy/agents-workspace-access-denied`                  | 代理程式沙箱模式或工作區存取不在原則允許清單中。           |
| `policy/agents-tool-not-denied`                          | 代理程式或預設設定未拒絕原則要求拒絕的工具。               |
| `policy/tools-profile-unapproved`                        | 已設定的全域或各代理程式工具設定檔不在允許清單中。           |
| `policy/tools-fs-workspace-only-required`                | 檔案系統工具未設定為僅限工作區的路徑態勢。             |
| `policy/tools-exec-security-unapproved`                  | 執行安全模式不在原則允許清單中。                               |
| `policy/tools-exec-ask-unapproved`                       | 執行詢問模式不在原則允許清單中。                                    |
| `policy/tools-exec-host-unapproved`                      | 執行主機路由不在原則允許清單中。                                |
| `policy/tools-elevated-enabled`                          | 原則拒絕提升權限的工具模式時，該模式仍處於啟用狀態。                              |
| `policy/tools-also-allow-missing`                        | 已設定的 `alsoAllow` 清單缺少原則要求的項目。             |
| `policy/tools-also-allow-unexpected`                     | 已設定的 `alsoAllow` 清單包含原則未預期的項目。           |
| `policy/tools-required-deny-missing`                     | 全域或各代理程式工具拒絕清單未包含必要的遭拒工具。     |
| `policy/sandbox-mode-unapproved`                         | 沙箱模式不在原則允許清單中。                                     |
| `policy/sandbox-backend-unapproved`                      | 沙箱後端不在原則允許清單中。                                  |
| `policy/sandbox-container-posture-unobservable`          | 容器態勢規則已針對無法觀察該規則的後端啟用。         |
| `policy/sandbox-container-host-network-denied`           | 容器支援的沙箱或瀏覽器使用主機網路模式。                     |
| `policy/sandbox-container-namespace-join-denied`         | 容器支援的沙箱或瀏覽器加入另一個容器命名空間。          |
| `policy/sandbox-container-mount-mode-required`           | 容器支援的沙箱或瀏覽器掛載並非唯讀。                     |
| `policy/sandbox-container-runtime-socket-mount`          | 容器支援的沙箱或瀏覽器掛載暴露了容器執行階段通訊端。 |
| `policy/sandbox-container-unconfined-profile`            | 原則拒絕無限制的容器沙箱設定檔時，該設定檔卻未受限制。                    |
| `policy/sandbox-browser-cdp-source-range-missing`        | 原則要求沙箱瀏覽器 CDP 來源範圍時，卻缺少該範圍。             |
| `policy/data-handling-redaction-disabled`                | 原則要求敏感記錄遮蔽時，該功能卻處於停用狀態。                  |
| `policy/data-handling-telemetry-content-capture`         | 原則拒絕遙測內容擷取時，該功能仍處於啟用狀態。                       |
| `policy/data-handling-session-retention-not-enforced`    | 原則要求工作階段保留維護時，該要求並未強制執行。            |
| `policy/data-handling-session-transcript-memory-enabled` | 原則拒絕工作階段文字記錄記憶索引時，該功能仍處於啟用狀態。              |
| `policy/secrets-unmanaged-provider`                      | 設定中的 SecretRef 參照了未在 `secrets.providers` 下宣告的提供者。  |
| `policy/secrets-denied-provider-source`                  | 設定中的祕密提供者或 SecretRef 使用原則拒絕的來源。             |
| `policy/secrets-insecure-provider`                       | 原則拒絕不安全態勢時，祕密提供者卻選擇採用該態勢。               |
| `policy/auth-profile-invalid-metadata`                   | 設定中的驗證設定檔缺少有效的提供者或模式中繼資料。                 |
| `policy/auth-profile-unapproved-mode`                    | 設定中的驗證設定檔模式不在原則允許清單中。                       |
| `policy/exec-approvals-missing`                          | 原則要求 `exec-approvals.json`，但缺少該成品。               |
| `policy/exec-approvals-invalid`                          | 無法剖析已設定的執行核准成品。                          |
| `policy/exec-approvals-default-security-unapproved`      | 執行核准預設值使用不在原則允許清單中的安全模式。          |
| `policy/exec-approvals-agent-security-unapproved`        | 各代理程式的有效執行核准安全模式不在允許清單中。       |
| `policy/exec-approvals-auto-allow-skills-enabled`        | 原則拒絕隱含自動允許 Skills 命令列介面時，執行核准代理程式卻如此設定。   |
| `policy/exec-approvals-allowlist-missing`                | 核准允許清單缺少原則要求的模式。                  |
| `policy/exec-approvals-allowlist-unexpected`             | 核准允許清單包含原則未預期的模式。                |
| `policy/tools-missing-risk-level`                        | 受管轄的工具宣告缺少風險中繼資料。                             |
| `policy/tools-unknown-risk-level`                        | 受管轄的工具宣告使用未知的風險值。                           |
| `policy/tools-missing-sensitivity-token`                 | 受管轄的工具宣告缺少敏感度中繼資料。                      |
| `policy/tools-missing-owner`                             | 受管轄的工具宣告缺少擁有者中繼資料。                            |
| `policy/tools-unknown-sensitivity-token`                 | 受管轄的工具宣告使用未知的敏感度值。                    |

一項發現可以同時包含 `target`（觀察到但不符合規範的工作區項目）
與 `requirement`（導致產生該發現的編寫規則）。
目前兩者都是 `oc://` 位址字串，但欄位名稱描述的是原則角色，
而非位址格式。

發現範例：

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "頻道 'telegram' 使用遭拒絕的提供者 'telegram'。",
  "source": "policy",
  "path": "openclaw 設定",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "此工作區未核准 Telegram。"
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "TOOLS.md 工具 'deploy' 沒有明確的風險分類。",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "MCP 伺服器 'remote' 不在原則允許清單中。",
  "source": "policy",
  "path": "openclaw 設定",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "模型參照 'anthropic/claude-sonnet-4.7' 使用未核准的提供者 'anthropic'。",
  "source": "policy",
  "path": "openclaw 設定",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "網路設定 'browser-private-network' 允許私人網路存取。",
  "source": "policy",
  "path": "openclaw 設定",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "閘道繫結設定 'gateway-bind' 允許對非迴路位址公開。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "閘道節點命令 'system.run' 已被原則拒絕，但未被 OpenClaw 設定拒絕。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "將 'system.run' 新增至 gateway.nodes.commands.deny，或在審查後更新原則。"
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "原則不允許 agents.defaults 沙箱 workspaceAccess 使用 'rw'。",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## 修復

`doctor --lint` 和 `policy check` 為唯讀。

只有在明確啟用 `workspaceRepairs` 時，`doctor --fix` 才會編輯由原則管理的工作區設定；否則，檢查只會報告其
將修復的內容，並保持設定不變。

在此版本中，修復功能可以停用被 `channels.denyRules` 拒絕的頻道，並
套用下列自動縮限修復。請僅在審查原則檔案後啟用 `workspaceRepairs`，
因為有效規則可能會變更工作區設定：

- 當全域原則禁止提升權限的工具時，設定 `tools.elevated.enabled=false`
- 當原則要求拒絕特定工具時，將缺少的必要拒絕工具 ID 新增至 `tools.deny` 或
  `agents.entries.*.tools.deny`
- 將不安全的 `gateway.controlUi.*` 切換項目設為 `false`
- 當原則拒絕遠端閘道模式時，設定 `gateway.mode=local`
- 當原則拒絕閘道 HTTP API 端點時，將回報的 `gateway.http.endpoints.*.enabled` 路徑設為 `false`
- 當原則拒絕開放群組輸入時，將回報的頻道輸入 `groupPolicy` 路徑設為 `allowlist`
- 當原則要求群組提及時，將回報的頻道輸入 `requireMention` 路徑設為 `true`
- 當原則要求遮蔽敏感記錄內容時，設定 `logging.redactSensitive=tools`
- 當原則拒絕擷取遙測內容時，設定 `diagnostics.otel.captureContent=false`，或
  針對物件形式的遙測擷取設定，設定 `diagnostics.otel.captureContent.enabled=false`

限定範圍的提升權限工具修復僅供偵測。當問題報告共用的記錄或遙測設定時，
也會略過限定範圍的資料處理修復，因為變更共用設定將影響
限定範圍原則目標以外的項目。

當問題報告繼承的根層級 `tools.deny` 時，會略過限定範圍的必要拒絕修復，
因為將必要工具新增至根層級設定會影響限定範圍原則目標以外的項目。
代理程式本機的必要拒絕修復可以更新所回報的 `agents.entries.*.tools.deny` 路徑。

當問題報告繼承的 `channels.defaults.*` 時，會略過限定範圍的頻道輸入修復，
因為變更共用頻道預設值會影響限定範圍原則目標以外的項目。
閘道 HTTP URL 擷取允許清單問題仍需手動處理，因為自動修復無法選擇正確的端點 URL
允許清單值。

閘道繫結和節點命令問題仍需要審查。當
`policy/gateway-non-loopback-bind` 或 `policy/gateway-node-command-denied`
可對應至設定路徑時，`doctor --fix` 會將建議的
`gateway.bind` 或 `gateway.nodes.commands.deny` 變更回報為已略過的預覽
指引。它不會套用變更，而且在操作人員審查並更新設定或原則前，
該問題不會計為已修復。

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## 結束代碼

| 命令          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | 沒有達到門檻的問題。                          | 一或多個問題達到門檻。                             | 引數或執行階段失敗。 |
| `policy compare` | 原則檔案至少與基準同樣嚴格。 | 原則檔案無效、遺失，或比基準規則寬鬆。 | 引數或執行階段失敗。 |
| `policy watch`   | 沒有問題，且已接受的雜湊為最新狀態。              | 存在問題，或已接受的證明已過期。                    | 引數或執行階段失敗。 |

## 相關內容

- [Doctor lint 模式](/zh-TW/cli/doctor#lint-mode)
- [路徑命令列介面](/zh-TW/cli/path)
