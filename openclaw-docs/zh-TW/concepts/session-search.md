---
read_when:
    - 你需要尋找先前工作階段中討論過的內容
    - 你想瞭解工作階段搜尋的隱私權或索引機制
summary: 搜尋過往工作階段逐字稿並重新開啟相符的上下文
title: 工作階段搜尋
x-i18n:
    generated_at: "2026-07-26T07:50:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3e9cda6b656b689eef0636592914f4890a64dca5e955aa03908377903aaa29c9
    source_path: concepts/session-search.md
    workflow: 16
---

# 工作階段搜尋

`sessions_search` 會搜尋你過往工作階段中的使用者與助理文字。每筆結果
都包含 `sessionKey`、時間戳記、角色，以及簡短的相符摘錄。需要查看前後對話時，
請將傳回的 `sessionKey` 傳給 `sessions_history`。

## 可見性與輸出

搜尋採用與 `sessions_history` 相同的工作階段可見性規則。系統會先移除呼叫者
可見工作階段樹以外的結果，再套用結果數量限制。啟用所建立工作階段的可見性時，沙箱化代理
仍僅能存取其所建立的工作階段。

摘錄會先經過遮蔽處理，再傳回模型。結果也會受到數量、摘錄長度
與回應總大小的限制。

## 索引生命週期

OpenClaw 會將全文索引與逐字稿資料列一同儲存在每個代理的 SQLite 資料庫中。
新的使用者與助理訊息會在持久化的同一筆交易中建立索引，因此索引
絕不會落後於即時對話；工具結果、推理區塊與圖片則不納入索引。
只有逐字稿的作用中分支可供搜尋。

早於索引建立的逐字稿（例如由 `openclaw doctor` 匯入的工作階段），以及
作用中分支曾回溯的工作階段，會由下一次搜尋時啟動的背景協調程序重新建立索引。
因此，包含 `indexing: true` 的回應可能不完整；請在索引建立完成後重試。
刪除工作階段時，會在同一筆交易中移除其索引項目。

搜尋目前使用 SQLite 的 Unicode 單字權杖化工具，並移除附加符號。
未來將改進為使用三元組權杖化，以支援 CJK 子字串比對。

## 工作階段搜尋與記憶搜尋

若要從原始工作階段逐字稿中搜尋完全相符的字詞或片語，請使用 `sessions_search`。若要搜尋
持久記憶檔案並進行語意回想，請使用 [`memory_search`](/zh-TW/concepts/memory-search)。實驗性的
工作階段記憶語料庫，是這項逐字稿精確搜尋的語意補充。
