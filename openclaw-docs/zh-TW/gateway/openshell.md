---
read_when:
    - 你想使用雲端管理的沙箱，而不是本機 Docker
    - 你正在設定 OpenShell 外掛
    - 你需要在鏡像與遠端工作區模式之間做出選擇
summary: 使用 OpenShell 作為 OpenClaw 代理程式的受管理沙箱後端
title: OpenShell
x-i18n:
    generated_at: "2026-07-26T07:20:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bf5c33912bd0db759a01cf58ea26712a8ada68c0804bf16f69f1f7cdd496828c
    source_path: gateway/openshell.md
    workflow: 16
---

OpenShell 是受管理的沙箱後端：OpenClaw 不會在本機執行 Docker 容器，
而是將沙箱生命週期委派給 `openshell` 命令列介面，由其佈建遠端環境並透過 SSH 執行命令。

此外掛會重複使用與通用[SSH 後端](/zh-TW/gateway/sandboxing#ssh-backend)相同的 SSH 傳輸與遠端檔案系統橋接，並加入 OpenShell
生命週期（`sandbox create/get/delete/ssh-config`）及選用的 `mirror`
工作區同步模式。

## 必要條件

- 已安裝 OpenShell 外掛（`openclaw plugins install @openclaw/openshell-sandbox`）
- `openshell` 命令列介面位於 `PATH`（或透過
  `plugins.entries.openshell.config.command` 指定自訂路徑）
- 具備沙箱存取權的 OpenShell 帳號
- OpenClaw 閘道正在主機上執行

## 快速開始

```bash
openclaw plugins install @openclaw/openshell-sandbox
```

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

重新啟動閘道。在下一次代理程式輪次中，OpenClaw 會建立 OpenShell
沙箱，並透過該沙箱路由工具執行。使用以下命令驗證：

```bash
openclaw sandbox list
openclaw sandbox explain
```

## 工作區模式

這是使用 OpenShell 時最重要的決策。

### mirror（預設）

`plugins.entries.openshell.config.mode: "mirror"` 會讓**本機工作區
維持為標準來源**：

- 在 `exec` 之前，OpenClaw 會將本機工作區同步至沙箱。
- 在 `exec` 之後，OpenClaw 會將遠端工作區同步回本機。
- 檔案工具會透過沙箱橋接運作，但在輪次之間，本機仍是資料來源。
  
最適合開發工作流程：在 OpenClaw 外部進行的本機編輯會在下一次執行時出現，而且沙箱的行為與 Docker 後端相近。

取捨：每個執行輪次都有上傳與下載成本。

### remote

`mode: "remote"` 會讓 **OpenShell 工作區成為標準來源**：

- 首次建立沙箱時，OpenClaw 只會從本機植入遠端工作區一次。
- 之後，`exec`、`read`、`write`、`edit` 和 `apply_patch` 會直接對遠端工作區進行操作。OpenClaw **不會**將遠端變更同步回本機。
- 提示詞處理階段的媒體讀取仍可運作（檔案／媒體工具會透過沙箱橋接讀取）。

最適合長時間執行的代理程式與 CI：每輪負擔較低，而且主機上的本機編輯不會在無提示的情況下覆寫遠端狀態。

<Warning>
初始植入後，在 OpenClaw 外部編輯主機上的檔案，遠端沙箱將無法看到這些變更。請執行 `openclaw sandbox recreate` 以重新植入。
</Warning>

### 選擇模式

|                          | `mirror`                   | `remote`                  |
| ------------------------ | -------------------------- | ------------------------- |
| **標準工作區**  | 本機主機                 | 遠端 OpenShell          |
| **同步方向**       | 雙向（每次執行） | 一次性植入             |
| **每輪負擔**    | 較高（上傳與下載） | 較低（直接遠端操作） |
| **可看到本機編輯？** | 是，在下次執行時          | 否，直到重新建立        |
| **最適合**             | 開發工作流程      | 長時間執行的代理程式、CI   |

## 設定參考

所有 OpenShell 設定都位於 `plugins.entries.openshell.config` 下：

| 鍵                       | 類型                     | 預設值       | 說明                                                                            |
| ------------------------- | ------------------------ | ------------- | -------------------------------------------------------------------------------------- |
| `mode`                    | `"mirror"` 或 `"remote"` | `"mirror"`    | 工作區同步模式                                                                    |
| `command`                 | `string`                 | `"openshell"` | `openshell` 命令列介面的路徑或名稱                                                    |
| `from`                    | `string`                 | `"openclaw"`  | 首次建立時的沙箱來源                                                   |
| `gateway`                 | `string`                 | 未設定         | OpenShell 閘道名稱（頂層 `--gateway`）                                         |
| `gatewayEndpoint`         | `string`                 | 未設定         | OpenShell 閘道端點（頂層 `--gateway-endpoint`）                            |
| `policy`                  | `string`                 | 未設定         | 用於建立沙箱的 OpenShell 原則 ID                                               |
| `providers`               | `string[]`               | `[]`          | 建立沙箱時附加的提供者名稱（去除重複項目，每個項目使用一個 `--provider` 旗標） |
| `gpu`                     | `boolean`                | `false`       | 要求 GPU 資源（`--gpu`）                                                        |
| `autoProviders`           | `boolean`                | `true`        | 建立時傳遞 `--auto-providers`（若為 false，則傳遞 `--no-auto-providers`）            |
| `remoteWorkspaceDir`      | `string`                 | `"/sandbox"`  | 沙箱內的主要可寫入工作區                                          |
| `remoteAgentWorkspaceDir` | `string`                 | `"/agent"`    | 代理程式工作區掛載路徑（工作區存取權不是 `rw` 時為唯讀）               |
| `timeoutSeconds`          | `number`                 | `120`         | `openshell` 命令列介面操作的逾時時間                                                 |

`remoteWorkspaceDir` 和 `remoteAgentWorkspaceDir` 必須是絕對路徑，且
必須位於受管理的根目錄 `/sandbox` 或 `/agent` 下；其他絕對路徑會被拒絕。

沙箱層級設定（`mode`、`scope`、`workspaceAccess`）與其他後端相同，位於
`agents.defaults.sandbox` 下。完整矩陣請參閱
[沙箱隔離](/zh-TW/gateway/sandboxing)。

## 範例

### 最小遠端設定

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
        },
      },
    },
  },
}
```

### 使用 GPU 的 mirror 模式

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "agent",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "mirror",
          gpu: true,
          providers: ["openai"],
          timeoutSeconds: 180,
        },
      },
    },
  },
}
```

### 使用自訂閘道的每代理程式 OpenShell

```json5
{
  agents: {
    defaults: {
      sandbox: { mode: "off" },
    },
    list: [
      {
        id: "researcher",
        sandbox: {
          mode: "all",
          backend: "openshell",
          scope: "agent",
          workspaceAccess: "rw",
        },
      },
    ],
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote",
          gateway: "lab",
          gatewayEndpoint: "https://lab.example",
          policy: "strict",
        },
      },
    },
  },
}
```

## 生命週期管理

```bash
# 列出所有沙箱執行階段（Docker + OpenShell）
openclaw sandbox list

# 檢查生效的原則
openclaw sandbox explain

# 重新建立（刪除遠端工作區，並在下次使用時重新植入）
openclaw sandbox recreate --all
```

對 `remote` 模式而言，重新建立尤其重要：它會刪除該範圍的標準遠端工作區，下一次使用時則會從本機植入新的工作區。對 `mirror` 模式而言，因為本機維持為標準來源，重新建立主要是重設遠端執行環境。

變更以下任何項目後，請重新建立：

- `agents.defaults.sandbox.backend`
- `plugins.entries.openshell.config.from`
- `plugins.entries.openshell.config.mode`
- `plugins.entries.openshell.config.policy`

## 安全強化

mirror 模式的檔案系統橋接會固定本機工作區根目錄，並在每次讀取、寫入、建立目錄、移除及重新命名前，重新檢查標準路徑（透過 realpath），拒絕路徑中段的符號連結。交換符號連結或重新掛載工作區，都無法將檔案存取重新導向至鏡像樹狀結構之外。

## 目前限制

- OpenShell 後端不支援沙箱瀏覽器。
- `sandbox.docker.binds` 不適用於 OpenShell；若已設定繫結，建立沙箱會失敗。
- `sandbox.docker.*` 下的 Docker 專用執行階段調整項目（`env` 除外）僅適用於 Docker 後端。

## 運作方式

1. OpenClaw 會針對沙箱名稱執行 `sandbox get`（包括任何已設定的
   `--gateway`/`--gateway-endpoint`）；若失敗，則使用
   `sandbox create` 建立沙箱，並在已設定時傳遞 `--name`、`--from`、`--policy`，在啟用時傳遞 `--gpu`，以及傳遞 `--auto-providers`/`--no-auto-providers`，並為每個已設定的提供者傳遞一個
   `--provider` 旗標。
2. OpenClaw 會針對沙箱名稱執行 `sandbox ssh-config`，以取得 SSH
   連線詳細資料。
3. 核心會將 SSH 設定寫入暫存檔，並透過與通用 SSH 後端相同的遠端檔案系統橋接開啟 SSH 工作階段。
4. 在 `mirror` 模式中：執行前從本機同步至遠端、執行，然後在執行後同步回來。
5. 在 `remote` 模式中：建立時植入一次，之後直接在遠端工作區操作。

## 相關內容

- [沙箱隔離](/zh-TW/gateway/sandboxing) - 模式、範圍與後端比較
- [沙箱、工具原則與提升權限的比較](/zh-TW/gateway/sandbox-vs-tool-policy-vs-elevated) - 偵錯遭封鎖的工具
- [多代理程式沙箱與工具](/zh-TW/tools/multi-agent-sandbox-tools) - 每代理程式覆寫
- [沙箱命令列介面](/zh-TW/cli/sandbox) - `openclaw sandbox` 命令
