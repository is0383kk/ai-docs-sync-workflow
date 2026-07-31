---
read_when:
    - 查找 Linux 配套应用状态
    - 在 Linux 节点主机上启用摄像头、位置或通知
    - 规划平台覆盖范围或贡献
    - 调试 VPS 或容器中的 Linux OOM 终止或退出码 137
summary: Linux 支持 + 配套应用状态
title: Linux 应用
x-i18n:
    generated_at: "2026-07-26T06:22:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

Gateway 网关在 Linux 上受到完整支持，并且需要 Node。Bun 仍可用作依赖安装程序或软件包脚本运行器，但无法运行 OpenClaw，因为它不提供 `node:sqlite`。

## 桌面配套应用

OpenClaw Linux 配套应用是用于本地 Gateway 网关的 Tauri 桌面应用。它：

- 在 OpenClaw CLI 和托管 Node 运行时缺失时进行安装；发布版本会自动安装稳定频道，而开发版本会先询问要使用的频道
- 在尝试更改服务之前连接到健康的 Gateway 网关
- 将安装、启动、停止和重启操作委托给由 CLI 管理的 systemd 用户服务
- 发现附近的 Bonjour Gateway 网关，并在按路由限定的窗口中打开各自的 Control UI，以便多个
  Gateway 网关仪表板可以保持连接并同时使用
- 使用解析后的身份验证 URL 打开由 Gateway 网关提供的 Control UI
- 首次运行安装后，以新手引导模式打开 Control UI，其中
  可以选择将检测到的 Claude Code、Codex 或 Hermes 记忆导入
  Agent 工作区（之后仍可在
  Settings → Import Memory 下使用同一导入功能）
- 为位于同一主机上的 CLI 节点主机渲染由智能体驱动的 Canvas 和内置 A2UI 内容
- 窗口关闭后仍可从系统托盘访问

从 `main` 构建的稳定版本会在对应标签的
[GitHub 发布版本](https://github.com/openclaw/openclaw/releases)中将 `.deb` 和 AppImage 软件包作为资产提供，
名称分别为 `OpenClaw-<version>-amd64.deb` 和 `OpenClaw-<version>-amd64.AppImage`，
旁边还附有 `SHA256SUMS.linux-app.txt` 校验和文件。下载
`.deb` 并使用 `sudo apt install ./OpenClaw-<version>-amd64.deb` 安装，
或者将 AppImage 标记为可执行文件并直接运行。AppImage 运行时
需要 FUSE 2（`sudo apt install libfuse2`，Ubuntu 24.04+ 上则为 `libfuse2t64`）；
如果没有它，请使用 `APPIMAGE_EXTRACT_AND_RUN=1` 运行 AppImage。

也可以从源代码检出构建相同的软件包：

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

`Linux App` CI 工作流会为涉及该应用的拉取请求和手动运行
上传相同的软件包，作为 `openclaw-linux-companion` 工件。
有关 Linux 构建依赖项和开发命令，请参阅仓库中的 `apps/linux/README.md`。

### 快速聊天

使用 `Ctrl+Shift+Space` 或托盘中的 **Quick Chat** 项打开快速聊天。智能体
标记会显示配置的头像、表情符号或字母组合；选择它可切换智能体。
消息使用所选智能体的主会话，并遵循全局会话作用域。
原生 Rust 客户端持有持久的 Ed25519 设备身份。它仅使用
CLI 移交的共享令牌或密码来引导配对，然后存储 Gateway 网关签发的设备令牌，
并在后续连接时优先使用该令牌。身份和
设备令牌位于应用配置目录中权限模式为 `0600` 的文件内；快速
聊天的 WebView 既不会收到凭据，也不会获得 WebSocket。

原生连接不可用时，快速聊天会显示 **Gateway
unreachable — retrying**，并在重新连接前禁用发送。已进入配对阶段的远程设备
则会显示 **Approve this device in the dashboard
(Nodes)**；如果 Gateway 网关提供了短设备 ID，也会一并显示。
如果 Gateway 网关要求提供缺失的共享凭据，则会显示 **Gateway requires a
credential — open the dashboard on the gateway host**；在此状态下，没有配对请求
等待审批。当服务器提供的补救指导更具体时，它会取代这些备用通知。
对于 TLS Gateway 网关，CLI 会将 Gateway 网关证书的 SHA-256
指纹移交给应用；原生客户端会固定该证书，并使用 **Gateway TLS
trust failed — check the certificate fingerprint** 单独报告信任失败，而不是将其视为停机。
通过 SecretRef 配置共享密钥的 Gateway 网关会在
CLI 移交内容中省略该密钥。现有已配对安装可继续通过存储的设备
令牌工作，但在共享密钥身份验证下，全新安装如果没有该引导凭据，
便无法创建待处理的配对请求。
设置代码和 `bootstrapToken` 兑换需要专用的产品 UI，仍属于
后续工作；快速聊天不会尝试其中任何一种流程。

在 X11 上，使用快速聊天中的齿轮图标记录或重置自定义快捷键。
托盘中的 **Quick Chat shortcut** 开关可启用或禁用该快捷键，而不会禁用
普通的 **Quick Chat** 托盘项。Wayland 不支持全局快捷键，因此
快捷键设置会隐藏，托盘项仍是入口。
发送被接受后，快速聊天会保持打开，并在编辑器下方流式显示所选智能体的
纯文本回复。按 `Esc` 可关闭栏及其回复；
`Ctrl+Enter` 仍会打开仪表板。

### Canvas

Linux Canvas 使用两个协同进程。`openclaw node run` 仍是唯一的 Gateway 网关节点连接；内置 `linux-canvas` 插件通过仅限用户访问的 Unix 套接字，将 `canvas.*` 调用转发给正在运行的桌面应用。该应用持有一个按需创建的 WebView 窗口，其中包括内置 A2UI 渲染器以及返回智能体的操作桥。

该插件默认启用。仅当桌面套接字存在于 `$XDG_RUNTIME_DIR/openclaw-canvas.sock`，或在 `XDG_RUNTIME_DIR` 不可用时存在于 `/tmp/openclaw-canvas-$UID.sock`，它才会公布 Canvas。使用 `plugins.entries.linux-canvas.enabled: false` 禁用它。在没有桌面应用的无头 Linux 服务器上，不会公布 Canvas。

Linux v1 使用一个 Canvas 窗口。HTTP 和 HTTPS 页面可以渲染，但仅接受来自内置渲染器的 A2UI 操作。

## CLI 和 SSH 替代方案

对于无头服务器、VPS 或远程 Gateway 网关，CLI 仍是最简单的选择：

1. 安装 Node 24.15+（推荐）、Node 22.22.3+（LTS）或 Node 25.9+。
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. 在你的笔记本电脑上：`ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. 打开 `http://127.0.0.1:18789/`，并使用配置的共享
   密钥进行身份验证（默认为令牌；如果 `gateway.auth.mode` 为 `"password"`，则使用密码）。

完整服务器指南：[Linux 服务器](/zh-CN/vps)。分步 VPS 示例：
[exe.dev](/zh-CN/install/exe-dev)。

## 节点能力

内置 Linux 节点插件无需桌面应用，即可为 CLI `openclaw node` 服务提供设备能力。仅当相应能力已启用且所需本地工具存在时，命令才会公布给 Gateway 网关。

| 能力                              | 默认值 | 要求                                                           |
| --------------------------------------- | ------- | --------------------------------------------------------------------- |
| 桌面通知（`system.notify`） | 开启      | libnotify 提供的 `notify-send` 和桌面通知会话       |
| 相机照片和短片（`camera.*`）    | 关闭     | FFmpeg、V4L2 相机访问权限，以及用于短片音频的 PulseAudio 或 PipeWire |
| 位置（`location.get`）               | 关闭     | GeoClue2 及其 `where-am-i` 演示程序                                    |

在 `openclaw.json` 中配置插件：

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

更改这些设置后，请重启节点服务。可用性在每个进程中只确定一次，节点公布信息会在重启时重新构建。

Gateway 网关对节点命令和能力范围的审批独立于设备配对。首次启动或启用更多能力后，请审批待处理的范围：

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

节点可以保持连接并完成设备配对，但在完成此审批前，其有效 `caps` 和 `commands` 仍可能为空。

相机设备必须可由服务用户读取，通常通过 `video` 组授予权限。当 `includeAudio` 为 true 时，相机短片使用默认的 PulseAudio 或 PipeWire 音源；麦克风音频仅作为该短片的音轨存在，不作为独立命令提供。位置功能要求主机的 GeoClue 策略允许节点服务用户访问。

`camera.snap` 和 `camera.clip` 还需要通过 `gateway.nodes.commands.allow` 在 Gateway 网关中明确启用。有关载荷、限制和错误，请参阅[相机捕获](/zh-CN/nodes/camera)和[位置命令](/zh-CN/nodes/location-command)。

## 安装

- [入门指南](/zh-CN/start/getting-started)
- [安装和更新](/zh-CN/install/updating)
- 可选：[Bun 软件包工作流](/zh-CN/install/bun)、[Nix](/zh-CN/install/nix)、[Docker](/zh-CN/install/docker)

## Gateway 网关服务（systemd）

使用以下任一命令安装：

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # 出现提示时选择 "Gateway service"
```

修复或迁移现有安装：

```bash
openclaw doctor
```

`openclaw gateway install` 默认生成 systemd **用户**单元。完整的
服务指南（包括适用于共享或始终在线主机的**系统**级单元变体）位于
[Gateway 网关运行手册](/zh-CN/gateway#supervision-and-service-lifecycle)。

仅在自定义设置中手动编写单元。最小用户单元示例
（`~/.config/systemd/user/openclaw-gateway[-<profile>].service`）：

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

手动编写的单元不会继承 `openclaw gateway install` 为托管 Gateway 网关服务写入的自适应堆大小设置。请优先使用托管安装程序；如果使用自定义监督程序，请在考虑原生内存余量后设置明确的堆限制。

启用它：

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## 内存压力和 OOM 终止

在 Linux 上，当主机、虚拟机或容器 cgroup
耗尽内存时，内核会选择一个 OOM 牺牲进程。Gateway 网关并不适合作为牺牲进程，因为它持有长期存在的
会话和渠道连接，因此 OpenClaw 会尽可能提高临时子
进程被优先终止的概率。

对于符合条件的 Linux 子进程生成操作，OpenClaw 会用一个简短的
`/bin/sh` 垫片包装命令，将子进程自身的 `oom_score_adj` 提高到 `1000`，然后
对实际命令执行 `exec`。此操作不需要特权：进程始终可以提高
自身的 OOM 分数。

涵盖的子进程范围：

- 由监督程序管理的命令子进程
- PTY shell 子进程
- MCP stdio 服务器子进程
- 由 OpenClaw 启动的浏览器/Chrome 进程（通过插件 SDK 进程运行时）

该包装器仅适用于 Linux；当 `/bin/sh` 不可用，或者
子进程环境将 `OPENCLAW_CHILD_OOM_SCORE_ADJ` 设置为 `0`、`false`、`no` 或
`off` 时，会跳过包装。

验证子进程：

```bash
cat /proc/<child-pid>/oom_score_adj
```

涵盖范围内的子进程预期值为 `1000`；Gateway 网关进程自身
保持正常分数（通常为 `0`）。

systemd 单元的 `OOMPolicy=continue` 可在 OOM 终止程序选中
临时子进程时保持 Gateway 网关服务运行，而不是将整个
单元标记为失败并重启所有渠道；失败的子进程/会话会报告其
自身错误。

这不能替代常规内存调优。如果 VPS 或容器反复
终止子进程，请提高内存限制、降低并发度，或添加更严格的
资源控制（systemd `MemoryMax=`、容器内存限制）。

## 相关内容

- [安装概览](/zh-CN/install)
- [Linux 服务器](/zh-CN/vps)
- [Raspberry Pi](/zh-CN/install/raspberry-pi)
- [Gateway 运行手册](/zh-CN/gateway)
- [Gateway 配置](/zh-CN/gateway/configuration)
