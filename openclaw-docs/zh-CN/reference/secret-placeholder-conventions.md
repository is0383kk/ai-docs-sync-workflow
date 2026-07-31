---
read_when:
    - 编写包含令牌、API 密钥或凭据片段的文档
    - 更新可能会被密钥检测工具扫描的示例
summary: 适用于文档和示例的密钥扫描器安全占位符约定
title: 密钥占位符约定
x-i18n:
    generated_at: "2026-07-26T06:59:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0864f0fcc6fb1e4a3147b4b2ce0aac475437a19d694f3d059374782428c7f248
    source_path: reference/secret-placeholder-conventions.md
    workflow: 16
---

# 密钥占位符约定

使用便于人类阅读、但不会与真实密钥相似的占位符。

## 推荐风格

- 优先使用 `example-openai-key-not-real` 或 `example-discord-bot-token` 之类的描述性值。
- 对于 shell 代码片段，优先使用 `${OPENAI_API_KEY}`，而不是内联的类似令牌的字符串。
- 确保示例明显为虚构内容，并限定于具体用途（提供商、渠道、身份验证类型）。

## 文档中应避免这些模式

- 字面形式的 PEM 私钥标头或页脚文本。
- 与真实凭据相似的前缀，例如 `sk-...`、`xoxb-...`、`AKIA...`。
- 从运行时日志复制的、看起来真实的持有者令牌。

## 示例

```bash
# 良好
export OPENAI_API_KEY="example-openai-key-not-real"

# 更好（当文档介绍环境变量接线时）
export OPENAI_API_KEY="${OPENAI_API_KEY}"
```
