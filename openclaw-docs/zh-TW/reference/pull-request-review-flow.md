---
read_when:
    - 針對 Barnacle 或 ClawSweeper 的意見進行後續處理
    - 請 ClawSweeper 進行審查
    - 偵錯 Barnacle、ClawSweeper、過時標籤或自動關閉問題
sidebarTitle: PR review flow
summary: Barnacle 和 ClawSweeper 的意見回饋如何協助 OpenClaw 提取要求順利通過審查。
title: PR 審查流程
x-i18n:
    generated_at: "2026-07-26T07:33:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e9bec4578d55d2279450e991480467946db7da5ca956f85c35b4221190b2babe
    source_path: reference/pull-request-review-flow.md
    workflow: 16
---

本頁說明你開啟或更新 OpenClaw PR 後的審查流程：Barnacle 和 ClawSweeper 的作用、如何根據其意見改進 PR，以及自動化沒有回應時應檢查哪些事項。

Barnacle 和 ClawSweeper 協助維護者維持審查佇列的可用性，但無法取代維護者的判斷。

## Barnacle

Barnacle 是確定性的 GitHub 分流工具。它會尋找已知的佇列管理情況，並透過標籤、留言或關閉項目來回應。

Barnacle 可能會在以下情況採取動作：

- PR 內文幾乎空白或缺少問題背景；
- PR 沒有實用的證據；
- 僅文件、僅測試、僅重構、僅 CI 或基礎架構變更未連結相關的維護者背景資訊；
- 變更看起來應屬於 ClawHub 或外掛，而非核心；
- 分支包含不相關的工作；
- 作者有超過 20 個開放中的 PR。

Barnacle 會從受信任的儲存庫工作流程程式碼執行。它不會簽出或執行貢獻者的程式碼。

大多數分流標籤都是供維護者或自動化使用的訊號，因此貢獻者不需要自行新增標籤。

## ClawSweeper

ClawSweeper 是供 OpenClaw 儲存庫使用的 AI 輔助審查與維護機器人。它可以審查 PR、評估證明、留下持久的審查留言，並透過受控的修復或自動合併流程協助維護者。

ClawSweeper 的正面結果是輔助證據，而非維護者的核准。維護者仍會決定 PR 是否已可合併，以及何時合併。

ClawSweeper 採用佇列機制。開啟 PR、推送提交或新增審查要求後，請勿期待立即收到回應。ClawSweeper 執行後，標籤更新也可能需要一段時間。

新的 PR 會進入 ClawSweeper 審查佇列。維護者也可以使用標籤或命令，將審查、修復或自動合併流程加入佇列。對於一般貢獻者的更新，只有在你已更新分支、PR 說明、證明或程式碼後，才要求 ClawSweeper 再次審查。接著以新的 PR 留言要求重新審查：

```text
@clawsweeper re-review
```

PR 作者也可以使用 `@clawsweeper re-run`；具有儲存庫寫入權限的使用者可以對任何開放中的項目使用任一命令。純 `@clawsweeper review` 命令僅限維護者使用。請耐心等候：在完成要求的變更前再次提出要求，只會增加佇列雜訊。

ClawSweeper 留下審查對話時，請將其視為一般審查意見，並使用下方的後續檢查清單。

如果人類貢獻者或維護者已接手 PR 且正在積極處理，請勿同時叫用 ClawSweeper 或以其他方式處理該 PR。請先讓人類完成審查或修復。如果活動停止，請檢查是否已要求作者提供證明或進行其他更新。

## 在審查期間改進 PR

Barnacle、ClawSweeper 或維護者回應後，請將該意見作為 PR 的後續步驟檢查清單。

1. 將 ClawSweeper 的 `Rank-up moves:` 和 `Proof guidance:` 視為該 PR 的行動清單。評分和標籤是審查訊號，不是固定的合併目標。
2. 推送所要求的程式碼或文件變更；如果問題、解決方案、使用者影響或證據有所變更，也請更新 PR 說明。
3. 新增所要求的證明，並使用與變更相符的證據。
4. 自行解決已處理的審查對話。只有需要維護者或審查者判斷時，才回覆並讓對話保持開放。
5. 只有在分支、PR 說明、證據及相關 CI 結果均為最新狀態後，才要求重新審查。作者、維護者與 ClawSweeper 之間經過多輪更新及審查是正常情況。
6. 盡可能在 PR 中進行討論。只有在 PR 需要維護者協調、自動化似乎受阻，或難以在 GitHub 留言中決定下一步時，才移至 Discord 上的 `#clawtributors`。請附上 PR 連結、目前狀態，以及具體問題或尚缺的證據。

請讓 PR 內文保持最新。留言有助於討論，但 PR 說明才是維護者和自動化會反覆查閱的持久摘要。

`status: ⏳ waiting on author` 表示下一步應由 PR 作者執行：更新分支、PR 說明、證明，或回覆缺少的背景資訊，之後再要求另一次審查。

實用的證據包括聚焦的測試輸出、CI 結果、螢幕擷取畫面、錄影、終端機輸出、即時觀察、經過遮蔽處理的記錄或成品連結。對於視覺變更，在可行情況下請附上變更前後的螢幕擷取畫面。對於證明檔案，建議連結 CI 成品、上傳至 GitHub 的螢幕擷取畫面或錄影，或一小段經過遮蔽處理的記錄。除非產生的證明檔案屬於實際文件、測試或產品變更的一部分，否則請勿將其提交。

遮蔽敏感資料是貢獻者的責任。發布證明前，請移除密鑰、權杖、私人 URL、使用者資料和不相關的記錄。

OpenClaw 也使用獨立的過期項目自動化。未指派的議題和 PR 在閒置 14 天後可能被標記為過期，再閒置 7 天後會被關閉。已指派的 PR 無論之後是否更新，都會在開啟 27 天後被標記為過期；若過期後 7 天內沒有活動，便會被關閉。如果已指派的 PR 仍在進行中，請與正在處理該 PR 的維護者協調。

## 自動化沒有回應時

當維護者已在處理項目、審查或修復要求仍在佇列中、事件屬於例行性質，或 ClawSweeper 管道未針對所要求的動作進行設定時，自動化可能不會回應。

當受信任的工作流程需要執行不受信任的貢獻者程式碼時，它也可能不採取動作。在這種情況下，維護者會改用一般審查或更安全的工作流程。

## 疑難排解

如果 ClawSweeper 沒有立即回應，請先等候再重試。此服務採用佇列機制；重複留言或變更標籤並不會加快佇列處理，反而可能使討論串更難審查。

尋求協助前，請檢查：

- PR 說明是最新的；
- 最新提交包含所要求的變更；
- CI 已完成，或 PR 內文已說明任何尚未解決的失敗為何與該 PR 無關；
- 最新的審查要求是以 PR 留言提出：
  `@clawsweeper re-review`；
- 沒有維護者或貢獻者已在積極處理該 PR；
- 最新要求並非仍處於正常的 ClawSweeper 佇列延遲時間內。

如果 PR 更新完成數小時後，ClawSweeper 仍未回應，或 PR 似乎受到自動化阻礙，請在 Discord 的 `#clawtributors` 中尋求協助。請附上 PR 連結、你預期的結果、提出要求的時間，以及自上次機器人留言後有哪些變更。

## 分支建立自動化

想使用類似審查自動化功能的專案，可以研究 ClawSweeper 或建立其分支：

- [openclaw/clawsweeper](https://github.com/openclaw/clawsweeper)
- [ClawSweeper 文件](https://clawsweeper.bot/)

## 相關資訊

- [參與貢獻](https://github.com/openclaw/openclaw/blob/main/CONTRIBUTING.md)
- [CI 流水線](/zh-TW/ci)
