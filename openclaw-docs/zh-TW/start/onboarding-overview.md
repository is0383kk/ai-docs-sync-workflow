---
read_when:
    - 選擇新手設定流程
    - 設定新環境
sidebarTitle: Onboarding Overview
summary: OpenClaw 新手引導選項與流程概覽
title: 入門設定概覽
x-i18n:
    generated_at: "2026-07-26T08:07:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4bcda1dcfb91f388ca6bef59f9bdf5177571d93c0d89c45025ef837628fa7ba0
    source_path: start/onboarding-overview.md
    workflow: 16
---

OpenClaw 提供終端與 macOS App 的新手設定流程。兩者都會先建立推論能力：
它們會偵測現有的 AI 存取權、要求完成一次即時回覆，之後才啟動
OpenClaw 以設定其餘項目。如果可連線且已設定的閘道，其預設代理程式
已有設定好的模型，便會略過新手設定並開啟一般代理程式 UI。終端流程也提供完整的傳統精靈，
以進行詳細設定。

## 我該使用哪種方式？

|                | 命令列介面新手設定                         | macOS App 新手設定           |
| -------------- | -------------------------------------- | ------------------------------ |
| **平台**  | macOS、Linux、Windows（原生或 WSL2） | 僅限 macOS                     |
| **介面**  | 先設定推論，再設定 OpenClaw         | 先設定推論，再設定 OpenClaw |
| **最適合**   | 伺服器、無頭環境、完整控制        | 桌面 Mac、視覺化設定      |
| **自動化** | 用於指令碼的 `--non-interactive`        | 僅限手動                    |
| **命令**    | `openclaw onboard`                     | 啟動 App                 |

大多數使用者應從 **命令列介面新手設定** 開始——它可在任何平台使用，並讓
你擁有最大的控制權。

## 新手設定會設定哪些項目

引導式推論階段只會建立：

1. **模型供應商與驗證**——偵測到的存取權，或已驗證的供應商登入、
   API 金鑰或權杖
2. **已驗證的推論**——使用預設代理程式實際生效的
   模型完成一次真實回覆

該回覆通過後，OpenClaw 即可設定工作區、閘道、
閘道服務、頻道、代理程式、外掛及其他選用功能。

傳統命令列介面精靈還能設定：

1. **頻道**（選用）——內建及隨附的聊天頻道，例如
   Discord、Feishu、Google Chat、iMessage、Mattermost、Microsoft Teams、
   Telegram、WhatsApp 等
2. **進階閘道控制項**——遠端模式、網路設定及常駐程式選項

## 命令列介面新手設定

在任何終端中執行：

```bash
openclaw onboard
```

引導式流程會偵測現有的 AI 存取權、依序即時測試候選項目，
並在失敗時繼續嘗試下一個。若偵測完所有項目仍無結果，會先顯示 OpenAI、
Anthropic、xAI（Grok）、Google 和 OpenRouter。**More…** 會依供應商群組列出
其餘供應商，第二層選單中包含地區、方案，以及支援的
瀏覽器、裝置、API 金鑰或權杖方式。只有在回覆測試通過後，才會儲存模型
與認證資訊，接著啟動 OpenClaw 以
設定工作區、閘道、頻道、代理程式、外掛及其他選用
功能。**Skip for now** 會直接結束而不啟動 OpenClaw。流程中不會
轉交至傳統精靈；若要改用傳統精靈，請結束並執行 `openclaw onboard --classic`。

推論通過後，OpenClaw 可將頻道設定交給使用遮罩輸入的終端
精靈。它不會開啟引導式或傳統供應商設定；若要變更模型供應商或其驗證方式，請結束 OpenClaw 並
執行 `openclaw onboard`。

使用 `openclaw onboard --classic` 進行詳細的模型／驗證、頻道、技能、
遠端閘道或匯入設定。加上 `--install-daemon` 也會選取
傳統流程，並在同一個步驟中安裝背景服務。使用 `openclaw
openclaw` 進行對話式的非推論設定與修復。`openclaw
onboard --modern` 是使用相同即時推論
閘門的相容性別名。

完整參考：[新手設定（命令列介面）](/zh-TW/start/wizard)
命令列介面命令文件：[`openclaw onboard`](/zh-TW/cli/onboard)

## macOS App 新手設定

開啟 OpenClaw App。如果其已設定的本機或遠端閘道可以連線，
且預設代理程式已有設定好的模型，App 會略過新手設定
與 OpenClaw 設定，並立即開啟一般代理程式 UI。

對於全新或設定不完整的閘道，首次執行流程會偵測現有的 AI
存取權（Claude Code、Codex 或 API 金鑰）、即時測試最佳
選項，並僅在收到真實回覆後儲存——若失敗則自動改用其他選項，
若找不到任何項目，則提供經驗證的手動 API 金鑰步驟。敏感的
認證資訊會使用遮罩輸入。推論通過後，OpenClaw 會啟動並
協助設定其餘項目。

設定完成後，一般代理程式仍可使用 Gemini CLI，但推論
閘門不會提供此選項，因為它無法強制執行不使用工具的探測。

完整參考：[新手設定（macOS App）](/zh-TW/start/onboarding)

## 自訂或未列出的供應商

如果你的供應商未列出，請執行 `openclaw onboard --classic`，選擇
**Custom Provider**，並輸入：

- 端點相容性：OpenAI 相容（`/chat/completions`）、OpenAI Responses 相容（`/responses`）、Anthropic 相容（`/messages`），或未知（探測全部三種並自動偵測）
- 基礎 URL 與 API 金鑰（若端點不需要 API 金鑰，則可省略）
- 模型 ID 與選用的模型別名

多個自訂端點可以並存——每個端點都會取得自己的端點 ID。

## 相關內容

- [開始使用](/zh-TW/start/getting-started)
- [命令列介面設定參考](/zh-TW/start/wizard-cli-reference)
