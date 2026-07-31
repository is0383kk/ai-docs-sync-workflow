---
read_when:
    - 你想使用 Grok 进行 `web_search`
    - 你想使用 xAI OAuth 或 XAI_API_KEY 进行 Web 搜索
summary: 通过 xAI Web 信息支撑响应进行 Grok Web 搜索
title: Grok 搜索
x-i18n:
    generated_at: "2026-07-26T07:03:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6e39edd660d0ffe8be066ae81317810da691a7dbd8c59a74222a59145cff5c77
    source_path: tools/grok-search.md
    workflow: 16
---

OpenClaw 支持将 Grok 用作 `web_search` 提供商，利用 xAI 基于 Web 信息的响应，
生成由实时搜索结果支撑并附带引用的 AI 综合答案。

Grok Web 搜索会优先使用现有的 xAI OAuth 登录（如有）。
如果不存在 OAuth 配置文件，同一个 xAI API key 也可为内置的
`x_search` 工具提供支持，以搜索 X（原 Twitter）帖子，并为 `code_execution`
工具提供支持。将密钥存储在 `plugins.entries.xai.config.webSearch.apiKey` 中，还可让
OpenClaw 将其作为内置 xAI 模型提供商的备用密钥复用。

如需获取帖子级 X 指标（转发、回复、书签、浏览量），请使用
[`x_search`](/zh-CN/tools/web#x_search) 并提供准确的帖子 URL 或状态 ID，
而不要使用宽泛的搜索查询。

## 新手引导和配置

在 `openclaw onboard` 或 `openclaw configure --section
web` 期间选择 **Grok**，OpenClaw 即可复用现有的 xAI OAuth 配置文件，而不会提示输入
单独的 Web 搜索密钥。如果没有 OAuth，则回退到 xAI API key 设置。

随后，OpenClaw 会提供一个后续步骤，以使用同一个 xAI
凭据启用 `x_search`。此后续步骤：

- 仅在为 `web_search` 选择 Grok 后显示
- 不是一个独立的顶级 Web 搜索提供商选项
- 可选择在同一流程中设置 `x_search` 模型

可跳过此步骤，稍后在配置中启用或更改 `x_search`。

## 登录或获取 API key

<Steps>
  <Step title="使用 xAI OAuth">
    如果你已在新手引导或模型身份验证期间登录 xAI，请选择
    Grok 作为 `web_search` 提供商。无需单独的 API key：

    ```bash
    openclaw onboard --auth-choice xai-oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Step>
  <Step title="使用 API key 作为备用方案">
    当 OAuth 不可用，或你有意使用由密钥支持的 Web 搜索配置时，
    请从 [xAI](https://console.x.ai/) 获取 API key。
  </Step>
  <Step title="存储密钥">
    在 Gateway 网关环境中设置 `XAI_API_KEY`，或通过以下命令配置：

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 配置

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          webSearch: {
            apiKey: "xai-...", // 如果 xAI OAuth 或 XAI_API_KEY 可用，则为可选项
            baseUrl: "https://api.x.ai/v1", // 可选的 Responses API 代理/基础 URL 覆盖值
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "grok",
      },
    },
  },
}
```

**凭据替代方案：** Gateway 网关环境中的 `openclaw models auth login --provider xai
--method oauth`、`XAI_API_KEY`，或
`plugins.entries.xai.config.webSearch.apiKey`。对于 Gateway 网关安装，请将环境变量
放入 `~/.openclaw/.env`。

## 工作原理

Grok 使用 xAI 基于 Web 信息的响应来综合生成带有内联引用的答案，
类似于 Gemini 的 Google Search 信息支撑方式。

## 支持的参数

Grok 搜索支持 `query`。为兼容共享的 `web_search`，
也接受 `count`，但 Grok 始终返回一个带有引用的综合答案，
而不是包含 N 个结果的列表。不支持提供商特定的筛选条件。

Grok 默认超时时间为 60 秒，因为 xAI Responses 基于 Web 信息的
搜索可能比共享的 `web_search` 默认超时时间运行得更久。可使用
`tools.web.search.timeoutSeconds` 覆盖此设置。

## 基础 URL 覆盖

设置 `plugins.entries.xai.config.webSearch.baseUrl`，可通过操作员代理或
兼容 xAI 的 Responses 端点路由 Grok Web 搜索。OpenClaw 会移除末尾的斜杠，
然后向 `<baseUrl>/responses` 发起 POST 请求。除非设置了
`plugins.entries.xai.config.xSearch.baseUrl`，否则 `x_search`
会回退到相同的 `webSearch.baseUrl`。

## 相关内容

- [Web 搜索概览](/zh-CN/tools/web) -- 所有提供商和自动检测
- [Web 搜索中的 x_search](/zh-CN/tools/web#x_search) -- 通过 xAI 提供一流的 X 搜索
- [Gemini 搜索](/zh-CN/tools/gemini-search) -- 通过 Google 信息支撑生成 AI 综合答案
