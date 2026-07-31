---
read_when:
    - 在节点平台上添加或修改摄像头拍摄功能
    - 扩展智能体可访问的 MEDIA 临时文件工作流
summary: 在 iOS、Android、macOS 和 Linux 节点上使用摄像头拍摄照片和短视频片段
title: 相机拍摄
x-i18n:
    generated_at: "2026-07-26T06:13:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw 支持在已配对的 **iOS**、**Android**、**macOS** 和 **Linux** 节点上为智能体工作流使用摄像头采集功能：通过 Gateway 网关 `node.invoke` 拍摄照片（`jpg`）或短视频片段（`mp4`，可选录制音频）。

所有摄像头访问均受各平台上由用户控制的设置约束。

## iOS 节点

### iOS 用户设置

- iOS Settings 标签页 → **Camera** → **Allow Camera**（`camera.enabled`）。
  - 默认值：**on**（缺少该键时视为已启用）。
  - 关闭时：`camera.*` 命令返回 `CAMERA_DISABLED`。

### iOS 命令（通过 Gateway 网关 `node.invoke`）

- `camera.list`
  - 响应载荷：`devices` — `{ id, name, position, deviceType }` 数组。

- `camera.snap`
  - 参数：
    - `facing`：`front|back`（默认值：`front`）
    - `maxWidth`：数字（可选；默认值为 `1600`）
    - `quality`：`0..1`（可选；默认值为 `0.9`，限制在 `[0.05, 1.0]` 范围内）
    - `format`：当前为 `jpg`
    - `delayMs`：数字（可选；默认值为 `0`，内部上限为 `10000`）
    - `deviceId`：字符串（可选；来自 `camera.list`）
  - 响应载荷：`format: "jpg"`、`base64`、`width`、`height`。
  - 载荷限制：照片会重新压缩，以使 base64 编码后的载荷保持在 5MB 以下。

- `camera.clip`
  - 参数：
    - `facing`：`front|back`（默认值：`front`）
    - `durationMs`：数字（默认值为 `3000`，限制在 `[250, 60000]` 范围内）
    - `includeAudio`：布尔值（默认值为 `true`）
    - `format`：当前为 `mp4`
    - `deviceId`：字符串（可选；来自 `camera.list`）
  - 响应载荷：`format: "mp4"`、`base64`、`durationMs`、`hasAudio`。

### iOS 前台要求

与 `canvas.*` 类似，iOS 节点仅允许在**前台**执行 `camera.*` 命令。后台调用返回 `NODE_BACKGROUND_UNAVAILABLE`。

### CLI 辅助工具

获取媒体文件最简单的方式是使用 CLI 辅助工具，它会将解码后的媒体写入临时文件并输出保存路径。

```bash
openclaw nodes camera snap --node <id>                 # 默认值：同时使用前置 + 后置摄像头（2 行 MEDIA）
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` 默认值为 `--facing both`，会同时使用前置和后置摄像头进行采集，为智能体提供两个视角；如需使用单个明确的朝向，请传递 `--device-id`（设置 `--device-id` 时会拒绝 `both`）。除非自行构建封装程序，否则输出文件均为临时文件（位于操作系统临时目录中）。

## Android 节点

### Android 用户设置

- Android Settings 面板 → **Camera** → **Allow Camera**（`camera.enabled`）。
  - **全新安装默认关闭。** 在引入此设置之前完成的现有安装会迁移为**开启**，以避免升级后原本正常工作的摄像头访问功能在没有提示的情况下失效。
  - 关闭时：`camera.*` 命令返回 `CAMERA_DISABLED: enable Camera in Settings`。

### 权限

- `CAMERA` 是 `camera.snap` 和 `camera.clip` 所必需的；缺少权限或权限被拒绝时返回 `CAMERA_PERMISSION_REQUIRED`。
- 当 `includeAudio` 为 `true` 时，`camera.clip` 需要 `RECORD_AUDIO`；缺少权限或权限被拒绝时返回 `MIC_PERMISSION_REQUIRED`。

应用会尽可能提示用户授予运行时权限。

### Android 前台要求

与 `canvas.*` 类似，Android 节点仅允许在**前台**执行 `camera.*` 命令。后台调用返回 `NODE_BACKGROUND_UNAVAILABLE: command requires foreground`。

### Android 命令（通过 Gateway 网关 `node.invoke`）

- `camera.list`
  - 响应载荷：`devices` — `{ id, name, position, deviceType }` 数组。

- `camera.snap`
  - 参数：`facing`（`front|back`，默认值为 `front`）、`quality`（默认值为 `0.95`，限制在 `[0.1, 1.0]` 范围内）、`maxWidth`（默认值为 `1600`）、`deviceId`（可选；未知 ID 会返回 `INVALID_REQUEST` 失败）。
  - 响应载荷：`format: "jpg"`、`base64`、`width`、`height`。
  - 载荷限制：重新压缩以使 base64 保持在 5MB 以下（与 iOS 的限额相同）。

- `camera.clip`
  - 参数：`facing`（默认值为 `front`）、`durationMs`（默认值为 `3000`，限制在 `[200, 60000]` 范围内）、`includeAudio`（默认值为 `true`）、`deviceId`（可选）。
  - 响应载荷：`format: "mp4"`、`base64`、`durationMs`、`hasAudio`。
  - 载荷限制：base64 编码前的原始 MP4 大小上限为 18MB；超出大小限制的片段会返回 `PAYLOAD_TOO_LARGE` 失败（减小 `durationMs` 后重试）。

## macOS 应用

### macOS 用户设置

macOS 配套应用提供了一个复选框：

- **Settings → General → Allow Camera**（`openclaw.cameraEnabled`）。
  - 默认值：**off**。
  - 关闭时：摄像头请求返回 `CAMERA_DISABLED: enable Camera in Settings`。

### CLI 辅助工具（节点调用）

使用主 `openclaw` CLI 在 macOS 节点上调用摄像头命令。

```bash
openclaw nodes camera list --node <id>                     # 列出摄像头 ID
openclaw nodes camera snap --node <id>                     # 输出保存路径
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # 输出保存路径
openclaw nodes camera clip --node <id> --duration-ms 3000   # 输出保存路径（旧版标志）
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- 除非被覆盖，否则 `openclaw nodes camera snap` 的默认值为 `maxWidth=1600`。
- `camera.snap` 会在预热/曝光稳定后等待 `delayMs`（默认值为 2000ms，限制在 `[0, 10000]` 范围内），然后再进行采集。
- 照片载荷会重新压缩，以使 base64 保持在 5MB 以下。

## Linux 节点主机

内置的 Linux Node 插件为 CLI `openclaw node` 服务添加摄像头采集功能。它可在无头主机上运行，不需要 Linux 桌面应用。

摄像头访问默认关闭。请在插件条目下启用它，然后重启节点服务，以重新构建其 Gateway 网关通告：

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

要求：

- 支持 V4L2 输入、`libx264` 和 AAC 的 FFmpeg
- 节点服务用户可读取的 `/dev/video*` 设备；在常见发行版上，请将该用户添加到 `video` 组
- 对于默认使用 `includeAudio: true` 的视频片段，需要具备默认输入源的可用 PulseAudio 服务器或 PipeWire PulseAudio 兼容层

Linux 会从 `camera.list` 返回具备采集能力且可读取的 V4L2 设备路径；FFmpeg 会探测每个 `/dev/video*` 候选项，并忽略元数据节点或仅输出节点。设备 `position` 为 `unknown`，因此未提供 `deviceId` 的朝向请求会生成一张或一段位置为 `unknown` 的照片或视频片段，而不会声称使用的是前置或后置摄像头。当主机有多个摄像头时，请使用 `deviceId`。`camera.snap` 使用 FFmpeg 输入预热 `delayMs`，并在限制宽度的同时保持宽高比。`camera.clip` 会将麦克风音频录制为 MP4 音轨；OpenClaw 特意不提供独立的麦克风命令。

该插件使用 `libx264` 编码 MP4 视频，不会静默更改编解码器。缺少所需输入或编码器的 FFmpeg 构建会返回 `CAMERA_UNAVAILABLE`。如果照片或视频片段会超过 25MB 的 base64 载荷限额，则会返回 `PAYLOAD_TOO_LARGE` 失败。

`camera.snap` 和 `camera.clip` 仍属于危险命令。仅在确实要启用采集时，才将它们添加到 `gateway.nodes.commands.allow`；仅启用插件并不会绕过 Gateway 网关策略。

## 安全性 + 实际限制

- 摄像头和麦克风访问会触发操作系统通常的权限提示（并且需要在 `Info.plist` 中提供用途说明字符串）。
- 视频片段的时长上限为 60s，以避免节点载荷过大（base64 开销加上消息大小限制）。

## macOS 屏幕视频（操作系统级）

对于_屏幕_视频（而非摄像头视频），请使用 macOS 配套应用：

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # 输出保存路径
```

需要 macOS **Screen Recording** 权限（TCC）。

## 相关内容

- [图像和媒体支持](/zh-CN/nodes/images)
- [媒体理解](/zh-CN/nodes/media-understanding)
- [位置命令](/zh-CN/nodes/location-command)
