---
read_when:
    - 新增位置節點支援或權限介面
    - 設計 Android 定位權限或前景行為
summary: 節點的位置命令、平台權限模式，以及 Linux GeoClue 設定
title: 位置指令
x-i18n:
    generated_at: "2026-07-26T08:30:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## 摘要

- `location.get` 是節點命令，透過 `node.invoke` 或 `openclaw nodes location get` 呼叫。
- 預設關閉。
- Android 第三方版本使用選擇器：關閉 / 使用 App 期間 / 永遠。Play 版本仍為關閉 / 使用 App 期間。
- 精確位置是獨立的切換開關。

## 為何使用選擇器（而非只有開關）

作業系統的位置權限分為多個層級。精確位置也是獨立的作業系統授權（iOS 14+ 的「精確」，Android 的「精確」與「約略」）。App 內的選擇器決定要求的模式，但實際授權仍由作業系統決定。

## 設定模型

每個節點裝置：

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

UI 行為：

- 選取 `whileUsing` 會要求前景權限。
- 在 Android 第三方版本中選取 `always`，會先要求前景權限、說明背景存取，接著開啟 Android App 設定，以取得獨立的 **Allow all the time** 授權。
- Android Play 版本不會宣告背景位置權限，也不會顯示 `always`。
- 如果作業系統拒絕要求的層級，App 會還原至已授權的最高層級並顯示狀態。

## 權限對應（node.permissions）

選用。macOS 節點會透過 `node.list`/`node.describe` 上的 `permissions` 對應表回報 `location`；iOS/Android 可能省略此項。

## 命令：`location.get`

透過 `node.invoke` 呼叫，或使用命令列介面輔助工具：

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

參數：

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

命令列介面旗標直接對應：`--location-timeout` -> `timeoutMs`、`--max-age` -> `maxAgeMs`、`--accuracy` -> `desiredAccuracy`。

回應承載資料：

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

錯誤（穩定代碼）：

- `LOCATION_DISABLED`：選擇器已關閉。
- `LOCATION_PERMISSION_REQUIRED`：缺少要求模式所需的權限。
- `LOCATION_BACKGROUND_UNAVAILABLE`：App 正在背景執行，但僅獲得使用 App 期間的授權。
- `LOCATION_TIMEOUT`：未能及時取得位置。
- `LOCATION_UNAVAILABLE`：系統失敗或沒有提供者。

## 背景行為

- 只有在使用者選取 `Always` 且 Android 授予背景位置權限時，Android 第三方版本才會接受背景 `location.get`。現有的常駐節點服務會新增 `location` 服務類型，並在啟用期間揭露 `Location: Always`。
- Android Play 版本和 `While Using` 模式在背景執行時會拒絕 `location.get`。
- 其他節點平台可能有所不同。

## Linux 節點主機

內建的 Linux 節點外掛會將 `location.get` 新增至命令列介面 `openclaw node` 服務，包括未安裝 Linux 桌面 App 的無頭主機。位置功能預設關閉。請在外掛項目下啟用此功能，然後重新啟動節點服務：

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

安裝 GeoClue2 及其 `where-am-i` 示範程式（Debian 和 Ubuntu 上為 `geoclue-2-demo`）。節點服務使用者必須獲得主機 GeoClue 原則和授權代理程式的允許。

此外掛使用 `where-am-i`，而不是依序呼叫多個 `busctl`。GeoClue 會將用戶端建立、屬性、啟動、更新及停止繫結至單一 D-Bus 用戶端連線；示範程式會將這些生命週期維持在一起，而個別的 `busctl` 子程序則不會。不會新增 npm 相依套件。

Linux 會將 `coarse`、`balanced` 和 `precise` 對應至 GeoClue 精確度層級 `4`、`6` 和 `8`。它會根據傳回的時間戳記驗證 `maxAgeMs`。GeoClue 的示範程式不會公開所選的提供者，因此 `source` 為 `unknown`；只有在回報的精確度為 100 公尺或更佳時，`isPrecise` 才為 true。

Linux 使用相同的穩定錯誤：`LOCATION_DISABLED`、`LOCATION_TIMEOUT` 和 `LOCATION_UNAVAILABLE`。

## 模型／工具整合

- 代理工具：`nodes` 工具的 `location_get` 動作（需要節點）。
- 命令列介面：`openclaw nodes location get --node <id>`。
- 代理指引：只有在使用者已啟用位置功能並了解其範圍時才呼叫。

## UX 文案（建議）

- 關閉：“位置分享已停用。”
- 使用 App 期間：“僅限 OpenClaw 開啟時。”
- 永遠：“允許在 OpenClaw 於背景執行時進行要求的位置檢查。”
- 精確：“使用精確的 GPS 位置。關閉此選項即可分享約略位置。”

## 相關內容

- [節點概覽](/zh-TW/nodes)
- [頻道位置解析](/zh-TW/channels/location)
- [相機擷取](/zh-TW/nodes/camera)
- [對話模式](/zh-TW/nodes/talk)
