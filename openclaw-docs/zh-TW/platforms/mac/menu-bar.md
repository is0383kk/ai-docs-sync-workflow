---
read_when:
    - 調整 Mac 選單介面或狀態邏輯
summary: 選單列狀態邏輯與向使用者顯示的內容
title: 選單列
x-i18n:
    generated_at: "2026-07-26T07:56:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d53cd15109864b88010f41ccf4c46ea7fff6721bc6632630d83a558084cb2d62
    source_path: platforms/mac/menu-bar.md
    workflow: 16
---

## 顯示內容

- 目前的代理程式工作狀態會顯示於選單列圖示及選單的第一個狀態列中。
- 工作進行時會隱藏健康狀態；所有工作階段皆閒置後便會重新顯示。
- 根層級的「內容」項目會開啟包含最近工作階段的子選單，而不會在根選單中展開這些工作階段。
- 根選單中的「節點」區塊只會列出已配對的**裝置**（來自 `node.list`），不包含用戶端／在線狀態項目。
- 提供供應商用量快照時，根層級的「用量」區段會顯示於「內容」下方；若有費用詳細資料，則會接續顯示。
- **快速聊天**會開啟浮動的主要工作階段撰寫器；該項目旁會顯示目前的全域快捷鍵。

## 狀態模型

- 來源：`WorkActivityStore`（`apps/macos/Sources/OpenClaw/WorkActivityStore.swift`）。
- 事件以含有 `runId` 的 `ControlAgentEvent` 傳入；處理常式（`ControlChannel.routeWorkActivity`）會從事件承載資料中讀取 `sessionKey`，若不存在則預設為 `"main"`。
- 優先順序：主要工作階段（預設為 `sessionKey == "main"`）一律優先。主要工作階段處於活動狀態時，會立即顯示其狀態。主要工作階段閒置時，則改為顯示最近處於活動狀態的非主要工作階段。儲存區不會在活動期間切換；只有目前的工作階段轉為閒置，或主要工作階段轉為活動狀態時才會切換。
- 活動種類：
  - `job`：高階命令執行（`state: started|streaming|done|error|...`）。
  - `tool`：具有 `name`，以及選用 `meta`/`args` 的 `phase: start|result`。

## IconState 列舉（Swift）

- `idle`
- `workingMain(ActivityKind)`
- `workingOther(ActivityKind)`
- `overridden(ActivityKind)`（偵錯覆寫）

### ActivityKind -> 徽章符號

`ActivityKind` 會包裝 `ToolKind`（`bash`、`read`、`write`、`edit`、`attach`、`other`）或單獨的 `job`。每種項目都會對應至繪製在小動物圖示（`IconState.badgeSymbolName`）上的 SF Symbol 徽章：

| 種類            | 符號                             |
| --------------- | ---------------------------------- |
| `bash`          | `chevron.left.slash.chevron.right` |
| `read`          | `doc`                              |
| `write`         | `pencil`                           |
| `edit`          | `pencil.tip`                       |
| `attach`        | `paperclip`                        |
| `other` / `job` | `gearshape.fill`                   |

### 視覺對應

- `idle`：一般小動物圖示，無徽章。
- `workingMain`：含符號的徽章、完整色調（`.primary` 顯著度），並播放腿部「工作中」動畫。
- `workingOther`：含符號的徽章、柔和色調（`.secondary` 顯著度），不會快速移動。
- `overridden`：無論實際活動為何，一律使用所選的符號／色調。

## 內容子選單

- 根選單會顯示一列「內容」及工作階段數量／狀態；此列會開啟子選單（`MenuSessionsInjector`）。
- 子選單標頭會顯示過去 24 小時內的活動工作階段數量。
- 每個工作階段列都會保留其權杖列、經過時間、預覽、思考／詳細切換、重設、壓縮及刪除動作。
- 載入中、已中斷連線及工作階段載入錯誤訊息會顯示於「內容」子選單內。
- 用量及費用區段會保留在「內容」下方的根層級，無須開啟子選單即可快速查看。

## 狀態列文字（選單）

- 工作進行時：`<Session role> · <activity label>`（`MenuContentView` 中的 `"\(roleLabel) · \(activity.label)"`），其中角色標籤為 `Main` 或 `Other`。
- 閒置時：改為顯示健康狀態摘要。

## 事件擷取

- 來源：由 `ControlChannel.routeWorkActivity(from:)` 路由的控制通道 `agent` 事件。
- 剖析的欄位：
  - `stream: "job"`，使用 `data.state` 表示開始／停止。
  - `stream: "tool"`，具有 `data.phase`、`data.name`，以及選用的 `data.meta`/`data.args`。
- 工具標籤來自 `ToolDisplayRegistry.resolve(name:args:meta:)`；無法解析的名稱會改用原始工具名稱。

## 偵錯覆寫

- Settings > Debug > "Icon override" 選擇器：
  - `System (auto)`（預設）
  - `Working: main` / `Working: other`（依工具種類：bash、read、write、edit、other）
  - `Idle`
- 儲存於 `UserDefaults` 鍵 `openclaw.iconOverride` 下；對應至 `IconState.overridden`。

## 測試檢查清單

- 觸發主要工作階段工作：圖示會立即切換，狀態列則顯示主要標籤。
- 在主要工作階段閒置時觸發非主要工作階段工作：圖示／狀態會顯示非主要工作階段，且維持穩定直到工作完成。
- 在另一個工作階段處於活動狀態時啟動主要工作階段：圖示會立即切換至主要工作階段。
- 快速連續使用工具：徽章不會閃爍（已完成工具清除前有 2 秒的寬限時間，`WorkActivityStore.toolResultGrace`）。
- 所有工作階段皆閒置後，健康狀態列會重新出現。

## 相關內容

- [macOS 應用程式](/zh-TW/platforms/macos)
- [選單列圖示](/zh-TW/platforms/mac/icon)
