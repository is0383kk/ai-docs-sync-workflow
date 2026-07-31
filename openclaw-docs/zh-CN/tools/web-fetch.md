---
read_when:
    - 你想要获取一个 URL 并提取可读内容
    - 你需要配置 `web_fetch` 或其 Firecrawl 回退机制
    - 你想了解 `web_fetch` 的限制和缓存机制
sidebarTitle: Web Fetch
summary: web_fetch 工具 —— 通过 HTTP 获取内容并提取可读文本
title: 网页获取
x-i18n:
    generated_at: "2026-07-26T06:05:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` 执行普通的 HTTP GET，并提取可读内容（将 HTML 转换为
Markdown 或文本）。它**不会**执行 JavaScript。对于大量依赖 JS 的网站或
受登录保护的页面，请改用 [Web 浏览器](/zh-CN/tools/browser)。

## 快速开始

默认启用，无需配置：

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## 工具参数

<ParamField path="url" type="string" required>
要获取的 URL。仅支持 `http(s)`。
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
提取主要内容后的输出格式。
</ParamField>

<ParamField path="maxChars" type="number">
将输出截断至此字符数。限制为 `tools.web.fetch.maxCharsCap`。
</ParamField>

## 结果

`web_fetch` 返回一个封闭的结构化结果，其中包含以下字段：

- 请求元数据：`url`、`finalUrl`、`status`、`extractMode` 和 `extractor`
- 可选响应元数据：`contentType`、`title` 和 `warning`（不存在时省略）
- 封装内容元数据：`externalContent`、`truncated`、`length`、`rawLength`、
  `fetchedAt`、`tookMs` 和 `text`
- 缓存命中时的可选 `cached: true`
- 截断内容写入私有临时文件时的可选 `spill: { path, chars, truncated? }`；
  仅当该文件包含部分源内容时，才会提供 `truncated`

`length` 是封装后 `text` 的长度。`rawLength` 是外部内容封装前
所提取内容的长度。

## 工作原理

<Steps>
  <Step title="获取">
    使用类似 Chrome 的 User-Agent 和 `Accept-Language`
    标头发送 HTTP GET。阻止私有/内部主机名，并重新检查重定向。
  </Step>
  <Step title="提取">
    对 HTML 响应运行 Readability（主要内容提取）。
  </Step>
  <Step title="回退（可选）">
    如果 Readability 失败且有可用的获取提供商，则通过
    该提供商重试（例如 Firecrawl 的 Bot 规避模式）。
  </Step>
  <Step title="缓存">
    结果会缓存 15 分钟（可配置），以减少对同一 URL 的
    重复获取。
  </Step>
</Steps>

## 进度更新

仅当获取操作在五秒后仍未完成时，`web_fetch` 才会发出一行公开进度信息：

```text
正在获取页面内容...
```

快速缓存命中和迅速的网络响应会在计时器触发前完成，因此
不会显示进度信息。取消调用会清除计时器。该进度信息仅表示渠道 UI 状态，
绝不会包含获取到的页面内容。

## 配置

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // 默认值：true
        provider: "firecrawl", // 可选；省略则自动检测
        maxChars: 20000, // 默认输出字符数；上限为 maxCharsCap
        maxCharsCap: 20000, // maxChars 参数的硬上限
        maxResponseBytes: 750000, // 截断前的最大下载大小（32000-10000000）
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // 让可信 HTTP(S) 环境代理解析 DNS
        readability: true, // 使用 Readability 提取
        userAgent: "Mozilla/5.0 ...", // 覆盖 User-Agent
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // 对使用 198.18.0.0/15 的可信伪 IP 代理选择性启用
          allowIpv6UniqueLocalRange: true, // 对使用 fc00::/7 的可信伪 IP 代理选择性启用
        },
      },
    },
  },
}
```

## Firecrawl 回退

如果 Readability 提取失败，`web_fetch` 可以回退到
[Firecrawl](/zh-CN/tools/firecrawl)，以规避 Bot 并改善提取效果：

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // 可选；省略则根据可用凭据自动检测
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // 可选；省略则使用免密钥的入门访问
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // 缓存时长（2 天）
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` 是可选的，并支持 SecretRef 对象。
旧版 `tools.web.fetch.firecrawl.*` 配置会通过 `openclaw doctor --fix`
自动迁移到 `plugins.entries.firecrawl.config.webFetch`。

<Note>
  如果配置了 Firecrawl API 密钥 SecretRef，但它无法解析且没有
  `FIRECRAWL_API_KEY` 环境变量作为回退，Gateway 网关启动会快速失败。
</Note>

<Note>
  Firecrawl 的 `baseUrl` 覆盖受到严格限制：托管流量使用
  `https://api.firecrawl.dev`；自托管覆盖必须指向私有或
  内部端点，并且仅对这些私有目标接受 `http://`。
</Note>

当前运行时行为：

- `tools.web.fetch.provider` 显式选择获取回退提供商。
- 如果省略 `provider`，OpenClaw 会从已配置凭据中自动检测第一个就绪的 Web 获取
  提供商。非沙箱隔离的 `web_fetch` 可以使用已安装的插件，这些插件需要声明
  `contracts.webFetchProviders`，并在运行时注册
  匹配的提供商。目前，官方 Firecrawl 插件提供此
  回退功能。
- 沙箱隔离的 `web_fetch` 调用允许使用内置提供商，以及
  官方 npm 或 ClawHub 来源经过验证的已安装提供商。目前允许使用
  官方 Firecrawl 插件；第三方外部获取插件仍被排除。
- 如果禁用 Readability，`web_fetch` 会直接转到选定的
  提供商回退。如果没有可用提供商，则会以关闭状态失败。

## 可信环境代理

如果你的部署要求 `web_fetch` 通过可信的出站
HTTP(S) 代理，请设置 `tools.web.fetch.useTrustedEnvProxy: true`。

在此模式下，OpenClaw 仍会在发送请求前应用基于主机名的 SSRF 检查，
但会让代理解析 DNS，而不是在本地固定 DNS。仅当代理由操作员控制，
并且会在 DNS 解析后强制执行出站策略时，才启用此功能。

<Note>
  如果未配置 HTTP(S) 代理环境变量，或者目标主机被
  `NO_PROXY` 排除，`web_fetch` 会回退到使用本地 DNS
  固定的常规严格路径。
</Note>

## 限制与安全

- `maxChars` 限制为 `tools.web.fetch.maxCharsCap`（默认值为 `20000`）
- 响应正文在解析前限制为 `maxResponseBytes`（默认值为 `750000`，限制在
  32000-10000000 范围内）；过大的响应将被截断并显示警告
- 阻止私有/内部主机名
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` 和
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` 是针对可信伪 IP 代理栈的窄范围选择性启用项；
  除非你的代理拥有这些合成地址范围并强制执行自身的目标策略，否则请勿设置
- 重定向会被检查，并受 `maxRedirects` 限制（默认值为 `3`）
- `useTrustedEnvProxy` 是显式选择性启用项，仅应为以下代理启用：
  由操作员控制，并且仍会在 DNS 解析后强制执行出站策略
- `web_fetch` 仅尽力而为——某些网站需要使用 [Web 浏览器](/zh-CN/tools/browser)

## 工具配置文件

如果使用工具配置文件或允许列表，请添加 `web_fetch` 或 `group:web`：

```json5
{
  tools: {
    allow: ["web_fetch"],
    // 或：allow: ["group:web"]  （包括 web_fetch、web_search 和 x_search）
  },
}
```

## 相关内容

- [Web 搜索](/zh-CN/tools/web)——使用多个提供商搜索 Web
- [Web 浏览器](/zh-CN/tools/browser)——针对大量依赖 JS 的网站进行完整浏览器自动化
- [Firecrawl](/zh-CN/tools/firecrawl)——Firecrawl 搜索和抓取工具
