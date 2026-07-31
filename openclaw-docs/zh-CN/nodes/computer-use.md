---
read_when:
    - 允许 Gateway 网关智能体查看并控制已配对的桌面设备
    - 计算机使用的启用、权限或安全措施
    - 扩展 computer.act 节点命令或其执行器
summary: 通过计算机工具和 `computer.act` 节点命令实现基于能力的桌面控制
title: 计算机使用
x-i18n:
    generated_at: "2026-07-26T06:20:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: df8ce87e607ce1b22d91e4ed8702d500bccd4d4f59dab7b0eafac565e730d48a
    source_path: nodes/computer-use.md
    workflow: 16
---

计算机使用功能允许 Gateway 网关智能体查看并控制已配对且具备相应能力的桌面设备。资格基于能力：已连接的节点必须同时声明 `computer.act` 和 `screen.snapshot`，后者的结果必须包含 `displayFrameId`。该工具会捕获屏幕截图作为参考帧，然后通过危险的 `computer.act` 命令驱动指针和键盘。操作集遵循 Anthropic 核心计算机使用操作；不提供可选的 `computer_20251124` 缩放功能。支持视觉的模型通过内置的 `computer` 智能体工具来驱动它。

智能体发出统一命令 `computer.act`；它无法得知节点如何执行该命令。内置的 macOS 应用使用嵌入式 Peekaboo 服务及少量 CoreGraphics 原语在进程内处理该命令（需具备正确的 TCC 权限，无需额外进程）。Windows 和 Linux 可使用可选的实验性 `cua-computer` 插件，并单独安装 `cua-driver` 二进制文件。两种执行程序使用相同的配对和启用策略。

## 要求

- 一个已配对并连接的节点，同时声明 `computer.act` 和 `screen.snapshot`，且 `screen.snapshot` 返回 `displayFrameId`。
- **macOS 执行程序：**启用应用设置 **Allow Computer Control**（默认：关闭）。
- **macOS 执行程序：**向 OpenClaw 授予 **Accessibility** 权限（用于注入指针/键盘输入）和 **Screen Recording** 权限（用于 `screen.snapshot`）。
- **Windows/Linux 执行程序：**启用内置的 `cua-computer` 插件，并安装兼容的 `cua-driver` 0.10.x 可执行文件。
- 在 Gateway 网关上启用 `computer.act` 命令（该命令具有危险性，默认未启用）。
- 支持视觉的智能体模型。
- 公开 `computer` 的工具策略。默认的 `coding` 配置文件不公开该工具。将 `computer` 添加到 `tools.alsoAllow`；沙箱隔离的智能体还需将其添加到 `tools.sandbox.tools.alsoAllow`。

## `computer` 智能体工具

内置的 `computer` 工具每次调用接受一个操作。坐标是最近一次屏幕截图中的非负整数像素；节点会将其映射为显示器上的点。坐标操作必须回传屏幕截图结果中的 `frameId`，并且显式提供的 `screenIndex` 必须与该帧匹配。OpenClaw 还会把节点随屏幕截图签发的显示器标识传递给操作，因此显示器重新连接或几何布局发生变化时，操作会以安全方式失败，而不是悄然重新定位到相同索引。这些检查会拒绝猜测的令牌，以及来自其他已传递帧或显示器的令牌。令牌不保证时效性：捕获后，同一显示器上的应用仍可能更改像素，因此每当场景可能已发生变化时，都应重新截取屏幕截图。

- 读取：`screenshot`。
- 指针：`left_click`、`right_click`、`middle_click`、`double_click`、`triple_click`、`mouse_move`、`left_click_drag`（带 `startCoordinate`）、`left_mouse_down`、`left_mouse_up`。
- 滚动：`scroll`，使用 `scrollDirection`（`up|down|left|right`）和 `scrollAmount`（滚轮刻度）。
- 键盘：`type`（文本）、`key`（例如 `cmd+shift+t` 或 `Return` 的组合键）、`hold_key`（按住 `text` 组合键 `duration` 秒）。
- 节奏控制：`wait`（`duration` 秒）。

修饰键通过点击和滚动操作的 `text` 字段传递（`shift`、`ctrl`、`alt`、`cmd`）。执行输入操作后，工具会返回新的屏幕截图，以便模型观察结果。如果连接了多个支持计算机使用的节点，请显式传入 `node`。

屏幕截图仅供**模型使用**：绝不会自动传递到聊天渠道。应将屏幕上的所有内容视为不可信输入；该工具会警告模型，不要遵循与用户请求冲突的屏幕指令。

## Windows 和 Linux（实验性，通过 cua-driver）

内置的 `cua-computer` 插件为 Windows 和 Linux 节点主机提供实验性执行程序。该插件默认禁用，并要求使用预发布的 0.10.x 驱动程序契约：

1. 从[上游发布页面](https://github.com/trycua/cua/releases)安装 `cua-driver` 0.10.x 二进制文件，并使其可通过 `PATH` 访问。若要使用其他可执行文件位置，请设置 `plugins.entries.cua-computer.config.driverPath`。
2. 启用插件：

   ```bash
   openclaw plugins enable cua-computer
   ```

3. 从交互式桌面会话中启动 `openclaw node run`。首次收到捕获或操作请求时，插件会延迟启动本地驱动程序守护进程。

该执行程序目前只能控制主显示器。X11/XWayland 是 Linux 的首选路径。原生 Wayland 仍需在上游显式启用：启动节点前自行设置 `CUA_DRIVER_RS_ENABLE_WAYLAND`；OpenClaw 绝不会自动设置它。上游原生 Wayland 输入路径不支持 KDE/KWin。`hold_key`、`left_mouse_down` 和 `left_mouse_up` 不可用，因为 cua-driver 0.10.x 没有跨平台、桌面范围的按住输入契约。两个平台均不支持按住修饰键时滚动和拖动，Linux 也不支持按住修饰键时点击。`key` 操作接受命名按键、字母和修饰键组合（例如 `cmd+c` 或 `Return`）；数字键和标点符号键会被拒绝，因为驱动程序会丢失其依赖键盘布局的 Shift 状态，因此请改用 `type` 操作发送相应文本。在一次 `type_text` 驱动程序调用期间，无法中途取消文本输入。

由于 cua-driver 不会报告稳定的显示器标识，帧授权会绑定到驱动程序连接以及当前主显示器的几何布局。守护进程或会话重新连接会使未使用的帧失效，但如果在保持连接的情况下将主显示器替换为几何布局相同的显示器，则无法检测到；此执行程序应优先使用稳定的单显示器会话。

OpenClaw 会为其管理的 `mcp` 和 `serve` 进程禁用 cua-driver 遥测和更新检查。它不会下载或更新驱动程序二进制文件。

### 故障排查

`cua-computer` 执行程序会在工具结果和节点日志中提供带类型的错误代码。常见错误如下：

| 代码                                                 | 原因                                                                                                                                                           | 修复方法                                                                                                                                                                                                                                  |
| ---------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `COMPUTER_DRIVER_UNAVAILABLE`                        | `cua-driver` 二进制文件不在 `PATH` 中（或 `driverPath` 错误）、守护进程未能及时就绪，或者节点不是 Windows/Linux。                 | 在 `PATH` 中安装 `cua-driver` 0.10.x，或设置 `driverPath`。在交互式桌面会话中运行 `openclaw node run`；在 Linux 上，确保存在 X11 `DISPLAY`（或带 `CUA_DRIVER_RS_ENABLE_WAYLAND` 的 `WAYLAND_DISPLAY`）。 |
| `COMPUTER_DRIVER_UNSUPPORTED`                        | 已连接的驱动程序不是 `cua-driver` 0.10.x，或者其能力/架构版本不同。                                                                      | 安装受支持的 0.10.x 版本。更正后，插件会在大约 30 秒后重新探测，因此无需重启节点。                                                                                                          |
| `COMPUTER_REFUSED_<code>`                            | 驱动程序拒绝了操作，并返回 `background_unavailable`、`background_occluded` 或 `foreground_unavailable`（KDE/KWin Wayland）等结构化代码。   | 将目标窗口置于前台、切换到 X11，或使用受支持的合成器。请参阅上面的兼容性说明。                                                                                                                    |
| `COMPUTER_STALE_FRAME`                               | 坐标引用的屏幕截图已不再是当前截图（上下文压缩、显示器几何布局变化或参考宽度变化）。                 | 在执行坐标操作前重新调用 `screenshot`。                                                                                                                                                                              |
| `COMPUTER_UNSUPPORTED_ACTION`                        | 此执行程序无法可靠执行的操作：`hold_key`、`left_mouse_down`、`left_mouse_up`、按住修饰键拖动/滚动，或在 Linux 上按住修饰键点击。 | 使用受支持的操作。cua-driver 0.10.x 没有桌面范围的按住输入契约。                                                                                                                                                  |
| `COMPUTER_UNSUPPORTED_DISPLAY`                       | 使用了非主 `screenIndex`、捕获与屏幕几何布局不匹配，或光标位于主显示器之外。                                                       | 仅控制主显示器。                                                                                                                                                                                                      |
| `COMPUTER_UNSUPPORTED_KEY`                           | 驱动程序无法可靠复现的 `key` 值：Shift 状态取决于键盘布局的数字或标点符号键，或未知按键。                        | 改用 `type` 操作发送该文本。                                                                                                                                                                                    |
| `COMPUTER_DRIVER_ERROR` / `COMPUTER_INVALID_REQUEST` | 驱动程序失败但未返回结构化代码，或操作参数格式错误。                                                                            | 检查驱动程序状态并重新截取屏幕截图；更正操作参数。                                                                                                                                                        |

## `computer.act` 节点命令

`computer.act` 是该工具用来路由输入的唯一节点命令（`node.invoke` 搭配 `command: "computer.act"`）。该命令具有以下特性：

- **默认危险**：它被列入内置的危险节点命令，在显式启用前不包含在运行时允许列表中。macOS、Windows 和 Linux 桌面节点仍可在配对时声明该命令，因此只需批准一次此功能面。
- **基于能力**：该工具要求已连接的节点同时声明 `computer.act` 和 `screen.snapshot`。内置的 macOS 应用和选择启用的实验性 `cua-computer` 插件执行相同的命令对。

读取操作复用 `screen.snapshot`；不存在第二条捕获路径。有关共享捕获命令，请参阅[摄像头和屏幕节点](/zh-CN/nodes/camera)。

## 启用并授权

1. 启用平台执行器：在 macOS 上，启用 **Settings → Allow Computer Control**，然后在 **Settings → Permissions** 下授予 **Accessibility** 和 **Screen Recording** 权限；在 Windows/Linux 上，按照上面的实验性 `cua-computer` 设置操作。
2. 在 Gateway 网关上批准配对更新（新命令会强制重新配对）。
3. 将该工具开放给具备视觉能力的智能体。对于默认的 `coding` 配置：

   ```json5
   {
     tools: {
       alsoAllow: ["computer"],
       // 沙箱隔离的智能体还需要通过第二道门控：
       sandbox: { tools: { alsoAllow: ["computer"] } },
     },
   }
   ```

4. 在限定时间窗口内启用 `computer.act`。`phone-control` 插件提供一个 `computer` 组：

   ```text
   /phone arm computer 30m
   /phone status
   /phone disarm
   ```

   启用需要 `operator.admin`（或所有者）权限，并会自动过期。旧版 `/phone arm all` 组特意排除了桌面控制；请使用显式的 `computer` 组。启用仅切换 Gateway 网关可以调用的内容；节点应用仍会强制执行其平台特定设置和操作系统权限，包括 macOS 上的 **Allow Computer Control**、Accessibility 和 Screen Recording。

对于持久授权，请将 `computer.act` 添加到 `gateway.nodes.commands.allow`，**并将其从** `gateway.nodes.commands.deny` **中移除**；拒绝列表优先。持久授权不会自动过期。在 `/phone arm` 之前已经存在的条目会在 `/phone disarm` 之后保留；临时授权处于启用状态时，不要将其转换为持久授权。

授权特意拆分为启用和使用两部分。启用或
持久配置 `computer.act` 需要管理权限。
启用后，具有 `operator.write` 权限的已通过身份验证的操作员可以通过
`node.invoke` 调用 `computer.act`，直到授权过期或被停用；
不会对每个操作单独执行管理员检查。批准声明了
`computer.act` 的节点仅会记录该能力，以便以后启用，
其本身并不会允许调用。

## 安全

- 授权前，每一层（工具策略、Gateway 网关命令策略、节点应用设置和平台权限）都必须同意。对于当前的 macOS 执行器，这包括 **Allow Computer Control**、Accessibility 和 Screen Recording。启用后，在过期或执行 `/phone disarm` 之前，操作无需逐次确认即可执行。
- macOS 执行器每次发送一个字素的文本，因此取消、断开连接、暂停、禁用或替换端点都会使其在发送下一个字素前停止。实验性的 cua-driver 执行器无法在输入过程中取消 `type_text` 调用。
- 屏幕截图仅供模型使用，绝不会自动发送到聊天中（问题 [#44759](https://github.com/openclaw/openclaw/issues/44759)）。
- 将屏幕内容视为不可信内容；其中可能包含提示词注入。

## 与其他桌面控制路径的关系

这是由智能体驱动的路径。请参阅 [Peekaboo bridge](/zh-CN/platforms/mac/peekaboo)，了解它与 PeekabooBridge 主机、Codex Computer Use 以及直接使用的 `cua-driver` MCP 之间的关系。
