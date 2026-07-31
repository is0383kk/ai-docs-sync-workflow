---
read_when:
    - 添加由智能体控制的浏览器自动化功能
    - 调试 OpenClaw 为何会干扰你自己的 Chrome
    - 在 macOS 应用中实现浏览器设置和生命周期管理
summary: 集成式浏览器控制服务和操作命令
title: 浏览器（由 OpenClaw 管理）
x-i18n:
    generated_at: "2026-07-26T06:35:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3afa2dda17520ae6c53fe3f1a7a12e7ca8a1414b2c12b79cf4a09ac8906bb3ca
    source_path: tools/browser.md
    workflow: 16
---

OpenClaw 可以运行由智能体控制的**专用 Chrome/Brave/Edge/Chromium 配置文件**。它通过 Gateway 网关内的小型本地控制服务（仅限环回）运行，并与你的个人浏览器隔离。

- 可以将其视为一个**仅供智能体使用的独立浏览器**。`openclaw` 配置文件绝不会接触你的个人浏览器配置文件。
- 智能体在这个隔离环境中打开标签页、读取页面、点击和输入。
- 内置的 `user` 配置文件则通过 Chrome DevTools MCP 连接到你真实的已登录 Chrome 会话。

## 你将获得

- 一个名为 **openclaw** 的独立浏览器配置文件（默认使用橙色强调色）。
- 确定性的标签页控制（列出/打开/聚焦/关闭）。
- 智能体操作（点击/输入/拖动/选择）、快照、屏幕截图和 PDF。
- 由 Playwright 支持的配置文件会将直接附件导航保存到托管下载目录，并在最终 URL 策略验证后返回 `{ url, suggestedFilename, path }` 元数据。
- 当操作立即启动一个或多个下载时，由 Playwright 支持的智能体操作会返回包含相同托管元数据的 `downloads` 数组。
- 内置的 `browser-automation` skill，会在启用浏览器插件时指导智能体使用快照、稳定标签页、过期引用和手动阻塞项恢复循环。
- 可选的多配置文件支持（`openclaw`、`work`、`remote` 等）。

此浏览器**并非**你的日常浏览器，而是用于智能体自动化和验证的安全隔离环境。

在 macOS 上，你可以明确地将 Cookie 从 Chrome 系浏览器的系统配置文件复制到单独的托管配置文件中。托管浏览器仍使用自己的用户数据目录；只会复制选定的 Cookie，本地存储和 IndexedDB 则不会复制。有关导入命令和限制，请参阅[配置文件](#profiles-multi-browser)或 [`openclaw browser` CLI 参考](/zh-CN/cli/browser)。

## 快速开始

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw doctor --deep
openclaw browser --browser-profile openclaw status
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

“浏览器已禁用”表示插件或 `browser.enabled` 已关闭；请参阅[配置](#configuration)和[插件控制](#plugin-control)。

如果完全缺少 `openclaw browser`，或智能体表示浏览器工具不可用，请直接跳转至[缺少浏览器命令或工具](#missing-browser-command-or-tool)。

## 插件控制

默认的 `browser` 工具是内置插件。禁用它后，可以换用另一个注册相同 `browser` 工具名称的插件：

```json5
{
  plugins: {
    entries: {
      browser: {
        enabled: false,
      },
    },
  },
}
```

默认情况下，`plugins.entries.browser.enabled` **和** `browser.enabled=true` 都必须启用。只禁用插件会将 `openclaw browser` CLI、`browser.request` Gateway 网关方法、智能体工具和控制服务作为一个整体移除；你的 `browser.*` 配置会保持不变，以供替代插件使用。

更改浏览器配置后需要重启 Gateway 网关，以便插件重新注册其服务。

## 智能体指南

工具配置文件说明：`tools.profile: "coding"` 包含 `web_search` 和 `web_fetch`，但不包含完整的 `browser` 工具。要允许智能体或派生的子智能体使用浏览器自动化，请在配置文件阶段添加 browser：

```json5
{
  tools: {
    profile: "coding",
    alsoAllow: ["browser"],
  },
}
```

对于单个智能体，请使用 `agents.entries.*.tools.alsoAllow: ["browser"]`。
仅设置 `tools.subagents.tools.allow: ["browser"]` 并不足够，因为子智能体策略会在配置文件筛选后应用。

浏览器插件提供两个层级的智能体指南：

- `browser` 工具描述包含精简且始终生效的约定：选择正确的配置文件、让引用保持在同一标签页中、使用 `tabId`/标签定位标签页，并为多步骤工作加载浏览器 skill。
- 内置的 `browser-automation` skill 包含更完整的操作循环：先检查状态/标签页、为任务标签页添加标签、操作前创建快照、UI 变化后重新创建快照、对过期引用尝试恢复一次，并将登录/双重身份验证/验证码或摄像头/麦克风阻塞项报告为需要手动操作，而不是进行猜测。

启用插件后，插件内置的 Skills 会列在智能体的可用 Skills 中。完整的 skill 指令按需加载，因此常规轮次无需承担全部 token 成本。

## 缺少浏览器命令或工具

如果升级后无法识别 `openclaw browser`、缺少 `browser.request`，或智能体报告浏览器工具不可用，通常是因为 `plugins.allow` 列表遗漏了 `browser`，且不存在根级 `browser` 配置块。请添加：

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

显式的根级 `browser` 块（`browser` 下的任意键，例如 `browser.enabled=true` 或 `browser.profiles.<name>`）会激活内置浏览器插件，即使 `plugins.allow` 有严格限制也是如此，这与内置渠道配置的行为一致。`plugins.entries.browser.enabled=true` 和 `tools.alsoAllow: ["browser"]` 本身不能替代允许列表成员资格。完全移除 `plugins.allow` 也会恢复默认设置。

## 配置文件：`openclaw`、`user`、`chrome`

- `openclaw`：托管的隔离浏览器（无需扩展程序）。
- `user`：内置的 Chrome DevTools MCP 连接配置文件，用于连接你**真实的已登录 Chrome** 会话。OpenClaw 首次连接时，Chrome 会显示阻塞式的“Allow remote debugging?”提示，因此必须有人在电脑旁操作。
- `chrome`：内置的 [Chrome 扩展程序](/zh-CN/tools/chrome-extension)配置文件，用于连接你**真实的已登录 Chrome** 会话。即使无人守在电脑旁，也可以通过手机使用，因为它通过 OpenClaw 浏览器扩展程序驱动标签页，而不是通过远程调试端口，因此不会出现“Allow remote debugging?”提示。

对于智能体浏览器工具调用：

- 默认：使用隔离的 `openclaw` 浏览器。
- 当现有登录会话很重要且用户**不在电脑旁**时（Telegram、WhatsApp 等），优先使用 `profile="chrome"`（扩展程序）。
- 当现有登录会话很重要且用户**在电脑旁**、可以批准连接提示时，优先使用 `profile="user"`（Chrome MCP）。
- 当你需要指定特定浏览器模式时，`profile` 是显式覆盖项。

如果希望默认使用托管模式，请设置 `browser.defaultProfile: "openclaw"`。

## 配置

浏览器设置位于 `~/.openclaw/openclaw.json` 中。

```json5
{
  browser: {
    enabled: true, // default: true
    evaluateEnabled: true, // default: true; false disables act:evaluate (arbitrary JS)
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // opt in only for trusted private-network access
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    // cdpUrl: "http://127.0.0.1:18792", // legacy single-profile override
    tabCleanup: {
      enabled: true, // default: true
    },
    // snapshotDefaults: { mode: "efficient" }, // default snapshot mode when the caller omits one
    defaultProfile: "openclaw",
    color: "#FF4500",
    headless: false,
    noSandbox: false,
    attachOnly: false,
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        headless: true,
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: {
        driver: "existing-session",
        attachOnly: true,
        color: "#00AA00",
      },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
  },
}
```

当调用方未传递显式的 `snapshotFormat` 或 `mode` 时，`browser.snapshotDefaults.mode: "efficient"` 会更改默认的 `snapshot` 提取模式；有关每次调用的快照选项，请参阅[浏览器控制 API](/zh-CN/tools/browser-control)。

### 标签页清理所有权

会话标签页清理仅适用于 OpenClaw 浏览器工具使用 `action: "open"` 创建的标签页。OpenClaw 不会接管已打开、由用户打开或所有权未知的标签页。`browser.tabCleanup` 块控制主会话的定期空闲清理和数量上限清理；禁用它不会禁用显式的会话生命周期清理。

对于主机本地打开的标签页，具有稳定原生 CDP 目标和浏览器标识的所有权会存储在共享 SQLite 状态中。这些记录可在 Gateway 网关重启后保留，并继续适用于 `/new` 和其他会话生命周期清理；会话生命周期清理包括子智能体、定时任务和 ACP 会话结束。工具侧目标为原生 CDP 目标的记录在重启后也仍可参与空闲清理和每会话数量上限清理。Chrome MCP 目标句柄仅在进程内有效，因此冷启动后的现有会话记录会等待生命周期清理，而不会冒险执行空闲清理，因为重启后无法安全地确定活动归属。只要 OpenClaw 能同时解析原生目标和稳定的浏览器标识，此持久化路径就可以覆盖 OpenClaw 托管的配置文件、常规远程 CDP 配置文件，以及显式设置了 `cdpUrl` 的现有会话配置文件。在关闭持久化记录之前，OpenClaw 会验证配置的配置文件和浏览器实例是否仍然匹配。

Chrome MCP `--autoConnect`、`/json/version` 响应中缺少稳定浏览器标识的 CDP 端点，以及无法解析原生目标的打开操作，仍采用仅限进程内的尽力跟踪方式。它们可以在相应 Gateway 网关进程运行时清理，但不会在 Gateway 网关重启后自动关闭。在持久化跟踪可用之前已打开的标签页不会被追溯接管；请手动关闭这些标签页。

清理采用尽力而为的方式，不保证每个符合条件的标签页都会立即关闭。短暂的所有权检查或关闭失败会让持久化清理保持待处理状态，以便稍后重试。重试并非没有上限：当浏览器持续无法访问且标签页已超过一天未使用时，跟踪行会被移除，防止持久化存储被永远无法再次验证的标签页占满。

### 屏幕截图视觉理解（支持纯文本模型）

当主模型仅支持文本（不支持视觉/多模态）时，浏览器屏幕截图会返回模型无法读取的图像块。浏览器屏幕截图会复用现有的图像理解配置，因此为媒体理解配置的图像模型可以将屏幕截图描述为文本，无需任何浏览器专用模型设置。

```json5
{
  tools: {
    media: {
      image: {
        models: [
          { provider: "bytedance", model: "doubao-seed-2.0-pro" },
          // Add fallback candidates; first success wins
          { provider: "openai", model: "gpt-4o" },
        ],
      },
      // Shared media models also work when tagged for image support.
      // models: [{ provider: "openai", model: "gpt-4o", capabilities: ["image"] }],
    },
  },
  agents: {
    defaults: {
      // Existing image-model defaults are also honored.
      // imageModel: { primary: "openai/gpt-4o" },
    },
  },
}
```

**工作原理：**

1. Agent 调用 `browser screenshot`，并像往常一样将图像捕获到磁盘。
2. 浏览器工具会询问现有的图像理解运行时，能否使用已配置的媒体图像模型、共享媒体模型、图像模型默认值或由身份验证支持的图像提供商来描述屏幕截图。
3. 视觉模型返回文本描述，该描述会由
   `wrapExternalContent`（提示注入防护）包装，并以文本块而非图像块的形式返回给智能体。
4. 如果图像理解不可用、被跳过或失败，浏览器将回退为返回原始图像块。

屏幕截图图像块是私有工具结果：智能体可以检查它们，但 OpenClaw 不会自动将其附加到渠道回复中。若要共享屏幕截图，请让智能体使用消息工具显式发送。

使用现有的 `tools.media.image` / `tools.media.models` 字段配置模型回退、超时、字节限制、配置文件和提供商请求设置。

如果当前主模型已支持视觉，并且未配置显式的图像理解模型，OpenClaw 会保留常规图像结果，以便主模型直接读取屏幕截图。

<AccordionGroup>

<Accordion title="端口和可达性">

- 控制服务绑定到 local loopback，其端口派生自 `gateway.port`（默认 `18791` = Gateway 网关 + 2）。`OPENCLAW_GATEWAY_PORT` 的优先级高于 `gateway.port`；两者都会相应偏移同一组派生端口。
- 本地 `openclaw` 配置文件会从控制端口之上 9 个端口开始的范围内自动分配 `cdpPort`/`cdpUrl`（默认 `18800`-`18899`）；仅对远程 CDP 配置文件或现有会话端点附加设置这些值。未设置时，`cdpUrl` 默认为托管的本地 CDP 端口。
- 远程和 `attachOnly` CDP 可达性、WebSocket 握手以及本地托管 Chrome 启动均使用内置截止时间。
- 系统会按配置文件对托管 Chrome 的重复启动/就绪失败进行熔断。连续失败数次后，OpenClaw 会短暂暂停新的启动尝试，而不是在每次调用浏览器工具时都生成 Chromium。请修复启动问题；如果不需要浏览器，则将其禁用；或在修复后重启 Gateway 网关。

</Accordion>

<Accordion title="SSRF 策略">

- 浏览器导航和打开标签页请求会接受预检。在操作期间及有界的操作后宽限期内，受保护的 Playwright 交互（点击、坐标点击、悬停、拖动、滚动、选择、按键、输入、表单填写和求值）会在发送 HTTP 请求字节之前拦截被策略拒绝的顶层和子框架文档加载，然后尽力重新检查最终的 `http(s)` URL。
- 每次全新启动由 OpenClaw 托管的 Chrome 前，OpenClaw 都会尽力禁用网络预测，从而抑制 Chromium 针对这些被拒绝加载所产生的已观察到的推测性预连接。这属于纵深防御，而非策略边界：在控制服务重启后继续复用的浏览器以及其他浏览器后端可能不具备相同的加固措施。Playwright 路由仍然不是网络防火墙，无法拦截重定向跳转、弹出窗口的第一个请求、Service Worker 流量、有界防护窗口结束后运行的页面代码，也无法拦截所有后台/子资源路径。完整的出站隔离需要所有者侧隔离或强制执行策略的代理。
- 在严格 SSRF 模式下，也会检查远程 CDP 端点发现和 `/json/version` 探测（`cdpUrl`）。
- Gateway 网关/提供商的 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY` 和 `NO_PROXY` 环境变量不会自动代理由 OpenClaw 托管的浏览器。托管 Chrome 默认直接启动，以免提供商代理设置削弱浏览器 SSRF 检查。
- 由 OpenClaw 托管的本地 CDP 就绪探测和 DevTools WebSocket 连接会针对启动时确定的 local loopback 端点绕过托管网络代理，因此即使操作员代理阻止 local loopback 出站，`openclaw browser start` 仍然可用。
- 若要代理托管浏览器本身，请通过 `browser.extraArgs` 传递显式的 Chrome 代理标志，例如 `--proxy-server=...` 或 `--proxy-pac-url=...`。除非有意启用私有网络浏览器访问，否则严格 SSRF 模式会阻止显式浏览器代理路由。
- `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork` 默认关闭；仅当明确信任私有网络浏览器访问时才启用。
- `browser.ssrfPolicy.allowPrivateNetwork` 仍作为旧版别名受到支持。

</Accordion>

<Accordion title="配置文件行为">

- `attachOnly: true` 表示绝不启动本地浏览器；仅当已有浏览器正在运行时才附加。
- `headless` 可全局设置，也可按本地托管配置文件设置。每个配置文件的值会覆盖 `browser.headless`，因此一个本地启动的配置文件可以保持无头状态，而另一个保持可见。
- `POST /start?headless=true` 和 `openclaw browser start --headless` 请求对本地托管配置文件执行一次性无头启动，而不重写 `browser.headless` 或配置文件设置。现有会话、仅附加和远程 CDP 配置文件会拒绝此覆盖，因为 OpenClaw 不会启动这些浏览器进程。
- 在没有 `DISPLAY` 或 `WAYLAND_DISPLAY` 的 Linux 主机上，如果环境和配置文件/全局配置均未显式选择有头模式，本地托管配置文件会自动默认为无头模式。请使用无歧义的浏览器级形式 `openclaw browser --json status`；末尾的 `openclaw browser status --json` 也有效，因为 `status` 未定义自己的 `--json`。该命令会将 `headlessSource` 报告为 `env`、`profile`、`config`、`request`、`linux-display-fallback` 或 `default`。
- `OPENCLAW_BROWSER_HEADLESS=1` 强制当前进程以无头模式启动本地托管浏览器。`OPENCLAW_BROWSER_HEADLESS=0` 强制常规启动使用有头模式，并在没有显示服务器的 Linux 主机上返回可操作的错误；对于该次启动，显式的 `start --headless` 请求仍具有优先权。
- 浏览器控制路由和编程客户端会保留无显示器错误中便于理解的 `error`，并公开稳定原因 `no_display_for_headed_profile`。其 `details` 仅包含 `profile`、`requestedHeadless`、`headlessSource` 和 `displayPresent`，因此 API 客户端无需匹配消息文本即可选择正确的补救措施。
- 对于正在运行的本地托管配置文件，状态和 Doctor 会查询 Chrome 的浏览器级 CDP 端点，以获取渲染器、后端、设备/驱动程序、功能状态、驱动程序变通方案和加速视频能力。结果会针对该浏览器进程缓存，并由 `openclaw browser --json status` 完整公开。被动状态调用不会启动 Chrome。现有会话、扩展、远程 CDP 和沙箱浏览器保持独立，不会通过此托管主机路径进行检查。
- 无头托管 Chrome 仍使用保守的 `--disable-gpu` 默认值。诊断不会启用加速、添加全局加速设置，也不会授予沙箱浏览器设备访问权限。
- `executablePath` 可全局设置，也可按本地托管配置文件设置。每个配置文件的值会覆盖 `browser.executablePath`，因此不同的托管配置文件可以启动不同的 Chromium 系浏览器。两种形式均接受 `~`，用于表示你的操作系统主目录。
- `color`（顶层和每个配置文件）会为浏览器 UI 着色，以便你识别当前处于活动状态的配置文件。
- 默认配置文件为 `openclaw`（托管的独立实例）。使用 `defaultProfile: "user"` 可选择使用已登录的用户浏览器。
- 自动检测顺序：如果系统默认浏览器基于 Chromium，则使用该浏览器；否则依次为 Chrome、Brave、Edge、Chromium、Chrome Canary。
- `driver: "existing-session"` 使用 Chrome DevTools MCP 而非原始 CDP。它可以通过 Chrome MCP 自动连接进行附加；如果你已有正在运行的浏览器的 DevTools 端点，也可通过 `cdpUrl` 进行附加。
- `driver: "extension"` 通过 [OpenClaw Chrome 扩展](/zh-CN/tools/chrome-extension)驱动你已登录的 Chrome。中继拥有其 local loopback 端点，因此这些配置文件不接受 `cdpUrl`。这是唯一可在计算机前无人值守时工作的已登录浏览器模式。
- 当现有会话配置文件需要附加到非默认的 Chromium 用户配置文件（Brave、Edge 等）时，请设置 `browser.profiles.<name>.userDataDir`。此路径也接受 `~`，用于表示你的操作系统主目录。

</Accordion>

</AccordionGroup>

## 使用 Brave 或其他 Chromium 系浏览器

如果你的**系统默认**浏览器基于 Chromium（Chrome/Brave/Edge 等），OpenClaw 会自动使用它。设置 `browser.executablePath` 可覆盖自动检测。顶层和每个配置文件的 `executablePath` 值都接受 `~`，用于表示你的操作系统主目录：

```bash
openclaw config set browser.executablePath "/usr/bin/google-chrome"
openclaw config set browser.profiles.work.executablePath "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
```

或者按平台在配置中设置：

<Tabs>
  <Tab title="macOS">
```json5
{
  browser: {
    executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
  },
}
```
  </Tab>
  <Tab title="Windows">
```json5
{
  browser: {
    executablePath: "C:\\Program Files\\BraveSoftware\\Brave-Browser\\Application\\brave.exe",
  },
}
```
  </Tab>
  <Tab title="Linux">
```json5
{
  browser: {
    executablePath: "/usr/bin/brave-browser",
  },
}
```
  </Tab>
</Tabs>

每个配置文件的 `executablePath` 仅影响由 OpenClaw 启动的本地托管配置文件。`existing-session` 配置文件改为附加到已在运行的浏览器，而远程 CDP 配置文件使用 `cdpUrl` 后面的浏览器。

## 本地控制与远程控制

- **本地控制（默认）：**Gateway 网关启动 local loopback 控制服务，并可启动本地浏览器。
- **远程控制（节点主机）：**在拥有浏览器的计算机上运行节点主机；Gateway 网关会将浏览器操作代理到该主机。
- **远程 CDP：**设置 `browser.profiles.<name>.cdpUrl`（或 `browser.cdpUrl`）以附加到远程 Chromium 系浏览器。在这种情况下，OpenClaw 不会启动本地浏览器。
- 对于 local loopback 上由外部管理的 CDP 服务（例如在 Docker 中发布到 `127.0.0.1` 的 Browserless），还需设置 `attachOnly: true`。未设置 `attachOnly` 的 local loopback CDP 会被视为由 OpenClaw 托管的本地浏览器配置文件。
- `headless` 仅影响由 OpenClaw 启动的本地托管配置文件。它不会重启或更改现有会话或远程 CDP 浏览器。
- `executablePath` 遵循相同的本地托管配置文件规则。在正在运行的本地托管配置文件上更改此设置，会将该配置文件标记为需要重启/协调，以便下次启动使用新的二进制文件。

停止行为因配置文件模式而异：

- 本地托管配置文件：`openclaw browser stop` 会停止由 OpenClaw 启动的浏览器进程
- 仅附加和远程 CDP 配置文件：`openclaw browser stop` 会关闭活动控制会话，并释放 Playwright/CDP 模拟覆盖设置（视口、配色方案、区域设置、时区、离线模式及类似状态），即使 OpenClaw 并未启动浏览器进程

远程 CDP URL 可以包含身份验证信息：

- 查询令牌（例如 `https://provider.example?token=<token>`）
- HTTP Basic 身份验证（例如 `https://user:pass@provider.example`）

OpenClaw 在调用 `/json/*` 端点以及连接到 CDP WebSocket 时会保留身份验证信息。对于令牌，请优先使用环境变量或密钥管理器，而不要将其提交到配置文件中。

## 节点浏览器代理（零配置默认方式）

如果你在装有浏览器的机器上运行**节点主机**，OpenClaw 可以自动将浏览器工具调用路由到该节点，无需任何额外的浏览器配置。这是远程 Gateway 网关的默认方式。

注意：

- 节点主机通过**代理命令**公开其本地浏览器控制服务器。
- 配置文件来自节点自身的 `browser.profiles` 配置（与本地相同）。
- 无论 `allowProfiles` 如何设置，代理命令都不允许持久修改配置文件（`create-profile`、`delete-profile`、`reset-profile`）；请直接在节点上进行这些更改。
- `nodeHost.browserProxy.allowProfiles` 是可选的。将其留空可使用旧版/默认行为：所有已配置的配置文件都可通过代理访问。
- 如果设置了 `nodeHost.browserProxy.allowProfiles`，OpenClaw 会将其视为最小权限边界，用于限制代理可访问的配置文件名称。
- 如果不需要此功能，请将其禁用：
  - 在节点上：`nodeHost.browserProxy.enabled=false`
  - 在 Gateway 网关上：`gateway.nodes.browser.mode="off"`（也接受 `"auto"` 以选择单个已连接的浏览器节点，或接受 `"manual"` 以要求显式指定节点参数）

## Browserless（托管式远程 CDP）

[Browserless](https://browserless.io) 是一种托管式 Chromium 服务，通过 HTTPS 和 WebSocket 提供 CDP 连接 URL。OpenClaw 可以使用这两种形式，但对于远程浏览器配置文件，最简单的方式是使用 Browserless 连接文档中提供的直接 WebSocket URL。

示例：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "wss://production-sfo.browserless.io?token=<BROWSERLESS_API_KEY>",
        color: "#00AA00",
      },
    },
  },
}
```

注意：

- 将 `<BROWSERLESS_API_KEY>` 替换为你的真实 Browserless 令牌。
- 选择与你的 Browserless 账户匹配的区域端点（请参阅其文档）。
- 如果 Browserless 提供的是 HTTPS 基础 URL，你可以将其转换为
  `wss://` 以建立直接 CDP 连接，也可以保留 HTTPS URL，让 OpenClaw
  发现 `/json/version`。

### 同一主机上的 Browserless Docker

当 Browserless 自托管在 Docker 中，而 OpenClaw 运行在主机上时，请将 Browserless 视为外部管理的 CDP 服务：

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserless",
    profiles: {
      browserless: {
        cdpUrl: "ws://127.0.0.1:3000",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

`browser.profiles.browserless.cdpUrl` 中的地址必须可从 OpenClaw 进程访问。Browserless 还必须公布一个匹配的可访问端点；请将 Browserless 的 `EXTERNAL` 设置为同一个对 OpenClaw 公开的 WebSocket 基础地址，例如 `ws://127.0.0.1:3000`、`ws://browserless:3000`，或稳定的 Docker 私有网络地址。如果 `/json/version` 返回的 `webSocketDebuggerUrl` 指向 OpenClaw 无法访问的地址，则 CDP HTTP 可能看起来状态正常，但 WebSocket 附加仍会失败。

对于使用环回地址的 Browserless 配置文件，不要让 `attachOnly` 保持未设置状态。没有 `attachOnly` 时，OpenClaw 会将该环回端口视为本地托管浏览器配置文件，并可能报告该端口正在使用，但并非由 OpenClaw 所有。

## 直接 WebSocket CDP 提供商

某些托管式浏览器服务提供**直接 WebSocket** 端点，而不是标准的基于 HTTP 的 CDP 发现（`/json/version`）。OpenClaw 接受三种 CDP URL 形式，并自动选择正确的连接策略：

- **HTTP(S) 发现** — `http://host[:port]` 或 `https://host[:port]`。
  OpenClaw 调用 `/json/version` 来发现 WebSocket 调试器 URL，然后建立连接。不使用 WebSocket 回退。
- **直接 WebSocket 端点** — 带有 `/devtools/browser|page|worker|shared_worker|service_worker/<id>`
  路径的 `ws://host[:port]/devtools/<kind>/<id>` 或 `wss://...`。
  OpenClaw 直接通过 WebSocket 握手建立连接，并完全跳过
  `/json/version`。
- **裸 WebSocket 根地址** — 不含 `/devtools/...` 路径的
  `ws://host[:port]` 或 `wss://host[:port]`（例如 [Browserless](https://browserless.io)、
  [Browserbase](https://www.browserbase.com)）。OpenClaw 首先尝试 HTTP
  `/json/version` 发现（将协议规范化为 `http`/`https`）；
  如果发现操作返回 `webSocketDebuggerUrl`，则使用它，否则 OpenClaw
  回退到在裸根地址上直接执行 WebSocket 握手。如果公布的
  WebSocket 端点拒绝 CDP 握手，但已配置的裸根地址
  接受该握手，OpenClaw 也会回退到该根地址。这样，指向本地 Chrome 的裸 `ws://`
  仍可建立连接，因为 Chrome 仅接受来自 `/json/version` 的特定目标路径上的 WebSocket
  升级；与此同时，当托管式提供商的发现端点公布不适用于 Playwright CDP
  的短期 URL 时，它们仍可使用其根 WebSocket 端点。

`openclaw browser doctor` 使用与运行时附加相同的“优先发现、回退到 WebSocket”逻辑，因此诊断不会将能够成功连接的裸根 URL 报告为无法访问。

### Browserbase

[Browserbase](https://www.browserbase.com) 是一个用于运行无头浏览器的云平台，内置验证码破解、隐身模式和住宅代理功能。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "browserbase",
    profiles: {
      browserbase: {
        cdpUrl: "wss://connect.browserbase.com?apiKey=<BROWSERBASE_API_KEY>",
        color: "#F97316",
      },
    },
  },
}
```

注意：

- [注册](https://www.browserbase.com/sign-up)，然后从 [Overview dashboard](https://www.browserbase.com/overview) 复制你的 **API Key**。
- 将 `<BROWSERBASE_API_KEY>` 替换为你的真实 Browserbase API 密钥。
- Browserbase 会在 WebSocket 连接时自动创建浏览器会话，因此无需手动创建会话。
- 有关当前免费套餐限制和付费方案，请参阅[定价](https://www.browserbase.com/pricing)。
- 有关完整的 API 参考、SDK 指南和集成示例，请参阅 [Browserbase 文档](https://docs.browserbase.com)。

### Notte

[Notte](https://www.notte.cc) 是一个用于运行无头浏览器的云平台，内置隐身功能、住宅代理和原生 CDP WebSocket 网关。

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "notte",
    profiles: {
      notte: {
        cdpUrl: "wss://us-prod.notte.cc/sessions/connect?token=<NOTTE_API_KEY>",
        color: "#7C3AED",
      },
    },
  },
}
```

注意：

- [注册](https://console.notte.cc)，然后从控制台设置页面复制你的 **API Key**。
- 将 `<NOTTE_API_KEY>` 替换为你的真实 Notte API 密钥。
- Notte 会在 WebSocket 连接时自动创建浏览器会话，因此无需手动创建会话。WebSocket 断开连接时，该会话会被销毁。
- 有关当前免费套餐限制和付费方案，请参阅[定价](https://www.notte.cc/#pricing)。
- 有关完整的 API 参考、SDK 指南和集成示例，请参阅 [Notte 文档](https://docs.notte.cc)。

## 安全

核心要点：

- 浏览器控制仅限 local loopback；访问通过 Gateway 网关的身份验证或节点配对进行。
- 独立的 local loopback 浏览器 HTTP API **仅使用共享密钥身份验证**：
  Gateway 网关令牌的 Bearer 身份验证、`x-openclaw-password`，或使用已配置 Gateway 网关密码的 HTTP Basic 身份验证。
- Tailscale Serve 身份标头和 `gateway.auth.mode: "trusted-proxy"`
  **不能**对此独立的 local loopback 浏览器 API 进行身份验证。
- 如果已启用浏览器控制，但未配置共享密钥身份验证，OpenClaw
  会在启动时自动生成并持久化浏览器控制凭据：
  当 `gateway.auth.mode` 为 `none` 时生成令牌，当其为
  `trusted-proxy` 时生成密码（通过 `gateway.auth.password` 持久化，
  以便进程外 local loopback 客户端能够解析它）。如果已为该模式
  显式配置字符串凭据，或 `gateway.auth.mode` 为 `password`，
  则会跳过自动生成。
- 如果你需要自己控制的稳定密钥，而不是自动生成的密钥，请显式配置
  `gateway.auth.token`、`gateway.auth.password`、`OPENCLAW_GATEWAY_TOKEN` 或
  `OPENCLAW_GATEWAY_PASSWORD`。

远程 CDP 建议：

- 尽可能优先使用加密端点（HTTPS 或 WSS）和短期令牌。
- 避免将长期令牌直接嵌入配置文件。
- 将 Gateway 网关和所有节点主机置于私有网络（Tailscale）中；避免公开暴露。
- 将远程 CDP URL/令牌视为密钥；优先使用环境变量或密钥管理器。

## 配置文件（多浏览器）

OpenClaw 支持多个命名配置文件（路由配置）。配置文件可以是：

- **由 OpenClaw 管理**：具有独立用户数据目录和 CDP 端口的专用 Chromium 浏览器实例
- **远程**：显式 CDP URL（在其他位置运行的 Chromium 浏览器）
- **现有会话**：通过 Chrome DevTools MCP 自动连接使用现有 Chrome 配置文件

默认值：

- 如果缺少 `openclaw` 配置文件，则会自动创建。
- `user` 配置文件是内置的，用于附加到 Chrome MCP 现有会话。
- 除 `user` 外，现有会话配置文件均需选择启用；使用 `--driver existing-session` 创建。
- 本地 CDP 端口默认从 **18800-18899** 范围分配。
- 删除配置文件时，其本地数据目录会移至废纸篓。

所有控制端点都接受 `?profile=<name>`；CLI 使用 `--browser-profile`。

## 通过 Chrome DevTools MCP 使用现有会话

OpenClaw 还可以通过官方 Chrome DevTools MCP 服务器附加到正在运行的 Chromium 浏览器配置文件。这样可以复用该浏览器配置文件中已打开的标签页和登录状态。

官方背景和设置参考：

- [Chrome 开发者文档：将 Chrome DevTools MCP 与浏览器会话配合使用](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session)
- [Chrome DevTools MCP README](https://github.com/ChromeDevTools/chrome-devtools-mcp)

内置配置文件：`user`。如果需要不同的名称、颜色或浏览器数据目录，请创建自己的自定义现有会话配置文件。

默认情况下，内置的 `user` 配置文件使用 Chrome MCP 自动连接，以默认的本地 Google Chrome 配置文件为目标。对于 Brave、Edge、Chromium 或非默认 Chrome 配置文件，请使用 `userDataDir`。`~` 会展开为你的操作系统主目录：

```json5
{
  browser: {
    profiles: {
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
    },
  },
}
```

然后在对应的浏览器中：

1. 打开该浏览器用于远程调试的检查页面。
2. 启用远程调试。
3. 保持浏览器运行，并在 OpenClaw 附加时批准连接提示。

常用检查页面：

- Chrome：`chrome://inspect/#remote-debugging`
- Brave：`brave://inspect/#remote-debugging`
- Edge：`edge://inspect/#remote-debugging`

实时附加冒烟测试：

```bash
openclaw browser --browser-profile user start
openclaw browser --browser-profile user status
openclaw browser --browser-profile user tabs
openclaw browser --browser-profile user snapshot --format ai
```

成功时的表现：

- `status` 显示 `driver: existing-session`
- `status` 显示 `transport: chrome-mcp`
- `status` 显示 `running: true`
- `tabs` 列出你已打开的浏览器标签页
- `snapshot` 返回所选实时标签页中的引用

如果附加无法正常工作，请检查：

- 目标 Chromium 内核浏览器的版本为 `144+`
- 已在该浏览器的检查页面中启用远程调试
- 浏览器显示了附加同意提示，并且你已接受
- 如果 Chrome 使用显式 `--remote-debugging-port` 启动，请将
  `browser.profiles.<name>.cdpUrl` 设置为该 DevTools 端点，而不要依赖
  Chrome MCP 自动连接
- `openclaw doctor` 会迁移旧的基于扩展程序的浏览器配置，并检查
  本地是否安装了 Chrome，以供默认自动连接配置文件使用，但它无法
  替你启用浏览器端远程调试

智能体使用方式：

- 需要使用用户已登录的浏览器状态时，请使用 `profile="user"`。
- 如果使用自定义的现有会话配置文件，请传入该配置文件的明确名称。
- 仅当用户在计算机旁可以批准附加
  提示时，才选择此模式。
- Gateway 网关或节点主机可以生成 `npx chrome-devtools-mcp@latest --autoConnect`。

注意：

- 此路径的风险高于隔离的 `openclaw` 配置文件，因为它可以
  在你已登录的浏览器会话中执行操作。
- OpenClaw 不会为此驱动程序启动浏览器；它只会进行附加。
- OpenClaw 在此使用官方 Chrome DevTools MCP `--autoConnect` 流程。如果
  设置了 `userDataDir`，则会将其原样传递，以指定该用户数据目录。
- 现有会话可以在所选主机上附加，也可以通过已连接的
  浏览器节点附加。如果 Chrome 位于其他位置且没有连接浏览器节点，请改用
  远程 CDP 或节点主机。
- Chrome MCP 目标和快照引用的作用域仅限于一个 MCP 子进程。该
  进程重启后，请再次运行 `browser tabs`，在执行特定于目标的工作前明确选择一个新的
  目标，并在使用引用前获取新快照。
  每个引用仅对其目标和最新快照有效。即使替换标签页的 URL 相同，旧别名也不会
  转移到该标签页。
- Chrome DevTools MCP 当前通过进程本地的数字页面
  ID 路由页面工具。进程作用域句柄可防止在子进程替换后复用，但在相邻工具调用之间，
  如果浏览器上下文在进程内被替换，操作仍可能被重新定向。要实现完全原子的路由，需要上游页面工具
  支持稳定的目标 ID。

### 自定义 Chrome MCP 启动

当默认的 `npx chrome-devtools-mcp@latest` 流程不符合需求时（离线主机、
固定版本、随附二进制文件），可以按配置文件覆盖所生成的 Chrome DevTools MCP 服务器：

| 字段        | 作用                                                                                                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `mcpCommand` | 用于替代 `npx` 生成的可执行文件。按原样解析；绝对路径会被采用。                                          |
| `mcpArgs`    | 逐字传递给 `mcpCommand` 的参数数组。替换默认的 `chrome-devtools-mcp@latest --autoConnect` 参数。 |

在现有会话配置文件中设置 `cdpUrl` 后，OpenClaw 会跳过
`--autoConnect`，并自动将端点转发给 Chrome MCP：

- `http(s)://...` → `--browserUrl <url>`（DevTools HTTP 发现端点）。
- `ws(s)://...` → `--wsEndpoint <url>`（直接 CDP WebSocket）。

端点标志不能与 `userDataDir` 组合使用：设置 `cdpUrl` 后，
启动 Chrome MCP 时会忽略 `userDataDir`，因为 Chrome MCP 会附加到
端点背后正在运行的浏览器，而不是打开配置文件
目录。

<Accordion title="现有会话功能限制">

与托管的 `openclaw` 配置文件相比，现有会话驱动程序受到更多限制：

- **屏幕截图** - 支持页面捕获和 `--ref` 元素捕获；不支持 CSS `--element` 选择器。页面截图或基于引用的元素截图不需要 Playwright。（在任何配置文件上，`--full-page` 都不能与 `--ref` 或 `--element` 组合使用，并非仅限现有会话。）
- **操作** - `click`、`type`、`hover`、`scrollIntoView`、`drag` 和 `select` 需要快照引用（不支持 CSS 选择器）。`click-coords` 点击可见视口坐标，不需要快照引用。`click` 仅支持鼠标左键（不支持覆盖按钮或使用修饰键）。`type` 不支持 `slowly=true`；请使用 `fill` 或 `press`。`press` 不支持 `delayMs`。`type`、`hover`、`scrollIntoView`、`drag`、`select` 和 `fill` 不支持按调用覆盖 `timeoutMs`；`evaluate` 支持。`select` 接受单个值。不支持 `batch`；请逐个发送操作。
- **等待/上传/对话框** - `wait --url` 支持精确、子字符串和 glob 模式（与托管配置文件相同）；现有会话配置文件不支持 `wait --load networkidle`（托管和原始/远程 CDP 配置文件支持）。上传钩子需要 `ref` 或 `inputRef`，每次上传一个文件，不支持 CSS `element`。对话框钩子不支持超时覆盖或 `dialogId`。
- **对话框可见性** - 当某个操作打开模态对话框时，托管浏览器操作响应会包含 `blockedByDialog` 和 `browserState.dialogs.pending`；快照也会包含待处理对话框状态。对话框处于待处理状态时，请使用 `browser dialog --accept/--dismiss --dialog-id <id>` 响应。在 OpenClaw 之外处理的对话框会显示在 `browserState.dialogs.recent` 下。
- **仅限托管模式的功能** - PDF 导出、下载拦截和 `responsebody` 仍需要托管浏览器路径。

</Accordion>

## 隔离保证

- **专用用户数据目录**：绝不会接触你的个人浏览器配置文件。
- **专用端口**：避开 `9222`，防止与开发工作流冲突。
- **确定性标签页控制**：`tabs` 首先返回 `suggestedTargetId`，然后返回
  稳定的 `tabId` 句柄（例如 `t1`）、可选标签和原始 `targetId`。
  智能体应复用 `suggestedTargetId`；原始 ID 仍可用于
  调试和兼容性。

## 浏览器选择

在本地启动时，OpenClaw 会选择第一个可用的浏览器：

1. Chrome
2. Brave
3. Edge
4. Chromium
5. Chrome Canary

可以使用 `browser.executablePath` 覆盖此设置。

平台：

- macOS：检查 `/Applications` 和 `~/Applications`。
- Linux：检查 `/usr/bin`、
  `/snap/bin`、`/opt/google`、`/opt/brave.com`、`/usr/lib/chromium` 和
  `/usr/lib/chromium-browser` 下常见的 Chrome/Brave/Edge/Chromium 位置，以及
  `PLAYWRIGHT_BROWSERS_PATH` 或 `~/.cache/ms-playwright` 下由 Playwright 管理的 Chromium。
- Windows：检查常见的安装位置。

## 控制 API（可选）

为了便于编写脚本和调试，Gateway 网关会公开一个小型的**仅限 local loopback 的 HTTP
控制 API**，以及对应的 `openclaw browser` CLI（快照、引用、增强等待功能、
JSON 输出、调试工作流）。完整参考请参阅
[浏览器控制 API](/zh-CN/tools/browser-control)。

## 故障排查

有关 Linux 特有的问题（尤其是 snap Chromium），请参阅
[浏览器故障排查](/zh-CN/tools/browser-linux-troubleshooting)。

有关 WSL2 Gateway 网关 + Windows Chrome 分离主机设置，请参阅
[WSL2 + Windows + 远程 Chrome CDP 故障排查](/zh-CN/tools/browser-wsl2-windows-remote-cdp-troubleshooting)。

### CDP 启动失败与导航 SSRF 阻止

这两者属于不同的故障类别，并指向不同的代码路径。

- **CDP 启动或就绪失败**表示 OpenClaw 无法确认浏览器控制平面是否健康。
- **导航 SSRF 阻止**表示浏览器控制平面健康，但页面导航目标被策略拒绝。

常见示例：

- CDP 启动或就绪失败：
  - `Chrome CDP websocket for profile "openclaw" is not reachable after start`
  - `Remote CDP for profile "<name>" is not reachable at <cdpUrl>`
  - 当配置了 local loopback 外部 CDP 服务，但未设置 `attachOnly: true` 时出现
    `Port <port> is in use for profile "<name>" but not by openclaw`
- 导航 SSRF 阻止：
  - `open`、`navigate`、快照或打开标签页流程因浏览器/网络策略错误而失败，但 `start` 和 `tabs` 仍可正常工作

使用以下最小操作序列区分这两种情况：

```bash
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

结果解读方式：

- 如果 `start` 失败并显示 `not reachable after start`，请先排查 CDP 就绪问题。
- 如果 `start` 成功，但 `tabs` 失败，则控制平面仍不健康。应将其视为 CDP 可达性问题，而不是页面导航问题。
- 如果 `start` 和 `tabs` 成功，但 `open` 或 `navigate` 失败，则说明浏览器控制平面已启动，故障位于导航策略或目标页面。
- 如果 `start`、`tabs` 和 `open` 均成功，则基本的托管浏览器控制路径健康。

重要行为细节：

- 即使未配置 `browser.ssrfPolicy`，浏览器配置也默认使用故障时关闭的 SSRF 策略对象。
- 对于 local loopback `openclaw` 托管配置文件，CDP 健康检查会有意跳过对 OpenClaw 自身本地控制平面的浏览器 SSRF 可达性强制检查。
- 导航保护是独立的。`start` 或 `tabs` 成功，并不意味着后续的 `open` 或 `navigate` 目标会被允许。

安全指南：

- 默认情况下，**不要**放宽浏览器 SSRF 策略。
- 优先使用 `hostnameAllowlist` 或 `allowedHostnames` 等范围较窄的主机例外，而不是广泛开放专用网络访问权限。
- 仅在有意设为可信、需要且已审查专用网络浏览器访问权限的环境中使用 `dangerouslyAllowPrivateNetwork: true`。

## 智能体工具及控制方式

智能体可获得用于浏览器自动化的**一个工具**：

- `browser` - doctor/status/start/stop/tabs/open/focus/close/snapshot/screenshot/navigate/act

映射方式：

- `browser snapshot` 返回稳定的 UI 树（AI 或 ARIA）。
- `browser act` 使用快照中的 `ref` ID 执行点击、输入、拖动和选择操作。
- `browser screenshot` 捕获像素（完整页面、元素或带标签的引用）。
- `browser doctor` 检查 Gateway 网关、插件、配置文件、浏览器和标签页是否就绪。
- `browser` 接受：
  - `profile`，用于选择已命名的浏览器配置文件（openclaw、chrome 或远程 CDP）。
  - `target`（`sandbox` | `host` | `node`），用于选择浏览器所在的位置。
  - 在沙箱隔离的会话中，`target: "host"` 需要 `agents.defaults.sandbox.browser.allowHostControl=true`。
  - 如果省略 `target`：沙箱隔离的会话默认为 `sandbox`，非沙箱会话默认为 `host`。
  - 如果连接了支持浏览器的节点，除非固定使用 `target="host"` 或 `target="node"`，否则该工具可能会自动路由到该节点。

这可确保智能体行为具有确定性，并避免使用脆弱的选择器。

## 相关内容

- [工具概览](/zh-CN/tools) - 所有可用的智能体工具
- [沙箱隔离](/zh-CN/gateway/sandboxing) - 沙箱隔离环境中的浏览器控制
- [安全性](/zh-CN/gateway/security) - 浏览器控制的风险与安全强化
