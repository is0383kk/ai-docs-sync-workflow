---
read_when:
    - 你希望智能体通过手机操控你已登录真实账号的 Chrome 浏览器
    - 你总是遇到 Chrome 的“Allow remote debugging?”提示，但桌前却没人操作
    - 你想了解通过扩展程序接管浏览器的安全模型
summary: Chrome 扩展：让 OpenClaw 控制你已登录的 Chrome，且无需远程调试提示
title: Chrome 扩展程序
x-i18n:
    generated_at: "2026-07-26T07:02:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d974f62bb5697a23dd6a6852137ce6af5a8a4a2a8ff738eec0098f259e8faa0
    source_path: tools/chrome-extension.md
    workflow: 16
---

# Chrome 扩展程序

OpenClaw Chrome 扩展程序可让智能体控制你**已登录的 Chrome
标签页**，无需启动单独的托管浏览器，也**不会**触发 Chrome
阻塞操作的“Allow remote debugging?”提示。

当你通过手机（Telegram、WhatsApp 等）操控 OpenClaw 时，这一点很重要：
[`user` 配置文件](/zh-CN/tools/browser#profiles-openclaw-user-chrome)通过
Chrome 的远程调试端口连接，这会弹出桌面同意对话框，而你不在电脑旁时无人可以
点击。该扩展程序改用 `chrome.debugger` API，
因此页面内唯一的提示是 Chrome 可关闭的“OpenClaw started debugging
this browser”横幅。

Anthropic 的 Claude in Chrome 和 OpenAI 的 Codex
Chrome 扩展程序也采用相同的架构。

## 工作原理

由三部分组成：

- **浏览器控制服务**（Gateway 网关或节点主机）：`browser`
  工具调用的 API。
- **扩展程序中继**（回环 WebSocket）：由控制服务
  在 `127.0.0.1` 上启动的小型服务器。它向
  OpenClaw 提供 Chrome DevTools Protocol 端点，并与扩展程序通信。双方都使用
  主机本地令牌进行身份验证（见下文）。
- **OpenClaw Chrome 扩展程序**（MV3）：使用 `chrome.debugger` 连接标签页、
  转发 CDP 流量并管理 **OpenClaw 标签页组**。

OpenClaw 只能查看和控制 **OpenClaw 标签页组**中的标签页。
该组是同意边界：将标签页拖入组中即可共享，将其拖出（或点击
工具栏按钮）即可立即撤销访问权限。

## 安装和配对

1. 输出未打包扩展程序的路径：

   ```bash
   openclaw browser extension path
   ```

2. 打开 `chrome://extensions`，启用 **Developer mode**，点击 **Load
   unpacked**，然后选择输出的目录。

3. 输出配对字符串：

   ```bash
   openclaw browser extension pair
   ```

4. 点击 OpenClaw 工具栏图标，并将配对字符串粘贴到弹出窗口中。
   扩展程序连接到中继后，徽标会变为 **ON**。

配对令牌是首次使用时创建并存储在状态目录
`credentials/` 下的**主机本地密钥**（模式为 `0600`）。每台运行
浏览器的计算机（Gateway 网关主机和每个浏览器节点主机）都有自己的
令牌，因此无需在计算机之间传输凭据。若要轮换令牌，请删除
`browser-extension-relay.secret` 文件并重新配对。

## 使用方法

在 `browser` 工具调用中选择内置的 `chrome` 配置文件，或将其设为
默认配置文件：

```bash
openclaw config set browser.defaultProfile chrome
```

```json5
{
  browser: {
    profiles: {
      chrome: { driver: "extension", color: "#FF4500" },
    },
  },
}
```

- 共享标签页：点击该标签页上的 OpenClaw 工具栏按钮（标签页会加入
  OpenClaw 标签页组），或将任意标签页拖入该组。
- 智能体也可以打开新标签页；这些标签页会自动加入该组。
- 撤销访问：再次点击按钮、将标签页拖出该组，或关闭
  Chrome 的调试横幅。智能体会立即失去对该标签页的访问权限。

### 标签页 Copilot 侧边面板

配对扩展程序后，点击其工具栏弹出窗口中的 **Open tab copilot**。
OpenClaw 会为该 Chrome 标签页精确配置 `sidepanel.html`；清单中没有
全局侧边面板路径。因此，每个标签页都会获得独立的面板文档、
Gateway 网关会话、消息订阅和类型化浏览器工具绑定。

该面板不会将页面 URL、标题、DOM 或可见文本放入你的
消息中。它只发送你输入的文本。浏览器操作携带一个单独的、
经 Gateway 网关身份验证的绑定，其中包含 Chrome 标签页和 CDP 目标；浏览器工具会拒绝
尝试替换该目标或使用浏览器范围操作。回复保留在面板中（`deliver: false`）；
不会继承 Telegram、Discord 或其他渠道路由。

该 Copilot 是一个已配对的专用 Gateway 网关设备，具有 `operator.read` 和
`operator.write` 权限范围。首次使用时，检查并批准其请求：

```bash
openclaw devices list
openclaw devices approve <requestId>
```

扩展程序会保留该设备身份和由 Gateway 网关签发的设备令牌，
其作用域限定为签发它们的规范 Gateway 网关端点。与其他
Gateway 网关配对会创建独立的身份、令牌和会话托管关系；绝不会跨端点复用凭据和
会话。扩展程序不会持久存储
Gateway 网关共享密钥。面板只能订阅自身的标签页会话，
Gateway 网关会在传递前筛选这些事件。

如果 Gateway 网关连接在运行过程中断开，扩展程序会继续持久保管
该运行 ID。重新连接时，它会先中止尚未解决的运行，再
重新启用任何面板，然后重新加载转录历史记录。此故障关闭步骤
可防止浏览器操作在消息传递中断期间不受察觉地继续执行。

关闭标签页会立即移除其实时订阅、中止所有可见的
运行，并将该标签页的会话标记为已归档。如果 Gateway 网关暂时
离线，扩展程序会持久保存待处理的归档操作，并仅在同一
Gateway 网关端点重新连接时重试；它绝不会向其他
Gateway 网关发送归档请求。浏览器崩溃后，下次启动时会归档
上一个浏览器实例遗留的会话。已归档的会话拒绝新任务，但
其转录记录仍可在会话历史记录中查看。浏览器 Copilot 键属于
线程会话，因此常规的会话时限和条目数维护会保留它们。
每个智能体的会话磁盘预算仍然适用（默认为 `2gb`），并且在空间紧张时可能会逐出
最旧的会话；请参阅[会话维护](/zh-CN/reference/session-management-compaction#store-maintenance-and-disk-controls)。

侧边面板目前需要由 Gateway 网关托管的扩展程序中继，或
直接远程 Gateway 网关中继。浏览器节点上的回环中继尚无法
提供类型化标签页绑定所需的节点路由，因此面板会拒绝
这种拓扑，而不是回退到浏览器范围路由。

## 将页面发送到 OpenClaw

使用工具栏弹出窗口中的 **Send page to OpenClaw**，与
OpenClaw 主会话共享可读的页面文本。你可以添加可选备注、使用页面或
所选内容的右键菜单，或按 `Alt+Shift+S`。存在当前选区时，OpenClaw 会优先
使用该选区，将共享内容作为系统事件加入队列，并立即唤醒
主会话。

该标签页无需位于 OpenClaw 标签页组中。这是一次性、
明确的共享：页面上的其他内容不会暴露，也不会授予持续的
访问权限。Google Docs 会使用你已登录的浏览器
会话导出为纯文本，无需设置 Google API。X 和 Twitter 帖子串会在移除
周边界面元素后提取。

页面文本会封装在 OpenClaw 的外部内容安全边界中。你的
可选备注会作为你自己的指令保留在该边界之外。页面文本
和选区上限约为 120,000 个字符，截短时会包含截断
标记。

当扩展程序中继由 Gateway 网关托管，并使用
同主机配对或直接 `wss://` Gateway 网关配对时，页面共享可用。节点托管的中继目前会返回
明确的错误。若要重新映射键盘快捷键，请打开
`chrome://extensions/shortcuts`。

## 远程/跨计算机

Chrome 无需在 Gateway 网关主机上运行。支持三种拓扑：

- **同一主机**（Gateway 网关和 Chrome 位于同一台计算机）：在该计算机上使用
  `openclaw browser extension pair` 进行配对。中继仅限回环访问。
  如果本地 Gateway 网关使用 TLS，请通过
  `--gateway-url wss://gateway-host.example` 明确传递其证书主机名；配对绝不会替换为回环 IP。
- **直接连接远程 Gateway 网关**（Chrome 位于你的笔记本电脑上，Gateway 网关位于 VPS 上，并且
  **笔记本电脑上没有其他组件**）：在 Gateway 网关上运行
  `openclaw browser extension pair --gateway-url wss://your-gateway.example.com`。
  它会输出一个 `wss://…/browser/extension#<secret>` 字符串；在笔记本电脑上加载并配对
  扩展程序。扩展程序通过 `wss://` **直接连接 Gateway 网关**
  ——笔记本电脑上无需安装 OpenClaw、Node 或 CLI，也无需开放入站端口。
  这是托管服务的使用路径。
- **通过浏览器节点主机**（Chrome 位于已运行 OpenClaw
  节点的计算机上）：在该节点上运行 `pair` 并在本地配对；Gateway 网关通过现有的已验证节点链路，将浏览器
  操作代理到该节点。

配对密钥按主机区分（直接连接时为 Gateway 网关的密钥），并由
Gateway 网关的 `/browser/extension` 路由验证。对于直接连接路径，请通过
TLS（`wss://`）提供 Gateway 网关，以便加密配对密钥和 CDP 流量。
该密钥保留在配对字符串的 URL 片段中，并在
WebSocket 握手期间作为子协议凭据提供，因此常规代理访问
日志不会在请求 URL 中收到它。确保所有反向代理都保留
标准的 `Sec-WebSocket-Protocol` 标头。

## 诊断

```bash
openclaw browser status --browser-profile chrome
openclaw browser doctor --browser-profile chrome
```

在扩展程序弹出窗口显示 **Connected** 之前，`doctor` 会将
**Chrome 扩展程序中继**检查报告为失败。

## 安全模型

- 中继仅绑定到回环地址；WebSocket 双方均使用
  派生令牌进行身份验证，并且扩展程序端会对来源执行 `chrome-extension://` 检查。
- 直接 Gateway 网关配对不接受请求 URL 中的中继令牌；
  内置扩展程序会改为通过 WebSocket 子协议列表携带该令牌。
- 智能体只能查看和操控 **OpenClaw 标签页组**中的标签页。你的
  其他标签页保持私密。
- 侧边面板运行受到双重作用域限制：Gateway 网关传递使用按会话设置的
  允许列表，浏览器工具则强制执行在提示之外携带的 Chrome 标签页/目标绑定。
- 与 `user`（Chrome MCP）配置文件相比，后者会在你批准远程调试提示后暴露整个
  已登录浏览器，而扩展程序会将共享范围限制在你一眼即可掌控的标签页组内。

另请参阅：[浏览器](/zh-CN/tools/browser)，了解完整的配置文件模型以及
托管的 `openclaw` 和 Chrome MCP `user` 配置文件。
