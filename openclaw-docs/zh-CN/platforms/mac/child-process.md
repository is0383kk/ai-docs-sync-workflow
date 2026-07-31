---
read_when:
    - 将 Mac 应用与 Gateway 网关生命周期集成
summary: macOS 上的 Gateway 网关生命周期（launchd）
title: macOS 上的 Gateway 网关生命周期
x-i18n:
    generated_at: "2026-07-26T06:50:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89a27334afcecb322feb2732cf6282b4c286ef27828a1b57157f9d4fc161aed6
    source_path: platforms/mac/child-process.md
    workflow: 16
---

macOS 应用默认通过 **launchd** 管理 Gateway 网关，而不会将 Gateway 网关作为子进程启动。它首先尝试连接到已在配置端口上运行的 Gateway 网关；如果无法访问任何 Gateway 网关，则会通过外部 `openclaw` CLI 启用 launchd 服务（无嵌入式运行时）。这样可以确保登录时可靠地自动启动，并在崩溃后重新启动。

目前**未使用**子进程模式（由应用直接启动 Gateway 网关）。如果需要与 UI 更紧密地耦合，请在终端中手动运行 Gateway 网关。

## 默认行为（launchd）

- 应用会安装一个每用户 LaunchAgent，其标签为 `ai.openclaw.gateway`（使用 `--profile`/`OPENCLAW_PROFILE` 时则为
  `ai.openclaw.<profile>`）。
- 启用本地模式后，应用会确保 LaunchAgent 已加载，并在需要时启动 Gateway 网关。
- 日志会写入 launchd Gateway 网关日志路径（可在调试设置中查看）。

常用命令：

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

运行命名配置文件时，请将标签替换为 `ai.openclaw.<profile>`。

## 未签名的开发构建

`scripts/restart-mac.sh --no-sign` 用于在没有签名密钥的情况下快速进行本地构建。为了防止 launchd 指向未签名的中继二进制文件，它会写入
`~/.openclaw/disable-launchagent`。

如果该标记存在，签名运行的 `scripts/restart-mac.sh` 会清除此覆盖设置。若要手动重置：

```bash
rm ~/.openclaw/disable-launchagent
```

## 仅连接模式

若要强制 macOS 应用永不安装或管理 launchd，请使用 `--attach-only`（或 `--no-launchd`）启动应用。这会设置
`~/.openclaw/disable-launchagent`，使应用仅连接到已经运行的 Gateway 网关。也可以在调试设置中切换相同行为。

## 远程模式

远程模式永远不会启动本地 Gateway 网关。应用通过 SSH 隧道连接到远程主机，并通过该隧道建立连接。

## 为什么首选 launchd

- 登录时自动启动。
- 内置重启/KeepAlive 语义。
- 可预测的日志和监管机制。

如果以后再次需要真正的子进程模式，应将其作为单独且明确仅供开发使用的模式进行记录。

## 相关内容

- [macOS 应用](/zh-CN/platforms/macos)
- [Gateway 网关运行手册](/zh-CN/gateway)
