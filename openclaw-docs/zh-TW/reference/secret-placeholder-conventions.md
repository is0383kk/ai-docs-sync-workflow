---
read_when:
    - 撰寫包含權杖、API 金鑰或認證資訊片段的文件
    - 更新可能會被機密偵測工具掃描的範例
summary: 適用於文件與範例且對機密掃描器安全的預留位置慣例
title: 密鑰預留位置慣例
x-i18n:
    generated_at: "2026-07-26T08:36:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0864f0fcc6fb1e4a3147b4b2ce0aac475437a19d694f3d059374782428c7f248
    source_path: reference/secret-placeholder-conventions.md
    workflow: 16
---

# 機密資訊預留位置慣例

使用人類可讀但不類似真實機密資訊的預留位置。

## 建議樣式

- 優先使用如 `example-openai-key-not-real` 或 `example-discord-bot-token` 這類描述性值。
- 在 shell 程式碼片段中，優先使用 `${OPENAI_API_KEY}`，而非看似權杖的行內字串。
- 讓範例明顯為虛構內容，並限定於特定用途（提供者、頻道、驗證類型）。

## 文件中應避免的模式

- 字面上的 PEM 私密金鑰標頭或頁尾文字。
- 類似有效認證資訊的前綴，例如 `sk-...`、`xoxb-...`、`AKIA...`。
- 從執行階段記錄複製、看似真實的不記名權杖。

## 範例

```bash
# 良好
export OPENAI_API_KEY="example-openai-key-not-real"

# 更好（當文件說明環境變數串接時）
export OPENAI_API_KEY="${OPENAI_API_KEY}"
```
