---
read_when:
    - 你想試用 ClawHub 提供的免費促銷模型優惠
    - 你是透過促銷活動設定供應商，而不是透過新手引導流程
summary: '`openclaw promos` 的命令列介面參考（列出並領取促銷模型優惠）'
title: 促銷活動
x-i18n:
    generated_at: "2026-07-26T07:38:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 779eab2e9500b7376fabf9accb333e83ff5f84b085d51b7d551b5507b1e73adb
    source_path: cli/promos.md
    workflow: 16
---

# `openclaw promos`

探索並領取在 ClawHub 上發布的模型促銷優惠。領取促銷活動會設定提供者（需要時包括驗證和外掛）並註冊該促銷活動的模型，而不必重新執行初始設定，也不會變更你的預設模型，除非你指定要這麼做。

相關內容：

- 預設模型與備援模型：[模型](/zh-TW/cli/models)
- 提供者驗證設定：[開始使用](/zh-TW/start/getting-started)

## 命令

```bash
openclaw promos list
openclaw promos claim <slug>
openclaw promos claim <slug> --api-key <key> --set-default
```

## `openclaw promos list`

列出目前有效的促銷活動，包括其模型、建議的預設模型、剩餘時間，以及確切的領取命令。`--json` 會輸出原始承載資料。

## `openclaw promos claim <slug>`

領取有效的促銷活動：

1. 從 ClawHub 取得促銷活動，並驗證其目前仍在有效期間內。
2. 根據你已安裝的 OpenClaw 版本，驗證促銷活動的提供者、驗證選項和宣告的外掛套件。未知的 ID 或套件不相符情況會遭到拒絕；促銷活動絕不可能讓命令列介面執行任何它原本不知道如何執行的操作。
3. 如果你已有提供者認證資訊，則會重複使用。否則，它會引導你完成提供者的一般驗證流程（先顯示促銷活動的註冊 URL，以取得免費金鑰）。`--api-key <key>` 會以無提示方式完成 API 金鑰驗證，行為與 `openclaw onboard` 非互動式旗標一致；若要避免在命令列中提供金鑰，請改為匯出提供者的環境變數（例如 `OPENROUTER_API_KEY`）。系統會自動偵測現有的環境認證資訊，無須使用旗標。
4. 註冊促銷活動的模型及其別名。絕不會覆寫現有別名。
5. 詢問是否將促銷活動建議的模型設為預設模型；`--set-default` 會略過此問題，否則你的預設設定不會有任何變更。

促銷活動的有效期間結束後，提供者將停止供應免費模型；你的設定和認證資訊不受影響。你隨時可以使用 `openclaw models set <model>` 切換回去。

## `models list` 中的被動探索

`openclaw models list` 也會在你未直接查詢 ClawHub 的情況下顯示促銷活動：

- 尚未設定其模型的有效優惠會顯示在表格下方的「可透過促銷活動取得」群組中，每項優惠都會附上其領取命令。
- 透過 `promos claim` 註冊的模型會帶有 `promo` 標籤，優惠有效期間結束後，該標籤會變為 `promo ended`。
- 首次發現新優惠時，一次性通知會指向 `openclaw promos list`。已列出或領取的優惠不會再次通知。

此功能會讀取 ClawHub 託管之促銷活動摘要的本機快取副本（通常每天透過條件式請求重新整理一次，或在快取快照提早到期時重新整理；重新整理失敗時會靜默略過）。過期快取的重新整理最多等待 2.5 秒，且絕不會導致列表功能失敗。`--json` 和 `--plain` 的輸出會保持機器可讀的純淨格式，不包含促銷活動區段或通知。領取時一律會透過即時 ClawHub API 重新驗證，因此即使快取副本仍顯示某項提前撤回的優惠，系統也會拒絕領取。
