---
read_when:
    - 調查提及缺少 __name 輔助函式的 tsx/esbuild 載入器當機問題
summary: 歷史上的 Node + tsx「__name is not a function」當機及其原因
title: 節點 + tsx 當機
x-i18n:
    generated_at: "2026-07-26T07:18:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# Node + tsx「\_\_name is not a function」當機

## 狀態

已解決。此當機問題無法在目前固定於
`package.json` 的 `tsx` 版本（`4.22.3`）或目前的 Node 版本上重現。保留此頁，以備未來
`tsx`/esbuild 升級再次引入此問題。

## 原始症狀

透過 `tsx` 執行 OpenClaw 開發指令碼時，啟動階段發生以下錯誤：

```text
[openclaw] 無法啟動命令列介面：TypeError: __name is not a function
    at createSubsystemLogger (src/logging/subsystem.ts)
    at <caller> (src/agents/auth-profiles/constants.ts)
```

此處省略行號；自原始當機發生以來，這兩個檔案都已變更，
特定行號已不再相符。

這個問題出現在開發指令碼從 Bun 切換至 `tsx`（`2871657e`，
2026-01-06）以讓 Bun 成為選用項目之後。等效的 Bun 路徑並未當機。
此問題最初是在 macOS 的 Node v25.3.0 上觀察到；其他執行
Node 25 的平台也被認為可能受到影響。

## 原因

`tsx` 透過 esbuild 轉換 TS/ESM，並在其轉換選項中硬式編碼
`keepNames: true`。此設定會讓 esbuild 將具名函式／類別
宣告包裝在對 `__name` 輔助函式的呼叫中，讓 `fn.name` 在縮小
與打包後仍可保留。此次當機表示，在受影響的 `tsx`/Node 組合中，
該模組呼叫位置的輔助函式遺失或遭到遮蔽，因此 `__name(...)`
擲出錯誤，而非傳回包裝後的值。

## 目前的重現檢查

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

最小化的獨立重現方式（僅載入原始堆疊追蹤中的模組）：

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

目前這兩個指令都能正常結束。如果任一指令再次擲出 `__name is not a
function`，請先擷取確切的 Node 版本、`tsx` 版本
（`node_modules/tsx/package.json`）及完整堆疊追蹤，再向上游回報。

## 因應措施（如果當機問題再次出現）

- 使用 Bun 而非 `node --import tsx` 執行開發指令碼。
- 執行 `pnpm tsgo` 進行型別檢查，然後執行建置後的輸出，而非透過
  `tsx` 執行原始碼：

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- 嘗試不同的 `tsx` 版本（`pnpm add -D tsx@<version>` 屬於相依套件
  變更，依儲存庫政策需要核准），以二分查找其內含的 esbuild
  版本是否再次引入此錯誤。
- 在不同的 Node 主要／次要版本上測試，以確認此故障是否僅發生於特定版本。

## 參考資料

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## 相關內容

- [Node.js 安裝](/zh-TW/install/node)
- [閘道疑難排解](/zh-TW/gateway/troubleshooting)
