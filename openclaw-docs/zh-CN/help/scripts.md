---
read_when:
    - 从仓库运行脚本
    - 在 `./scripts` 下添加或更改脚本
summary: 仓库脚本：用途、范围和安全注意事项
title: 脚本
x-i18n:
    generated_at: "2026-07-26T06:12:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` 包含用于本地工作流和运维任务的辅助脚本。当任务明确与某个脚本相关时，请使用这些脚本；否则优先使用 CLI。

## 约定

- 除非文档或发布检查清单中引用了脚本，否则脚本均为**可选**。
- 如果存在 CLI 接口，请优先使用（示例：`openclaw models status --check`）。
- 假定脚本与特定主机相关；在新机器上运行前，请先阅读脚本。

## 身份验证监控脚本

常规模型身份验证详见[身份验证](/zh-CN/gateway/authentication)。以下脚本是一个独立的可选系统，用于在远程/无头主机上监控 **Claude Code CLI 订阅令牌**，并通过手机重新进行身份验证：

- `scripts/setup-auth-system.sh` - 一次性设置：检查当前身份验证状态，帮助生成长期有效的 `claude setup-token`，并输出 systemd/Termux 安装步骤。
- `scripts/claude-auth-status.sh [full|json|simple]` - 检查 Claude Code 和 OpenClaw 的身份验证状态。
- `scripts/auth-monitor.sh` - 轮询状态，并在令牌即将过期时发送通知（通过 OpenClaw send 和/或 ntfy.sh）。环境变量：`WARN_HOURS`（默认值为 `2`）、`NOTIFY_PHONE`、`NOTIFY_NTFY`。通过内置的 `scripts/systemd/openclaw-auth-monitor.{service,timer}` 定时运行（每 30 分钟一次）。
- `scripts/mobile-reauth.sh` - 重新运行 `claude setup-token`，并输出可在手机上打开的 URL，供通过 Termux 使用 SSH 时使用。
- `scripts/termux-quick-auth.sh`、`scripts/termux-auth-widget.sh`、`scripts/termux-sync-widget.sh` - Termux:Widget 脚本：通过 SSH 连接主机、显示状态 Toast，并在身份验证过期时打开重新验证控制台/说明。

## GitHub 读取辅助工具

如果希望 `gh` 使用 GitHub App 安装令牌执行限定仓库范围的读取调用，同时让常规 `gh` 保持使用你的个人登录以执行写入操作，请使用 `scripts/gh-read`。

必需的环境变量：

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

可选的环境变量：

- `OPENCLAW_GH_READ_INSTALLATION_ID`：希望跳过基于仓库的安装查找时使用
- `OPENCLAW_GH_READ_PERMISSIONS`：以逗号分隔，用于覆盖要请求的读取权限子集

仓库解析顺序：

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

示例：

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## 添加脚本时

- 保持脚本功能聚焦，并提供相应文档。
- 在相关文档中添加简短条目（如果文档不存在，则创建一个）。

## 相关内容

- [测试](/zh-CN/help/testing)
- [实时测试](/zh-CN/help/testing-live)
