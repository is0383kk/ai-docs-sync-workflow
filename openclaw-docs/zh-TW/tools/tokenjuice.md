---
read_when:
    - 你想在 OpenClaw 中取得更簡短的 `exec` 或 `bash` 工具結果
    - 你想要安裝或啟用 Tokenjuice 外掛
    - 你需要了解 Tokenjuice 會變更哪些內容，以及哪些內容會保留原始形式
summary: 使用選用的 Tokenjuice 外掛壓縮雜訊較多的 exec 與 bash 工具結果
title: Tokenjuice
x-i18n:
    generated_at: "2026-07-26T08:17:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96b110563a2600429dd9f0d38997cf7cc5ae4952b7f146a6ab64c96f2f202440
    source_path: tools/tokenjuice.md
    workflow: 16
---

`tokenjuice` 是選用的外部外掛，會在命令已執行後，壓縮雜亂的 `exec` 和 `bash`
工具結果。

它會變更傳回的 `tool_result`，而非命令本身。Tokenjuice 不會
重寫 shell 輸入、重新執行命令或變更結束代碼。

目前這適用於 OpenClaw 內嵌執行，以及 Codex app-server 測試框架中的 OpenClaw
動態工具。Tokenjuice 會掛接 OpenClaw 的工具結果中介軟體，並在輸出傳回
作用中測試框架工作階段之前加以修剪。

## 啟用外掛

安裝一次：

```bash
openclaw plugins install clawhub:@openclaw/tokenjuice
```

接著啟用：

```bash
openclaw config set plugins.entries.tokenjuice.enabled true
```

等效命令：

```bash
openclaw plugins enable tokenjuice
```

如果偏好直接編輯設定：

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## Tokenjuice 會變更什麼

- 在雜亂的 `exec` 和 `bash` 結果送回工作階段之前加以壓縮。
- 保持原始命令執行完全不變。
- 套用安全清查原則：精確的檔案內容讀取會保留原始輸出，獨立的儲存庫清查命令可以壓縮，而不安全的混合命令序列則保留原始輸出。
- 維持選擇性啟用：如果希望所有輸出都逐字保留，請停用此外掛。

## 驗證是否正常運作

1. 啟用外掛。
2. 啟動可呼叫 `exec` 的工作階段。
3. 執行雜亂的命令，例如 `git status`。
4. 確認傳回的工具結果比原始 shell 輸出更短且更有結構。

## 停用外掛

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
```

或：

```bash
openclaw plugins disable tokenjuice
```

## 相關內容

- [Exec 工具](/zh-TW/tools/exec)
- [思考層級](/zh-TW/tools/thinking)
- [上下文引擎](/zh-TW/concepts/context-engine)
