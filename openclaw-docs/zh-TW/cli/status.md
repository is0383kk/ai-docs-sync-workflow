---
read_when:
    - 你想快速診斷頻道健康狀況和最近的工作階段收件者
    - 你想要一份可直接貼上的「全部」狀態資訊，以便進行偵錯
summary: '`openclaw status` 的命令列介面參考（診斷、探查、用量快照）'
title: openclaw status
x-i18n:
    generated_at: "2026-07-26T08:13:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52e8076339216f11ddadf35e0ae8e5604322a47a5a9e2ee305468b2624d7cfde
    source_path: cli/status.md
    workflow: 16
---

通道與工作階段的診斷。

```bash
openclaw status
openclaw status --all
openclaw status --deep
openclaw status --usage
```

| 旗標                    | 說明                                                                                                     |
| ----------------------- | --------------------------------------------------------------------------------------------------------------- |
| `--all`                 | 完整診斷（唯讀、可直接貼上）。包含安全性稽核、外掛相容性與記憶向量探測。 |
| `--deep`                | 執行即時探測（WhatsApp Web + Telegram + Discord + Slack + Signal）。同時啟用安全性稽核。         |
| `--usage`               | 將正規化的供應商用量時段輸出為 `X% left`。                                                          |
| `--json`                | 機器可讀的輸出。                                                                                        |
| `--verbose` / `--debug` | 也會在報告前輸出原始閘道目標解析結果。                                                 |

一般的 `openclaw status` 會維持快速唯讀路徑，並在略過記憶檢查時將記憶標示為
`not checked`，而非無法使用。繁重的
安全性稽核、外掛相容性與記憶向量探測則留給
`openclaw status --all`、`openclaw status --deep`、`openclaw security audit`
及 `openclaw memory status --deep`。

## 工作階段與模型解析

- 工作階段狀態輸出會區分 `Execution:` 與 `Runtime:`。`Execution`
  是沙箱路徑（`direct`、`docker/*`），而 `Runtime` 會指出
  工作階段使用的是 `OpenClaw Default`、`OpenAI Codex`、命令列介面
  後端，或 `codex (acp/acpx)` 等 ACP 後端。請參閱
  [代理程式執行階段](/zh-TW/concepts/agent-runtimes)，了解供應商、模型與執行階段之間的
  差異。
- 當目前的工作階段快照資訊不足時，`/status` 可從最近的逐字稿用量記錄回填權杖
  與快取計數器。現有的非零即時值仍優先於逐字稿的備援值。
- 當即時工作階段項目缺少目前使用的執行階段模型標籤時，
  逐字稿備援也能加以復原。如果該逐字稿模型與所選模型不同，
  狀態會依復原的執行階段模型解析上下文視窗，而非依所選模型解析。
- 在計算提示詞大小時，若工作階段中繼資料缺失或數值較小，
  逐字稿備援會優先採用較大的提示詞導向總計，因此
  自訂供應商工作階段的權杖顯示不會縮減為 `0`。
- 當工作階段固定使用與已設定主要模型不同的模型時，
  狀態會輸出兩個值、原因（`session override`），以及
  提示 `/model default`。已設定的主要模型適用於新的或
  未固定的工作階段；現有的固定工作階段會保留其工作階段選擇，
  直到清除為止。
- 設定多個代理程式時，輸出會包含各代理程式的工作階段儲存區。

## 用量與配額

- `--usage` 將正規化的供應商用量時段輸出為 `X% left`。
- MiniMax 的原始 `usage_percent` / `usagePercent` 欄位代表剩餘配額，
  因此 OpenClaw 會在顯示前反轉這些值；若有以計數為基礎的欄位，
  則優先採用。`model_remains` 回應會優先採用聊天模型項目、在需要時根據時間戳記推導
  時段標籤，並在方案標籤中包含模型名稱。
- 模型定價重新整理失敗時，會顯示為選用的定價警告。
  這不代表閘道或通道狀況異常。

## 概觀與更新狀態

- 概觀會在可用時包含閘道與節點主機服務的安裝／執行階段狀態，
  以及精簡的閘道程序運作時間與主機系統運作時間。
- 概觀包含更新通道與 git SHA（適用於原始碼簽出）。
- 更新資訊會顯示在概觀中；如果有可用更新，狀態
  會輸出提示，建議執行 `openclaw update`（請參閱[更新](/zh-TW/install/updating)）。

## 密鑰

- 當執行中的閘道有任何因啟動、重新載入或設定寫入而遭隔離的 SecretRef 擁有者時，狀態會在 JSON 中包含 `degradedSecretOwners`，並在人類可讀輸出的概觀中顯示 **已降級的密鑰** 資料列。每個項目都會列出擁有者、降級狀態（`cold` 或 `stale`）、設定路徑，以及經過遮蔽的原因。冷狀態的擁有者無法使用；過期狀態的擁有者則會繼續使用最後已知正常的值。
- 唯讀狀態介面（`status`、`status --json`、`status --all`）
  會在可行時，為其目標設定路徑解析支援的 SecretRef。
- 如果已設定支援的通道 SecretRef，但在目前的命令路徑中無法使用，
  狀態會保持唯讀並回報降級輸出，
  而不會當機。人類可讀輸出會顯示「此命令路徑無法使用已設定的權杖」
  等警告，而 JSON 輸出會包含
  `secretDiagnostics`。
- 當命令本機的 SecretRef 解析成功時，狀態會優先採用
  已解析的快照，並從最終輸出中清除暫時性的「密鑰無法使用」通道
  標記。
- `status --all` 包含密鑰概觀資料列與診斷區段，
  其中會摘要密鑰診斷資訊（為提高可讀性而截斷），且不會
  停止產生報告。

## 記憶

`status --json --all` 會回報由 `plugins.slots.memory` 選取的主動記憶外掛
執行階段中的記憶詳細資料。自訂記憶外掛可以停用
內建的 `memory.search.enabled`，同時仍回報
其自身的檔案、區塊、向量與 FTS 狀態。

## 相關內容

- [命令列介面參考](/zh-TW/cli)
- [Doctor](/zh-TW/gateway/doctor)
