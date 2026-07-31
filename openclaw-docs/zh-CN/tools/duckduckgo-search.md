---
read_when:
    - 你需要一个无需 API key 的 Web 搜索提供商
    - 你想将 DuckDuckGo 用于 `web_search`
    - 你需要一个明确选择的无密钥搜索提供商
summary: DuckDuckGo Web 搜索——无需密钥的提供商（实验性，基于 HTML）
title: DuckDuckGo 搜索
x-i18n:
    generated_at: "2026-07-26T07:03:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84e90532de276dcb3f73c67015dffe5f5a62be673e44a19053b2b1dfcb0986ac
    source_path: tools/duckduckgo-search.md
    workflow: 16
---

OpenClaw 支持将 DuckDuckGo 用作**无需密钥**的 `web_search` 提供商。无需 API 密钥或账户。

<Warning>
  DuckDuckGo 是一项**实验性、非官方**集成，它会抓取 DuckDuckGo 的非 JavaScript HTML 搜索页面，而非使用官方 API。机器人验证页面或 HTML 变更可能会偶尔导致功能失效。
</Warning>

## 设置

由于自动检测仅考虑具有可用凭据的提供商，因此绝不会自动选择 DuckDuckGo。请明确设置：

<Steps>
  <Step title="配置">
    ```bash
    openclaw configure --section web
    # 选择 "duckduckgo" 作为提供商
    ```
  </Step>
</Steps>

## 配置

直接在配置中设置提供商：

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

可选的插件级区域和 SafeSearch 设置：

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // DuckDuckGo 区域代码
            safeSearch: "moderate", // "strict"、"moderate" 或 "off"
          },
        },
      },
    },
  },
}
```

## 工具参数

<ParamField path="query" type="string" required>
搜索查询。
</ParamField>

<ParamField path="count" type="number" default="5">
要返回的结果数（1-10）。
</ParamField>

<ParamField path="region" type="string">
DuckDuckGo 区域代码（例如 `us-en`、`uk-en`、`de-de`）。
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
SafeSearch 级别。
</ParamField>

`region` 和 `safeSearch` 工具参数会按查询覆盖上述插件配置值。

## 注意事项

- **无需 API 密钥**——选择 DuckDuckGo 作为 `web_search` 提供商后即可使用。
- **实验性**——抓取 DuckDuckGo 的非 JavaScript HTML 搜索页面，而非使用官方 API 或 SDK。结果取决于页面结构，而页面结构可能随时变更，恕不另行通知。
- **机器人验证风险**——在高频或自动化使用的情况下，DuckDuckGo 可能会提供 CAPTCHA 或阻止请求。
- **只能明确选择**——OpenClaw 的自动检测仅考虑具有可用凭据的提供商，因此绝不会自动选择 DuckDuckGo 这类无需密钥的提供商；你必须设置 `provider: "duckduckgo"`。
- **未配置时，SafeSearch 默认为 `moderate`**。

<Tip>
  对于生产用途，请考虑使用 [Brave Search](/zh-CN/tools/brave-search)（提供免费套餐）或其他基于 API 的提供商。
</Tip>

## 相关内容

- [Web 搜索概览](/zh-CN/tools/web)——所有提供商和自动检测
- [Brave Search](/zh-CN/tools/brave-search)——提供免费套餐的结构化结果
- [Exa Search](/zh-CN/tools/exa-search)——具有内容提取功能的神经搜索
