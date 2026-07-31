---
read_when:
    - 你想使用 Brave Search 进行 web_search
    - 你需要 BRAVE_API_KEY 或套餐详情
summary: 用于 `web_search` 的 Brave Search API 设置
title: Brave 搜索
x-i18n:
    generated_at: "2026-07-26T07:03:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw 支持将 Brave Search API 用作 `web_search` 提供商。

## 获取 API 密钥

1. 在 [https://brave.com/search/api/](https://brave.com/search/api/) 创建 Brave Search API 账户
2. 在控制面板中，选择 **Search** 套餐并生成 API 密钥。
3. 将密钥存储在配置中，或在 Gateway 网关环境中设置 `BRAVE_API_KEY`。

## 配置示例

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // 或 "llm-context"
            baseUrl: "https://api.search.brave.com", // 可选的代理/基础 URL 覆盖值
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Brave 搜索的提供商特定设置位于 `plugins.entries.brave.config.webSearch.*` 下；这是规范配置路径。

`webSearch.mode` 控制 Brave 传输方式：

- `web`（默认）：常规 Brave Web 搜索，包含标题、URL 和摘要
- `llm-context`：Brave LLM Context API，提供预先提取的文本块和来源以进行依据支撑

`webSearch.baseUrl` 可将 Brave 请求指向受信任且兼容 Brave 的代理
或网关。OpenClaw 会将 `/res/v1/web/search` 或 `/res/v1/llm/context` 追加到
配置的基础 URL，并将该基础 URL 纳入缓存键。公共
端点必须使用 `https://`；仅受信任的 local loopback
或专用网络代理主机可以使用 `http://`。

## 工具参数

<ParamField path="query" type="string" required>
搜索查询。
</ParamField>

<ParamField path="count" type="number" default="5">
返回的结果数量（1–10）。
</ParamField>

<ParamField path="country" type="string">
2 位 ISO 国家/地区代码（例如 `US`、`DE`）。
</ParamField>

<ParamField path="language" type="string">
搜索结果的 ISO 639-1 语言代码（例如 `en`、`de`、`fr`）。
</ParamField>

<ParamField path="search_lang" type="string">
Brave 搜索语言代码（例如 `en`、`en-gb`、`zh-hans`）。
</ParamField>

<ParamField path="ui_lang" type="string">
UI 元素的 ISO 语言代码。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
时间筛选条件——`day` 表示 24 小时。
</ParamField>

<ParamField path="date_after" type="string">
仅返回在此日期之后发布的结果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="date_before" type="string">
仅返回在此日期之前发布的结果（`YYYY-MM-DD`）。
</ParamField>

**示例：**

```javascript
// 针对特定国家/地区和语言进行搜索
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// 最近的结果（过去一周）
await web_search({
  query: "AI news",
  freshness: "week",
});

// 按日期范围搜索
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## 注意事项

- OpenClaw 使用 Brave **Search** 套餐。如果你拥有旧版订阅（例如最初的 Free 套餐，每月包含 2,000 次查询），该订阅仍然有效，但不包含 LLM Context 或更高速率限制等较新的功能。
- 每个 Brave 套餐都包含**每月 \$5 的免费额度**（每月续期）。Search 套餐每 1,000 次请求收费 \$5，因此该额度可覆盖每月 1,000 次查询。请在 Brave 控制面板中设置用量上限，以避免意外收费。有关当前套餐，请参阅 [Brave API 门户](https://brave.com/search/api/)。
- Search 套餐包含 LLM Context 端点和 AI 推理权利。存储结果以训练或调优模型需要具有明确存储权利的套餐。请参阅 Brave [服务条款](https://api-dashboard.search.brave.com/terms-of-service)。
- `llm-context` 模式返回有来源依据的条目，而不是常规 Web 搜索摘要格式。
- `llm-context` 模式支持 `freshness` 以及有界的 `date_after` + `date_before` 范围。它不支持 `ui_lang`；如果设置 `date_before` 而未设置 `date_after`，请求将被拒绝，因为 Brave 要求自定义新鲜度范围必须同时包含开始日期和结束日期。
- `ui_lang` 必须包含类似 `en-US` 的区域子标签。
- 默认情况下，结果会缓存 15 分钟（可通过 `cacheTtlMinutes` 配置）。
- 自定义 `webSearch.baseUrl` 值会纳入 Brave 缓存标识，因此
  特定于代理的响应不会发生冲突。
- 启用 `brave.http` 诊断标志，可在故障排查期间记录 Brave 请求 URL/查询参数、响应状态/耗时，以及搜索缓存命中/未命中/写入事件。该标志绝不会记录 API 密钥或响应正文，但搜索查询可能包含敏感信息。

## 相关内容

- [Web 搜索概览](/zh-CN/tools/web) -- 所有提供商和自动检测
- [Perplexity Search](/zh-CN/tools/perplexity-search) -- 支持域名筛选的结构化结果
- [Exa Search](/zh-CN/tools/exa-search) -- 支持内容提取的神经网络搜索
