---
read_when:
    - 你想要一个自托管的 Web 搜索提供商
    - 你想将 SearXNG 用于 `web_search`
    - 你需要一个注重隐私或与外网隔离的搜索选项
summary: SearXNG Web 搜索——自托管、无需密钥的元搜索提供商
title: SearXNG 搜索
x-i18n:
    generated_at: "2026-07-26T07:05:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cae8de9f8e2c8dd9cec615adb48da5c1fd7654bffe96c7afc1acea3effbcf1fc
    source_path: tools/searxng-search.md
    workflow: 16
---

OpenClaw 支持将 [SearXNG](https://docs.searxng.org/) 用作**自托管、无需密钥**的 `web_search` 提供商。SearXNG 是一个开源元搜索引擎，可聚合来自 Google、Bing、DuckDuckGo 和其他来源的结果。

优势：

- **免费且无限制**——无需 API key 或商业订阅
- **隐私保护 / 网络隔离**——查询绝不会离开你的网络
- **随处可用**——不受商业搜索 API 的地域限制

## 设置

<Steps>
  <Step title="安装插件">
    ```bash
    openclaw plugins install @openclaw/searxng-plugin
    ```
  </Step>
  <Step title="运行 SearXNG 实例">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    或使用你有权访问的任何现有 SearXNG 部署。有关生产环境设置，请参阅
    [SearXNG 文档](https://docs.searxng.org/)。

  </Step>
  <Step title="配置">
    ```bash
    openclaw configure --section web
    # 选择 "searxng" 作为提供商
    ```

    或设置环境变量，让自动检测发现它：

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## 配置

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

SearXNG 实例的插件级设置：

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // 可选
            language: "en", // 可选
          },
        },
      },
    },
  },
}
```

`baseUrl` 也接受 SecretRef 对象（例如 `{ source: "env", id: "SEARXNG_BASE_URL" }`）。

## 环境变量

可设置 `SEARXNG_BASE_URL` 作为配置的替代方案：

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

解析顺序：已配置的 `baseUrl` 字符串，然后是 `baseUrl` 上的内联环境变量 SecretRef，最后是 `SEARXNG_BASE_URL`。如果未设置任何配置路径，存在 `SEARXNG_BASE_URL`，并且未明确选择提供商，自动检测会选择 SearXNG。

## 插件配置参考

| 字段        | 说明                                                        |
| ------------ | ------------------------------------------------------------------ |
| `baseUrl`    | 你的 SearXNG 实例的基础 URL（必填）                       |
| `categories` | 以逗号分隔的类别，例如 `general`、`news` 或 `science` |
| `language`   | 结果的语言代码，例如 `en`、`de` 或 `fr`              |

`web_search` 工具调用还接受 `count`（1-10 条结果）、`categories` 和 `language` 作为单次调用的覆盖值。

## 注意事项

- **JSON API**——使用 SearXNG 原生的 `format=json` 端点，而非抓取 HTML
- **图片结果 URL**——当 SearXNG 返回直接图片 URL 时，图片类别的结果会包含 `img_src`
- **无需 API key**——可直接用于任何 SearXNG 实例
- **基础 URL 验证**——`baseUrl` 必须是有效的 `http://` 或 `https://` URL
- **网络防护**——`http://` 基础 URL 必须指向可信的私有主机或
  loopback 主机（公共主机必须使用 `https://`）；解析到私有/内部地址的
  `https://` 基础 URL 可获得相同的自托管许可，而解析到公共地址的
  `https://` 基础 URL 仍会受到严格的 SSRF 防护
- **自动检测顺序**——SearXNG 需要已配置的 `baseUrl`（在已具备所需凭据的提供商中，顺序为
  200）。DuckDuckGo 或 Ollama Web 搜索等无需密钥的提供商绝不会通过自动检测隐式胜出；
  只有明确选择 `provider` 时才会启用
- **自托管**——你可以控制实例、查询和上游搜索引擎
- **类别**未配置时默认为 `general`
- **类别回退**——如果非 `general` 类别的请求成功但返回零条结果，
  OpenClaw 会使用 `general` 对同一查询重试一次，然后才返回空结果集
- **结果缓存**——相同的查询（查询文本、数量、类别、语言和基础 URL 均相同）
  会在进程内缓存一段较短的 TTL
- **版本要求**——该插件声明 `minHostVersion: >=2026.6.9`

<Tip>
  要使 SearXNG JSON API 正常工作，请确保你的 SearXNG 实例已在 `settings.yml` 的
  `search.formats` 下启用 `json` 格式。
</Tip>

## 相关内容

- [Web 搜索概览](/zh-CN/tools/web)——所有提供商和自动检测
- [DuckDuckGo 搜索](/zh-CN/tools/duckduckgo-search)——另一个无需密钥的提供商
- [Brave 搜索](/zh-CN/tools/brave-search)——提供免费层级的结构化结果
