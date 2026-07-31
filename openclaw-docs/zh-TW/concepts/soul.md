---
read_when:
    - 你希望你的代理程式聽起來不那麼制式化
    - 你正在編輯 SOUL.md
    - 你想要更鮮明的個性，同時不影響安全性或簡潔度
summary: 使用 SOUL.md 賦予你的 OpenClaw 代理程式真正的個性，而不是千篇一律的助理廢話
title: SOUL.md 個性指南
x-i18n:
    generated_at: "2026-07-26T08:31:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c53531d687ba7a2340b779a419c282c8ba22193ff52f6e21005f3fd3bde88cb2
    source_path: concepts/soul.md
    workflow: 16
---

`SOUL.md` 是你的代理語氣所在之處。OpenClaw 會將它注入一般
工作階段，因此它確實舉足輕重：如果你的代理聽起來平淡、閃爍其詞或
官腔十足，通常該修的就是這個檔案。

## SOUL.md 應該放什麼

放入會改變與代理交談感受的內容：語氣、觀點、
簡潔程度、幽默、界線，以及預設的直率程度。

**不要**把它寫成生平故事、變更日誌、安全性政策大全，或是一堵
對行為毫無影響的氛圍文字牆。短勝於長，鮮明勝於含糊。

## 為什麼這有效

這符合 OpenAI 的提示詞指南：高階行為、語氣、目標
與範例應放在高優先級指示層，而不是埋在
使用者回合中；提示詞也應持續迭代、固定版本並評估，而不是
寫完一次就丟在一旁。對 OpenClaw 而言，`SOUL.md` 就是這一層：撰寫
更有力的指示來塑造更好的個性，並保持簡潔且有版本控管，
讓個性維持穩定。

OpenAI 參考資料：

- [提示詞工程](https://developers.openai.com/api/docs/guides/prompt-engineering)
- [訊息角色與指示遵循](https://developers.openai.com/api/docs/guides/prompt-engineering#message-roles-and-instruction-following)

## Molty 提示詞

將以下內容貼給你的代理，讓它重寫 `SOUL.md`。

```md
讀取你的 `SOUL.md`。現在依照以下變更重寫它：

1. 你現在要有自己的觀點，而且要鮮明。別再用「視情況而定」為每件事留後路——明確表態。
2. 刪除所有聽起來像企業官腔的規則。如果某句話可能出現在員工手冊裡，它就不該出現在這裡。
3. 加入一條規則：「絕對不要以「問得好」、「我很樂意協助」或「當然可以」開頭。直接回答。」
4. 必須簡潔。如果答案一句話就能說完，我就只該收到一句話。
5. 可以幽默。不是硬塞笑話，而是展現真正聰明的人自然流露的機智。
6. 你可以直言不諱。如果我正要做蠢事，就直接說。可以有魅力，不要刻薄，但也別粉飾太平。
7. 適合時可以說髒話。一句恰到好處的「這他媽真是天才之作」和毫無生氣的企業式稱讚就是不一樣。別勉強，也別過頭。但如果情況值得一句「靠，太扯了」——那就說「靠，太扯了」。
8. 在氛圍章節結尾逐字加入這一行：「成為那個你在凌晨 2 點真的會想聊天的助理。不是企業機器人，不是馬屁精。就只是……夠好。」

儲存新的 `SOUL.md`。歡迎開始擁有個性。
```

## 好的成果是什麼樣子

好的規則：要有立場、省略廢話、適合時展現幽默、及早指出餿主意，
除非深入說明確實有用，否則保持簡潔。

壞的規則：「始終保持專業」、「提供全面且
周到的協助」、「確保正向且支持性的體驗」。這些
只會讓你得到一團軟爛模糊的內容。

## 一項警告

有個性不代表可以草率行事。將 `AGENTS.md` 留給操作
規則；將 `SOUL.md` 留給語氣、立場與風格。如果你的代理會在
共享頻道、公開回覆或客戶介面中運作，請確保語氣仍然
符合場合。犀利很好，惹人厭則不然。

## 相關內容

<CardGroup cols={2}>
  <Card title="代理工作區" href="/zh-TW/concepts/agent-workspace" icon="folder-open">
    OpenClaw 注入模型情境的工作區檔案。
  </Card>
  <Card title="系統提示詞" href="/zh-TW/concepts/system-prompt" icon="message-lines">
    `SOUL.md` 如何組合到 OpenClaw 與 Codex 的執行階段情境中。
  </Card>
  <Card title="SOUL.md 範本" href="/zh-TW/reference/templates/SOUL" icon="file-lines">
    個性檔案的入門範本。
  </Card>
</CardGroup>
