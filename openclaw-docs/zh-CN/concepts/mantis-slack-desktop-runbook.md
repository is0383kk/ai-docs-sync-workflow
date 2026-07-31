---
read_when:
    - 从 GitHub 或本地运行 Mantis Slack 桌面端 QA
    - 调试缓慢的 Mantis Slack 桌面端运行任务
    - 选择源码、预水合或暖租约模式
    - 向 PR 发布截图和视频证据
summary: Mantis Slack 桌面端 QA 操作员运行手册：GitHub 调度、本地 CLI、预热 VNC 租约、hydrate 模式、耗时解读、工件和故障处理。
title: Mantis Slack 桌面端运行手册
x-i18n:
    generated_at: "2026-07-26T06:42:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack 桌面 QA 是针对 Slack 类缺陷的真实 UI 验证通道，适用于需要
Linux 桌面、VNC 救援、Slack Web、真实 OpenClaw Gateway 网关、屏幕截图、
视频和 PR 证据评论的场景。当单元测试或无头
Slack 实时验证通道无法证明缺陷时，请使用它。

## 存储模型

Mantis 使用三个存储层：

- **提供商镜像** - 由 Crabbox 所有，存储在云提供商账户中。
  包含机器能力（Chrome/Chromium、ffmpeg、scrot、
  Node/corepack/pnpm、原生构建工具）和空缓存目录。
- **热租约状态** - 由当前操作员会话所有。在租约有效期间，可以保存
  已登录的浏览器配置文件、`/var/cache/crabbox/pnpm` 和准备好的源代码
  检出。
- **Mantis 工件** - 由 OpenClaw 运行所有。位于
  `.artifacts/qa-e2e/mantis/...` 下；GitHub Actions 会上传它们，Mantis
  GitHub App 会在 PR 上评论内联证据。

绝不要将密钥、浏览器 Cookie、Slack 登录状态、仓库检出、
`node_modules` 或 `dist/` 烘焙进提供商镜像。

## GitHub 调度

从 `main` 运行工作流：

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

`candidate_ref` 受到限制，因为工作流使用实时凭据：它
必须解析为当前 `main` 的祖先版本、发布标签，或
`openclaw/openclaw` 中开放 PR 的头部版本。

工作流会生成：

- 已上传的工件 `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- 来自 Mantis GitHub App 的内联 PR 评论
- `slack-desktop-smoke.png`、`slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`、`slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`、`mantis-slack-desktop-smoke-report.md`
- 远程日志：`slack-desktop-command.log`、`openclaw-gateway.log`、`chrome.log`、`ffmpeg.log`

PR 评论通过隐藏的 `<!-- mantis-slack-desktop-smoke -->` 标记原地更新。

## 本地 CLI

冷源代码验证：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

保留 VM 以便通过 VNC 救援：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

打开 VNC：

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

复用热租约：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

仅当复用的远程工作区已经具有
`node_modules` 和已构建的 `dist/` 时，才使用 `--hydrate-mode prehydrated`；否则 Mantis 会以关闭方式失败。

验证原生 Slack 审批 UI：

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` 与 `--gateway-setup` 互斥。除非传入显式的审批检查点
`--scenario`，否则它会运行选择启用的 `slack-approval-exec-native` 和
`slack-approval-plugin-native` 场景；其他 Slack 场景会在 VM 启动前被拒绝。Slack QA 运行器会根据
它观察到的真实 Slack API 消息写入各检查点 JSON 文件，然后
远程监视器会将该消息渲染到
`approval-checkpoints/<scenario>-pending.png` 和
`approval-checkpoints/<scenario>-resolved.png`。如果任何
检查点 JSON、消息证据、确认 JSON 或渲染后的屏幕截图缺失
或为空，运行就会失败。

冷 GitHub Actions 租约没有 Slack Web Cookie，因此其浏览器捕获
可能会停留在 Slack 登录屏幕。对于审批检查点验证，应信任
渲染后的检查点图像和 Slack QA 工件，而不是
`slack-desktop-smoke.png`。仅当浏览器屏幕截图本身必须显示
Slack Web 时，才使用带有手动登录 Slack Web 配置文件的保留热租约。

## 填充模式

| 模式          | 使用场景                                  | 远程行为                                                                       | 权衡                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | 常规 PR 验证、冷机器、CI        | 在 VM 内运行 `pnpm install --frozen-lockfile --prefer-offline` 和 `pnpm build` | 最慢，但源代码检出验证最强                 |
| `prehydrated` | 有意准备了复用租约 | 要求已有 `node_modules` 和 `dist/`；跳过安装/构建                     | 速度快，但仅对操作员控制的热租约有效 |

GitHub Actions 始终会在 VM 运行前准备候选检出。
pnpm 存储按操作系统、Node 版本和锁文件进行缓存。VM 的 `source` 运行
也会在 `/var/cache/crabbox/pnpm` 存在时复用它。

## 耗时解读

`mantis-slack-desktop-smoke-report.md` 包含各阶段耗时：

- `crabbox.warmup` - 云提供商启动、桌面/浏览器就绪、SSH。
- `crabbox.inspect` - 租约元数据查询。
- `credentials.prepare` - 获取 Convex 凭据租约。
- `crabbox.remote_run` - 同步、启动浏览器、安装/构建 OpenClaw 或
  验证填充状态、启动 Gateway 网关、捕获屏幕截图和视频。
- `artifacts.copy` - 从 VM 反向 rsync。

当 Crabbox 返回非零远程状态，但 Mantis 复制的元数据证明 OpenClaw Gateway 网关
设置已完成或 Slack QA 命令本身成功退出时，`crabbox.remote_run` 可能显示
`accepted`。应将 `accepted` 视为带说明的通过，而不是场景失败。

如果运行缓慢：

- 预热耗时占主导：预烘焙或提升更好的 Crabbox 提供商镜像。
- `source` 中的 `remote_run` 耗时占主导：使用热租约、改进 pnpm 存储
  复用，或将机器先决条件移入提供商镜像。
- `prehydrated` 中的 `remote_run` 耗时占主导：远程工作区实际上尚未
  就绪，或者 Gateway 网关/浏览器/Slack 设置较慢。
- 工件复制耗时占主导：检查视频大小和工件目录内容。

## 证据清单

良好的 PR 评论会显示：

- 场景 ID 和候选 SHA
- GitHub Actions 运行 URL 和工件 URL
- 内联审批检查点屏幕截图，或来自已登录热租约的 Slack Web
  屏幕截图
- 可用时提供内联动画预览
- 完整 MP4 和裁剪后 MP4 的链接
- 通过/失败状态和报告中的耗时摘要

不要将屏幕截图或视频提交到仓库。将它们保留在 GitHub
Actions 工件或 PR 评论中。

## 故障处理

如果工作流在 VM 运行前失败，请先检查 Actions 作业。
常见原因：不受信任的 `candidate_ref`、缺少环境密钥，或
候选版本安装/构建失败。

如果 VM 运行失败，但屏幕截图已复制回来，请检查：

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

如果运行保留了租约，请使用报告中的 `crabbox vnc ...`
命令打开 VNC，然后在完成后停止租约：

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

如果 Slack 登录已过期，请在保留租约的 VNC 中修复登录，然后使用
`--lease-id` 重新运行。不要将该浏览器配置文件烘焙进提供商镜像。

## 相关内容

- [QA overview](/zh-CN/concepts/qa-e2e-automation)
- [Slack 渠道](/zh-CN/channels/slack)
- [测试](/zh-CN/help/testing)
