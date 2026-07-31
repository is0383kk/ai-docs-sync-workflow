---
read_when:
    - 在 WSL2 中运行 OpenClaw Gateway 网关，同时 Chrome 位于 Windows 上
    - 在 WSL2 和 Windows 上遇到相互重叠的浏览器/Control UI 错误
    - 在分离主机设置中选择主机本地 Chrome MCP 还是原始远程 CDP
summary: 分层排查 WSL2 Gateway 网关与 Windows Chrome 远程 CDP 故障
title: WSL2 + Windows + 远程 Chrome CDP 故障排查
x-i18n:
    generated_at: "2026-07-26T06:24:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

在常见的主机分离设置中，OpenClaw Gateway 网关运行在 WSL2 内，Chrome 运行
在 Windows 上，而浏览器控制必须跨越 WSL2/Windows 边界。多个
相互独立的问题可能会同时出现（参见
[issue #39369](https://github.com/openclaw/openclaw/issues/39369)）：CDP
传输、Control UI 来源安全性以及令牌/配对都可能各自失败，
同时产生看起来相似的错误。请按顺序逐层排查
以下各层，而不要猜测哪一层出了问题。

## 首先选择正确的浏览器模式

### 选项 1：从 WSL2 到 Windows 的原始远程 CDP

使用远程浏览器配置文件，从 WSL2 指向 Windows Chrome CDP
端点。当 Gateway 网关保留在 WSL2 内、Chrome 运行在
Windows 上，并且浏览器控制需要跨越 WSL2/Windows 边界时，请选择此模式。

### 选项 2：主机本地 Chrome MCP

仅当 Gateway 网关与 Chrome 运行在同一主机上、你希望使用本地已登录的浏览器状态、
不需要跨主机浏览器传输，并且不需要 `responsebody`、
PDF 导出、下载拦截或批量操作（Chrome MCP 配置文件不
支持这些功能）时，才使用 `existing-session` 驱动程序（`user` 配置文件）。

对于 WSL2 Gateway 网关 + Windows Chrome，请使用原始远程 CDP。Chrome MCP 是
主机本地模式，并非 WSL2 到 Windows 的桥接方式。

## 工作架构

- WSL2 在 `127.0.0.1:18789` 上运行 Gateway 网关
- Windows 在普通浏览器中通过 `http://127.0.0.1:18789/` 打开 Control UI
- Windows Chrome 在端口 `9222` 上公开 CDP 端点
- WSL2 可以访问该 Windows CDP 端点
- OpenClaw 将浏览器配置文件指向可从 WSL2 访问的地址

## Control UI 的关键规则

从 Windows 打开 UI 时，请使用 Windows localhost，除非你
有意配置了 HTTPS：

```text
http://127.0.0.1:18789/
```

不要默认使用 LAN IP。通过 LAN 或 tailnet 地址使用纯 HTTP 可能会
触发与 CDP 本身无关的不安全来源/设备身份验证行为。请参阅
[Control UI](/zh-CN/web/control-ui)。

## 分层验证

从上到下排查；不要跳过任何层。修复一层后，仍可能会看到
更下层产生的其他错误。

### 第 1 层：验证 Chrome 是否正在 Windows 上提供 CDP 服务

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 及更高版本会忽略针对默认 Chrome 数据目录的
远程调试命令行开关。请使用如上所示的独立非默认数据目录。
请参阅 Chrome 的
[远程调试安全变更](https://developer.chrome.com/blog/remote-debugging-port)。
这不会使正常的已登录 Chrome 配置文件变得可远程控制。

首先从 Windows 验证 Chrome 本身：

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

如果此操作失败，请诊断下面的 Windows 监听器。此时问题尚不在
OpenClaw。

#### 更改 portproxy 前先诊断 IPv4 和 IPv6

Chromium 会先尝试将远程调试绑定到 `127.0.0.1`，只有在 IPv4 绑定失败时才回退到
`[::1]`。持续存在且监听
`127.0.0.1:9222` 的 `v4tov4` 规则可能会在 Chrome 启动前占用该端点。随后 Chrome
会回退到 `[::1]:9222`，而旧规则会将 IPv4 流量转发回
它自己的监听器，并返回空响应。

请从 Windows 检查实际监听器和代理规则，而不要根据
Chrome 版本推断：

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

对 `netstat` 中的每个 PID 使用 `tasklist /fi "PID eq <PID>"`。

- 如果 `chrome.exe` 在 `127.0.0.1` 上有响应，请删除所有同样
  监听 `127.0.0.1:9222` 的 portproxy 规则。只将 WSL2 可访问的 Windows 适配器
  地址转发到 `127.0.0.1`。
- 如果 `chrome.exe` 仅在 `[::1]` 上有响应，请使用 `v4tov6` 将 WSL2 可访问的监听器指向
  `::1`，而不是转发到未使用的 IPv4 地址：

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

将监听器绑定到 WSL2 所需的适配器地址。不要在 `0.0.0.0`、
LAN 地址或 tailnet 地址上公开 CDP 端口：CDP 会授予对
浏览器会话的控制权。

### 第 2 层：验证 WSL2 是否可以访问该 Windows 端点

从 WSL2 测试你计划在 `cdpUrl` 中使用的确切地址：

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

正常结果：

- `/json/version` 返回包含 Browser / Protocol-Version 元数据的 JSON
- `/json/list` 返回 JSON（如果没有打开页面，空数组也可以）

如果此操作失败，则可能是 Windows 尚未向 WSL2 公开该端口、地址
不适用于 WSL2 端，或者缺少防火墙/端口转发/代理配置。请先修复
该问题，再修改 OpenClaw 配置。

### 第 3 层：配置正确的浏览器配置文件

将 OpenClaw 指向可从 WSL2 访问的地址：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

注意：

- 使用 WSL2 可访问的地址，而不是仅能在 Windows 上使用的地址
- 对于外部管理的浏览器，请保留 `attachOnly: true`
- `cdpUrl` 可以是 `http://`、`https://`、`ws://` 或 `wss://`
- 当你希望 OpenClaw 发现 `/json/version` 时，请使用 HTTP(S)
- 仅当浏览器提供商为你提供直接的 DevTools
  套接字 URL 时，才使用 WS(S)
- 在期望 OpenClaw 成功前，请先使用 `curl` 测试同一 URL

### 第 4 层：单独验证 Control UI 层

从 Windows 打开 `http://127.0.0.1:18789/`，然后验证：

- 页面来源与 `gateway.controlUi.allowedOrigins` 的预期相符
- 令牌身份验证或配对配置正确
- 你没有将 Control UI 身份验证问题误当作浏览器
  问题进行调试

实用页面：[Control UI](/zh-CN/web/control-ui)。

### 第 5 层：验证端到端浏览器控制

从 WSL2 运行：

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

正常结果：

- 标签页在 Windows Chrome 中打开
- `browser tabs` 返回目标
- 后续操作（`snapshot`、`screenshot`、`navigate`）可以通过同一
  配置文件正常执行

## 常见的误导性错误

| 消息                                                                                    | 含义                                                                                                                                                                               |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | UI 来源/安全上下文问题，而不是 CDP 传输问题                                                                                                                                        |
| `token_missing`                                                                         | 身份验证配置问题                                                                                                                                                                  |
| `pairing required`                                                                      | 设备审批问题                                                                                                                                                                      |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 无法访问已配置的 `cdpUrl`                                                                                                                                          |
| 通过 portproxy 获得空 CDP 响应 / `other side closed`                               | Windows 监听器不匹配或出现自循环；检查两个 loopback 地址族和 `netsh interface portproxy show all`                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | HTTP 端点有响应，但无法打开 DevTools WebSocket                                                                                                                                    |
| 远程会话后仍残留旧的视口 / 深色模式 / 区域设置 / 离线覆盖配置          | 运行 `openclaw browser --browser-profile remote stop` 关闭会话并释放缓存的 Playwright/CDP 连接，无需重启 Gateway 网关或外部浏览器 |
| CDP 可达性检查期间超时                                                        | 通常仍是 CDP 可达性问题，或远程端点响应缓慢/无法访问                                                                                                                              |
| `Playwright page enumeration timed out after 3000ms`                                    | 远程 CDP 已连接，但其持久标签页读取停滞                                                                                                                                           |
| `No Chrome tabs found for profile="user"`                                               | 选择了本地 Chrome MCP 配置文件，但没有可用的主机本地标签页                                                                                                                       |

## 快速分类排查清单

1. Windows：`127.0.0.1` 或 `[::1]` 中的哪一个在 `/json/version` 上有响应，该
   监听器是否属于 `chrome.exe`？
2. WSL2：`curl http://WINDOWS_HOST_OR_IP:9222/json/version` 是否有效？
3. OpenClaw 配置：`browser.profiles.<name>.cdpUrl` 是否使用该确切的
   WSL2 可访问地址？
4. Control UI：你打开的是 `http://127.0.0.1:18789/`，而不是 LAN IP 吗？
5. 你是否在尝试跨 WSL2 和 Windows 使用 `existing-session`，
   而不是原始远程 CDP？

首先在 Windows 本地验证 Chrome 端点，然后从 WSL2 验证同一端点，
之后再调试 OpenClaw 配置或 Control UI 身份验证。

## 相关内容

- [浏览器](/zh-CN/tools/browser)
- [浏览器登录](/zh-CN/tools/browser-login)
- [浏览器 Linux 故障排查](/zh-CN/tools/browser-linux-troubleshooting)
