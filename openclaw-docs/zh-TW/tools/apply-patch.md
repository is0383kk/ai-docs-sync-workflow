---
read_when:
    - 你需要對多個檔案進行結構化編輯
    - 你想要記錄或偵錯以修補程式為基礎的編輯作業
summary: 使用 apply_patch 工具套用多檔案修補程式
title: apply_patch 工具
x-i18n:
    generated_at: "2026-07-26T08:14:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c0422550ea8d9b0cb6b0ea22d7dcaecc462426f9600003f70c177746f30a3d9
    source_path: tools/apply-patch.md
    workflow: 16
---

使用結構化修補格式套用檔案變更。這非常適合多檔案
或多區塊編輯，因為使用單一 `edit` 呼叫容易出錯。

此工具接受單一 `input` 字串，其中包含一個或多個檔案操作：

```text
*** Begin Patch
*** Add File: path/to/file.txt
+第 1 行
+第 2 行
*** Update File: src/app.ts
@@ 選用的變更內容
-舊行
+新行
*** Delete File: obsolete.txt
*** End Patch
```

## 參數

- `input`（必填）：完整的修補內容，包括 `*** Begin Patch` 和 `*** End Patch`。

## 注意事項

- 修補路徑支援相對路徑（相對於工作區目錄）和絕對路徑。
- `tools.exec.applyPatch.workspaceOnly` 預設為 `true`（限制在工作區內）。只有在你確實想讓 `apply_patch` 寫入或刪除工作區目錄以外的內容時，才將其設為 `false`。
- 若要重新命名檔案，請在 `*** Update File:` 區塊內使用 `*** Move to:`。
- `*** End of File` 會在需要時標示僅插入至 EOF 的操作。
- 預設為所有模型啟用。將 `tools.exec.applyPatch.enabled: false`
  設為停用，或使用
  `tools.exec.applyPatch.allowModels` 將其限制為特定模型（接受 `gpt-5.4` 之類的原始 ID，或
  `openai/gpt-5.4` 之類的完整 ID）。
- 設定位於 `tools.exec.applyPatch.*` 下。

## 範例

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```

## 相關內容

<CardGroup cols={2}>
  <Card title="差異" href="/zh-TW/tools/diffs" icon="code-compare">
    用於呈現變更的唯讀差異檢視器。
  </Card>
  <Card title="執行工具" href="/zh-TW/tools/exec" icon="terminal">
    從代理程式執行 Shell 命令。
  </Card>
  <Card title="程式碼執行" href="/zh-TW/tools/code-execution" icon="square-code">
    使用 xAI 在沙箱化的遠端環境中進行 Python 分析。
  </Card>
</CardGroup>
