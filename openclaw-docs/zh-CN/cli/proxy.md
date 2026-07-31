---
read_when:
    - 你需要在部署前验证由操作员管理的代理路由
    - 你需要在本地捕获 OpenClaw 传输流量以进行调试
    - 你想要检查调试代理会话、二进制大对象或内置查询预设
summary: '`openclaw proxy` 的 CLI 参考，包括操作员管理的代理验证和本地调试代理捕获检查器'
title: 代理
x-i18n:
    generated_at: "2026-07-26T06:10:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 91583f785032bfffe455a1963804108550f6fbb735ac4de1dd91d0ca5ae0df35
    source_path: cli/proxy.md
    workflow: 16
---

# `openclaw proxy`

验证由操作员管理的代理路由，或运行本地显式调试代理并检查捕获的流量。

```bash
openclaw proxy validate [--json] [--proxy-url <url>] [--proxy-ca-file <path>] [--allowed-url <url>] [--denied-url <url>] [--apns-reachable] [--apns-authority <url>] [--timeout-ms <ms>]
openclaw proxy start [--host <host>] [--port <port>]
openclaw proxy run [--host <host>] [--port <port>] -- <cmd...>
openclaw proxy coverage
openclaw proxy sessions [--limit <count>]
openclaw proxy query --preset <name> [--session <id>]
openclaw proxy blob --id <blobId>
openclaw proxy purge
```

`validate` 对由操作员管理的正向代理执行预检。其余命令是用于传输层调查的调试工具：启动本地流量捕获代理、通过该代理运行子命令、列出捕获会话、查询流量模式、读取捕获的二进制大对象，以及清除本地捕获数据。

## 验证

按照优先级顺序检查来自 `--proxy-url`、配置（`proxy.proxyUrl`）或 `OPENCLAW_PROXY_URL` 的有效操作员管理代理 URL。如果未启用并配置任何代理，则报告配置问题；传入 `--proxy-url` 可执行一次性预检，而不修改配置。

托管代理 URL 使用 `http://` 表示普通正向代理监听器；如果 OpenClaw 必须先与代理端点建立 TLS 连接，再发送代理请求，则使用 `https://`。使用 `--proxy-ca-file` 可为该 TLS 连接信任私有 CA。

默认情况下，它会运行：

- 一次针对 `https://example.com/` 的**允许**检查（使用 `--allowed-url` 覆盖或添加，可重复指定）
- 一次针对临时回环金丝雀目标的**拒绝**检查（使用 `--denied-url` 覆盖，可重复指定）

自定义 `--denied-url` 目标采用故障关闭策略：HTTP 响应和含义不明确的传输故障均视为失败，除非你能够独立验证部署特有的拒绝信号。内置回环金丝雀目标是唯一将传输错误视为阻止证明的目标。

添加 `--apns-reachable` 还可通过代理建立 APNs HTTP/2 CONNECT 隧道，并确认沙箱 APNs 能够响应。该探测会发送故意无效的提供商令牌，因此 APNs 返回 `403 InvalidProviderToken` 响应会被视为成功的可达性信号（而非失败）。

### 选项

| 标志                     | 作用                                                                                                             |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `--json`                 | 输出机器可读的 JSON                                                                                        |
| `--proxy-url <url>`      | 验证此 `http://`/`https://` 代理 URL，而不是配置或环境变量中的 URL                                              |
| `--proxy-ca-file <path>` | 信任此 PEM CA 文件，以对 HTTPS 代理端点执行 TLS 验证                                             |
| `--allowed-url <url>`    | 预期可通过代理成功访问的目标（可重复指定）                                                     |
| `--denied-url <url>`     | 预期被代理阻止的目标（可重复指定）                                                       |
| `--apns-reachable`       | 同时验证能否通过代理访问沙箱 APNs HTTP/2                                                     |
| `--apns-authority <url>` | 要探测的 APNs 权威地址（默认值为 `https://api.sandbox.push.apple.com`；生产环境为 `https://api.push.apple.com`） |
| `--timeout-ms <ms>`      | 单次请求超时时间                                                                                                |

代理配置或目标检查失败时，以代码 1 退出。

有关部署指导和拒绝语义，请参阅[网络代理](/zh-CN/security/network-proxy)。

## 调试代理

`start` 启动本地流量捕获代理，并输出其 URL、CA 证书路径和捕获数据库路径；使用 Ctrl+C 停止。除非设置了 `--host`，否则默认绑定到 `127.0.0.1`。

`run` 启动本地调试代理，然后在应用代理环境变量的情况下，于独立的捕获会话中运行 `<cmd...>`（位于 `--` 之后）。

调试代理的直接上游转发会打开上游套接字以进行诊断。当 OpenClaw 托管代理模式处于活动状态时，默认禁用代理请求和 CONNECT 隧道的直接转发；仅在获准的本地诊断中设置 `OPENCLAW_DEBUG_PROXY_ALLOW_DIRECT_CONNECT_WITH_MANAGED_PROXY=1`。

`coverage` 输出一份 JSON 报告（`summary` + 各传输协议的 `entries`），说明哪些传输协议已被捕获、仅支持代理或尚未覆盖。

`sessions` 列出最近的捕获会话（`--limit`，默认为 20）。

`query --preset <name>` 对捕获的流量运行内置查询，也可选择限定到 `--session <id>`。预设：

- `double-sends`
- `retry-storms`
- `cache-busting`
- `ws-duplicate-frames`
- `missing-ack`
- `error-bursts`

`blob --id <blobId>` 输出所捕获负载二进制大对象的原始内容。

`purge` 删除所有捕获的流量元数据和二进制大对象。捕获内容属于本地调试数据；完成后请将其清除。

## 相关内容

- [CLI 参考](/zh-CN/cli)
- [网络代理](/zh-CN/security/network-proxy)
- [受信任代理身份验证](/zh-CN/gateway/trusted-proxy-auth)
