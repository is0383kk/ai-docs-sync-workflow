---
read_when:
    - 你希望对 SSRF 和 DNS 重绑定攻击实施纵深防御
    - 为 OpenClaw 运行时流量配置外部正向代理
summary: 如何通过操作员管理的过滤代理路由 OpenClaw 运行时的 HTTP 和 WebSocket 流量
title: 网络代理
x-i18n:
    generated_at: "2026-07-26T06:28:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e948189d691e2cfe32e911e24071fd77157397b510d606423ef738c2565071b5
    source_path: security/network-proxy.md
    workflow: 16
---

OpenClaw 可通过运维人员管理的正向代理路由运行时 HTTP 和 WebSocket 流量。这是一项可选的纵深防御措施：在网络边界实现集中式出站控制、更强的 SSRF 防护和目标审计能力。由于代理会在连接时（即 DNS 解析之后、打开上游连接之前）评估目标，因此也缩小了 DNS 重绑定攻击所依赖的时间窗口，即应用程序先前执行 DNS 检查与实际建立出站连接之间的间隔。统一的代理策略还为运维人员提供了一个集中位置，无需重新构建 OpenClaw 即可实施目标规则、网络分段、速率限制或出站允许列表。

OpenClaw 不会随附、下载、启动、配置或认证任何代理。你需要运行适合自身环境的代理技术；OpenClaw 会通过该代理路由自身的 HTTP 和 WebSocket 客户端。

## 配置

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
```

也可以通过环境变量设置 URL：

```bash
OPENCLAW_PROXY_URL=http://127.0.0.1:3128 openclaw gateway run
```

`proxy.proxyUrl` 的优先级高于 `OPENCLAW_PROXY_URL`。配置 URL 后会启用托管代理路由；移除这两个 URL 即会将其禁用。

| 键                   | 类型                                 | 默认值         | 说明                                                                                                                                  |
| -------------------- | ------------------------------------ | -------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `proxy.proxyUrl`     | string                               | 未设置         | `http://` 或 `https://` 正向代理 URL。嵌入 URL 的凭据会被视为敏感信息，并从快照/日志中隐去。 |
| `proxy.tls.caFile`   | string                               | 未设置         | 用于验证由私有 CA 签名的 `https://` 代理端点的 CA 证书包。                                                          |
| `proxy.loopbackMode` | `gateway-only` \| `proxy` \| `block` | `gateway-only` | 控制 loopback 绕过行为；详见下文。                                                                                         |

对于托管 Gateway 网关服务，应将 URL 存储在配置中，使其在重新安装后仍然保留，而不是依赖前台环境变量：

```bash
openclaw config set proxy.proxyUrl http://127.0.0.1:3128
openclaw gateway install --force
openclaw gateway start
```

`OPENCLAW_PROXY_URL` 环境变量回退最适合前台运行。若要将其用于已安装的服务，请将其放入服务的持久环境中（`$OPENCLAW_STATE_DIR/.env`，默认为 `~/.openclaw/.env`），然后重新安装，以便 launchd/systemd/Scheduled Tasks 获取该变量。

### 使用私有 CA 的 HTTPS 代理端点

```yaml
proxy:
  proxyUrl: https://proxy.corp.example:8443
  tls:
    caFile: /etc/openclaw/proxy-ca.pem
```

`proxy.tls.caFile` 用于验证代理端点自身的 TLS 证书。它不是目标 MITM 信任设置、客户端证书，也不能替代代理的目标策略。仅当整个 Node 进程必须从启动时信任额外的 CA（例如，企业 TLS 检查系统对每个 HTTPS 目标证书重新签名）时，才应改用 `NODE_EXTRA_CA_CERTS`——该变量作用于整个进程，并且必须在 Node 启动前设置，因此 OpenClaw 无法像应用 `proxy.tls.caFile` 那样在运行期间应用它。对于 HTTPS 代理端点信任，优先使用 `proxy.tls.caFile`：其作用域仅限于托管代理路由，而非整个进程。

```bash
openclaw config set proxy.proxyUrl https://proxy.corp.example:8443
openclaw config set proxy.tls.caFile /etc/openclaw/proxy-ca.pem
openclaw gateway run
```

## 路由工作原理

配置有效的代理 URL 后，受保护的运行时进程（`openclaw gateway run`、`openclaw node run`、`openclaw agent --local`）会通过代理路由普通 HTTP 和 WebSocket 出站流量：

```text
OpenClaw 进程
  fetch、node:http、node:https、WebSocket 客户端  -> 运维人员代理 -> 目标
```

在内部，OpenClaw 会安装 [Proxyline](https://github.com/openclaw/proxyline) 作为进程级路由运行时。它涵盖 `fetch`、基于 undici 的客户端、`node:http`/`node:https`、常见 WebSocket 客户端以及由辅助函数创建的 `CONNECT` 隧道，并会替换调用方提供的 Node HTTP agent，使显式 agent（包括 `axios`、`got`、`node-fetch` 以及类似的基于 Node agent 的客户端）无法在不引人注意的情况下绕过代理。

代理 URL 的方案描述的是从 OpenClaw 到代理的跃点，而不是到最终目标的跃点：

- `http://proxy.example:3128` — 通过明文 TCP 连接到代理；OpenClaw 会发送 HTTP 代理请求，包括针对 HTTPS 目标的 `CONNECT`。
- `https://proxy.example:8443` — OpenClaw 会与代理本身建立 TLS 连接（并验证代理的证书），然后在该会话内发送 HTTP 代理请求。

目标 TLS 与代理端点 TLS 相互独立：对于 HTTPS 目标，OpenClaw 始终要求代理建立 `CONNECT` 隧道，并通过该隧道启动目标 TLS。

代理处于活动状态时，OpenClaw 会清除 `no_proxy`/`NO_PROXY`。这些绕过列表基于目标；如果其中保留 `localhost` 或 `127.0.0.1`，SSRF 目标就能完全绕过代理。关闭时，OpenClaw 会恢复先前的代理环境并重置缓存的路由状态。

某些插件拥有自定义传输，即使进程级路由已启用，也需要单独接入代理。Telegram 的 Bot API 客户端使用自己的 HTTP/1 undici dispatcher，并单独遵循进程代理环境变量及 `OPENCLAW_PROXY_URL` 回退。

### Gateway 网关 loopback 模式

本地 Gateway 网关控制平面客户端通常连接到 loopback WebSocket，例如 `ws://127.0.0.1:18789`。`proxy.loopbackMode` 控制此流量是否绕过托管代理：

```yaml
proxy:
  proxyUrl: http://127.0.0.1:3128
  loopbackMode: gateway-only # gateway-only、proxy 或 block
```

配置 `proxyUrl` 或 `OPENCLAW_PROXY_URL` 会启用托管路由。仅在需要保留 URL 但不启用它的高级选择性退出场景中，才设置
`proxy.enabled: false`。

| 模式                     | 行为                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway-only`（默认） | OpenClaw 会将活动 Gateway 网关的 loopback authority 注册为直连例外，因此本地 Gateway 网关 WebSocket 流量无需经过代理即可连接。由于例外针对精确配置的主机/端口，因此自定义 loopback 端口也可正常工作。内置浏览器插件会为 OpenClaw 启动的托管浏览器的精确本地 CDP 就绪 URL 和 DevTools WebSocket URL 注册同类例外；内置 Ollama 记忆嵌入提供商针对其精确配置的主机本地 loopback 嵌入源提供了范围更窄且受保护的直连路径。 |
| `proxy`                  | 不注册任何 loopback 例外；Gateway 网关和 Ollama loopback 流量均通过代理。远程代理必须能够路由回 OpenClaw 主机的 loopback 服务（例如通过可访问的主机名、IP 或隧道）——标准远程代理会相对于其自身解析 `127.0.0.1`/`localhost`，而不是相对于 OpenClaw 主机解析。                                                                                                                                                                                                                |
| `block`                  | OpenClaw 会在打开套接字之前拒绝 Gateway 网关 loopback 控制平面连接和受保护的 Ollama loopback 嵌入连接。                                                                                                                                                                                                                                                                                                                                                                                                                               |

Gateway 网关控制平面绕过仅限于 `localhost` 和字面量 loopback IP URL——请使用 `ws://127.0.0.1:18789`、`ws://[::1]:18789` 或 `ws://localhost:18789`。其他主机名按普通流量进行路由。

### 容器

对于 `openclaw --container ...` 命令，设置 `OPENCLAW_PROXY_URL` 后，OpenClaw 会将其转发到以容器为目标的子 CLI。该 URL 必须能从容器内部访问——其中的 `127.0.0.1` 指容器本身，而不是主机。对于以容器为目标的命令，OpenClaw 会拒绝 loopback 代理 URL，除非设置 `OPENCLAW_CONTAINER_ALLOW_LOOPBACK_PROXY_URL=1` 以显式覆盖此检查。

## 相关代理术语

- `proxy.enabled` / `proxy.proxyUrl` — 用于运行时出站流量的出站正向代理路由。即本页所述内容。
- `gateway.auth.mode: "trusted-proxy"` — 用于访问 Gateway 网关的入站身份感知反向代理身份验证。请参阅[可信代理身份验证](/zh-CN/gateway/trusted-proxy-auth)。
- `openclaw proxy` — 用于开发和支持的本地调试代理及流量捕获检查器。请参阅 [openclaw proxy](/zh-CN/cli/proxy)。
- `tools.web.fetch.useTrustedEnvProxy` — `web_fetch` 的选择性启用项，允许运维人员控制的 HTTP(S) 环境代理执行 DNS 解析，同时默认保持严格的 DNS 固定和主机名策略。请参阅 [Web fetch](/zh-CN/tools/web-fetch#trusted-env-proxy)。
- 渠道或提供商专用代理设置 — 针对单一传输的所有者专用覆盖。若要对整个运行时实施集中式出站控制，请优先使用托管网络代理。

## 验证代理

代理的目标策略才是真正的安全边界；OpenClaw 无法验证你的代理是否拦截了正确的目标。应将其配置为：

- 仅绑定到 loopback 或受信任的私有接口，并且只有 OpenClaw 进程/主机/容器/服务账号可以访问。
- 由代理自行解析目标，并在 DNS 解析后的连接时按 IP 拦截，包括明文 HTTP 和 HTTPS `CONNECT` 隧道。
- 拒绝针对 loopback、私有地址、链路本地地址、元数据地址、多播地址、保留地址和文档地址范围的目标绕过。
- 除非完全信任 DNS 解析路径，否则避免使用主机名允许列表。
- 记录目标、决策、状态和原因——绝不记录请求正文、授权标头、Cookie 或其他机密信息。
- 将策略纳入版本控制，并将策略变更作为安全敏感变更进行审查。

从运行 OpenClaw 的同一主机/容器/服务账号进行验证：

```bash
openclaw proxy validate --proxy-url http://127.0.0.1:3128
```

使用私有 CA 的 HTTPS 代理端点：

```bash
openclaw proxy validate --proxy-url https://proxy.corp.example:8443 --proxy-ca-file /etc/openclaw/proxy-ca.pem
```

| 标志                     | 用途                                                              |
| ------------------------ | -------------------------------------------------------------------- |
| `--proxy-url <url>`      | 验证此 URL，而不是解析配置/环境变量。                   |
| `--proxy-ca-file <path>` | HTTPS 代理端点的 CA 证书包。                               |
| `--allowed-url <url>`    | 预期可成功访问的目标（可重复指定）。                        |
| `--denied-url <url>`     | 预期被阻止的目标（可重复指定）。                     |
| `--apns-reachable`       | 同时验证代理能否通过隧道传输直接的沙箱 APNs HTTP/2 探测。 |
| `--apns-authority <url>` | 覆盖使用 `--apns-reachable` 探测的 APNs 权威地址。          |
| `--timeout-ms <ms>`      | 单次请求超时时间。                                                 |
| `--json`                 | 机器可读输出。                                             |

如果没有可用的配置、环境变量或 `--proxy-url` 值，该命令会报告配置问题；在更改配置之前，可传入 `--proxy-url` 进行一次性预检。

未提供 `--allowed-url`/`--denied-url` 时，默认检查为：`https://example.com/` 必须成功，并且代理必须无法访问一个临时的 loopback 金丝雀服务器。发生传输失败时，或者收到不含该金丝雀单次运行令牌的非 2xx 响应时，loopback 检查通过；如果收到缺少令牌的 2xx 响应（表示金丝雀之外的某个服务意外成功响应），检查失败；尤其是当任何响应携带匹配令牌时，检查会失败，因为这证明代理确实转发了本应拒绝的 loopback 目标。自定义 `--denied-url` 目标没有此类金丝雀令牌，因此采用失败关闭策略：任何 HTTP 响应都视为目标可达（失败），而传输错误会报告为无法确定，而非已证实阻止，因为 OpenClaw 无法确认是代理拒绝了一个本来可达的源站，还是发生了其他错误。`--apns-reachable` 会发送一个刻意设置为无效的提供商令牌，因此 `403 InvalidProviderToken` 响应可视为隧道已到达 Apple 的证明。任何验证失败都会使该命令以 `1` 退出；文本和 JSON 输出中的代理 URL 凭据都会被脱敏。

```json
{
  "ok": true,
  "config": {
    "enabled": true,
    "proxyUrl": "http://127.0.0.1:3128/",
    "source": "override",
    "errors": []
  },
  "checks": [
    { "kind": "allowed", "url": "https://example.com/", "ok": true, "status": 200 },
    { "kind": "apns", "url": "https://api.sandbox.push.apple.com", "ok": true, "status": 403 }
  ]
}
```

手动执行 `curl` 检查（公共请求应成功；loopback 和元数据请求应由代理本身阻止——仅凭 `curl` 无法像 `openclaw proxy validate` 的内置金丝雀那样，区分代理拒绝与源站不可达）：

```bash
curl -x http://127.0.0.1:3128 https://example.com/
curl -x http://127.0.0.1:3128 http://127.0.0.1/
curl -x http://127.0.0.1:3128 http://169.254.169.254/
```

## 建议阻止的目标

这是适用于任何正向代理、防火墙或出站策略的初始拒绝列表。OpenClaw 自身的 SSRF 分类器位于 `src/infra/net/ssrf.ts` 和 `packages/net-policy/src/ip.ts`（`BLOCKED_HOSTNAMES`、`BLOCKED_IPV4_SPECIAL_USE_RANGES`、`BLOCKED_IPV6_SPECIAL_USE_RANGES`、RFC 2544 基准测试前缀，以及对 NAT64/6to4/Teredo/ISATAP/IPv4 映射形式中嵌入式 IPv4 的处理）——这些是有用的参考，但 OpenClaw 不会在你的外部代理中导出或强制执行这些规则。

| 范围或主机                                                                        | 阻止原因                                      |
| ------------------------------------------------------------------------------------ | ------------------------------------------------- |
| `127.0.0.0/8`, `localhost`, `localhost.localdomain`                                  | IPv4 loopback                                     |
| `::1/128`                                                                            | IPv6 loopback                                     |
| `0.0.0.0/8`, `::/128`                                                                | 未指定地址/本网络地址              |
| `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`                                      | RFC 1918 私有网络                         |
| `169.254.0.0/16`, `fe80::/10`                                                        | 链路本地地址，包括常见的云元数据路径 |
| `169.254.169.254`, `metadata.google.internal`                                        | 云元数据服务                           |
| `100.64.0.0/10`                                                                      | 运营商级 NAT 共享地址空间            |
| `198.18.0.0/15`, `2001:2::/48`                                                       | 基准测试范围                               |
| `192.0.0.0/24`, `192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`, `2001:db8::/32` | 特殊用途和文档示例范围              |
| `224.0.0.0/4`, `ff00::/8`                                                            | 组播                                         |
| `240.0.0.0/4`                                                                        | 保留的 IPv4 范围                                     |
| `fc00::/7`, `fec0::/10`                                                              | IPv6 本地/私有范围                         |
| `100::/64`, `2001:20::/28`                                                           | IPv6 丢弃和 ORCHIDv2 范围                  |
| `64:ff9b::/96`, `64:ff9b:1::/48`                                                     | 含嵌入式 IPv4 的 NAT64 前缀                 |
| `2002::/16`, `2001::/32`                                                             | 含嵌入式 IPv4 的 6to4 和 Teredo                |
| `::/96`, `::ffff:0:0/96`                                                             | IPv4 兼容和 IPv4 映射 IPv6              |

请添加你的云提供商或网络平台文档中列出的任何其他元数据主机或保留范围。

## 限制

| 范围                                                      | 托管代理状态                                                                                                                                     |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `fetch`, `node:http`, `node:https`、常见 WebSocket 客户端 | 配置后通过托管代理钩子路由。                                                                                                      |
| APNs 直接 HTTP/2                                           | 通过 APNs 托管 `CONNECT` 辅助程序路由。                                                                                                        |
| Gateway 网关控制平面 loopback                               | 仅对配置中完全匹配的本地 loopback Gateway 网关 URL 使用直连。                                                                                         |
| 调试代理上游转发                              | 托管代理模式启用时会禁用，除非为本地诊断显式启用。                                                             |
| IRC                                                          | 原始 TCP/TLS；托管 HTTP 代理模式不会对其进行代理。如果你的部署要求所有出站流量都经过正向代理，请设置 `channels.irc.enabled: false`。 |
| 其他原始 `net`、`tls` 或 `http2` 客户端调用              | 必须先由原始套接字防护机制完成分类，才能合入。                                                                                               |

- 这是针对 JavaScript HTTP/WebSocket 客户端的进程级覆盖，而不是操作系统级网络沙箱。
- 原始 `net`、`tls`、`http2` 套接字、原生附加组件和非 OpenClaw 子进程可能绕过 Node 级路由，除非它们继承并遵循代理环境变量。派生的 OpenClaw 子 CLI 会继承托管代理 URL 和 `proxy.loopbackMode` 状态。
- 用户的本地 WebUI 和本地模型服务器不受通用本地网络绕过机制覆盖——如有需要，请在操作员代理策略中将其加入允许列表。唯一例外是内置 Ollama 记忆嵌入提供商受保护的直连路径，其范围仅限于配置的 `baseUrl` 所指向的、与主机处于同一设备的确切 loopback 源站；LAN、tailnet、私有网络和公共 Ollama 主机仍使用托管代理。
- 托管代理模式启用时，本地调试代理的直接上游转发（用于代理请求和 `CONNECT` 隧道）默认禁用；仅应为经批准的本地诊断启用。
- OpenClaw 不会检查、测试或认证你的代理策略。请将代理策略更改视为安全敏感的运维变更。
