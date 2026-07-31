---
read_when:
    - 在同一台计算机上运行多个 Gateway 网关
    - 每个 Gateway 网关都需要隔离的配置、状态和端口
summary: 在一台主机上运行多个 OpenClaw Gateway 网关（隔离、端口和配置文件）
title: 多个 Gateway 网关
x-i18n:
    generated_at: "2026-07-26T06:15:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

大多数设置只需要一个 Gateway 网关——单个 Gateway 网关可以处理多个消息连接和智能体。仅当需要更强的隔离或冗余（例如救援 Bot）时，才使用隔离的配置文件/端口运行多个独立的 Gateway 网关。

## 救援 Bot 快速开始

最简单的救援 Bot 设置：

- 让主 Bot 继续使用默认配置文件。
- 在 `--profile rescue` 上运行救援 Bot，并为其配置独立的 Telegram Bot 令牌。
- 为救援 Bot 使用不同的基础端口，例如 `19789`。

这样，如果主 Bot 停止运行，救援 Bot 仍能进行调试或应用配置更改。基础端口之间至少留出 20 个端口，以确保派生的浏览器/CDP 端口绝不会发生冲突。

```bash
# 救援 Bot（独立的 Telegram Bot、独立的配置文件、端口 19789）
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

如果主 Bot 已在运行，通常只需执行这些操作。如果新手引导已经安装了救援服务，请跳过最后的 `gateway install`。

在 `openclaw --profile rescue onboard` 期间：

- 使用专供救援账户使用的独立 Telegram Bot 令牌（便于限制为仅操作员可用，独立于主 Bot 的渠道/应用安装，并提供简单的基于私信的恢复路径）。
- 保留 `rescue` 配置文件名称。
- 使用至少比主 Bot 高 20 的基础端口。
- 接受默认的救援工作区，除非你已经自行管理了一个工作区。

### `--profile rescue onboard` 会更改什么

`--profile rescue onboard` 会运行常规新手引导流程，但会将所有内容写入独立的配置文件，因此救援 Bot 拥有自己的：

- 配置文件/配置文件
- 状态目录
- 工作区（默认：`~/.openclaw/workspace-rescue`）
- 托管服务名称
- 基础端口（以及派生端口）
- Telegram Bot 令牌

除此之外，提示与常规新手引导相同。

## 常规多 Gateway 网关设置

同样的隔离模式适用于一台主机上的任意两个或多个 Gateway 网关——为每个额外的 Gateway 网关分配独立的命名配置文件和基础端口：

```bash
# 主 Gateway 网关（默认配置文件）
openclaw setup
openclaw gateway --port 18789

# 额外的 Gateway 网关
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

两边都使用命名配置文件也可以：

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

服务也遵循相同的模式：

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

将救援 Bot 快速开始用于备用操作员通道；将常规配置文件模式用于跨不同渠道、租户、工作区或运维角色运行多个长期存在的 Gateway 网关。

## 隔离检查清单

每个 Gateway 网关实例都应为以下设置使用唯一值：

| 设置                      | 用途                              |
| ---------------------------- | ------------------------------------ |
| `OPENCLAW_CONFIG_PATH`       | 每个实例的配置文件             |
| `OPENCLAW_STATE_DIR`         | 每个实例的会话、凭据和缓存 |
| `agents.defaults.workspace`  | 每个实例的工作区根目录          |
| `gateway.port`（或 `--port`） | 每个实例使用唯一值                  |
| 派生的浏览器/CDP 端口    | 见下文                            |

共享其中任何一项都会导致配置、状态或端口冲突。即使
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` 跳过了每个配置的单实例限制，Gateway 网关启动时
也会强制要求状态目录的所有权唯一。

## 端口映射（派生）

基础端口 = `gateway.port`（或 `OPENCLAW_GATEWAY_PORT` / `--port`）。

- 浏览器控制服务端口 = 基础端口 + 2（仅 local loopback）。
- Canvas 主机由 Gateway 网关 HTTP 服务器自身提供服务（与 `gateway.port` 使用相同端口）。
- 浏览器配置文件的 CDP 端口从 `browser control port + 9` 到 `+ 108` 自动分配。

如果在配置或环境变量中覆盖其中任何端口，则必须确保每个实例使用唯一值。

## 浏览器/CDP 注意事项（常见陷阱）

- **不要**在多个实例上将 `browser.cdpUrl` 固定为相同的值。
- 每个实例都需要独立的浏览器控制端口和 CDP 范围（从其 Gateway 网关端口派生）。
- 如需显式指定 CDP 端口，请为每个实例设置 `browser.profiles.<name>.cdpPort`。
- 对于远程 Chrome，请使用 `browser.profiles.<name>.cdpUrl`（每个配置文件、每个实例分别设置）。

## 手动环境变量示例

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## 快速检查

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` 会捕获旧安装遗留的 launchd/systemd/schtasks 服务。
- 只有当你有意运行多个隔离的 Gateway 网关，或者 OpenClaw 无法证明可访问的探测目标是同一个 Gateway 网关时，才会出现 `gateway probe` 警告文本（例如 `multiple reachable gateway identities detected`），这是预期行为。指向同一 Gateway 网关的 SSH 隧道、代理 URL 或已配置的远程 URL，都属于一个具有多个传输方式的 Gateway 网关，即使各传输端口不同也是如此。

## 相关内容

- [Gateway 网关运行手册](/zh-CN/gateway)
- [Gateway 网关锁](/zh-CN/gateway/gateway-lock)
- [配置](/zh-CN/gateway/configuration)
