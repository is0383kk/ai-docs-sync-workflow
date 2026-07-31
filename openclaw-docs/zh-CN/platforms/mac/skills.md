---
read_when:
    - 更新 macOS Skills 设置界面
    - 更改 Skills 门控或安装行为
summary: macOS Skills 设置界面和由 Gateway 网关支持的状态
title: Skills（macOS）
x-i18n:
    generated_at: "2026-07-26T06:14:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

macOS 应用通过 Gateway 网关呈现 OpenClaw Skills；它不会在本地解析 Skills。

## 数据源

- `skills.status`（Gateway 网关）返回所有 Skills，以及资格状态和缺失的要求，包括内置 Skills 的允许列表拦截项。
- 要求来自每个 `SKILL.md` 中的 `metadata.openclaw.requires`。

## 安装操作

- `metadata.openclaw.install` 定义安装选项（brew/node/go/uv/download）。
- 应用调用 `skills.install`，在 Gateway 网关主机上运行安装程序。
- 由操作员管理的 `security.installPolicy`（`enabled`、`targets`、`exec`）可在运行安装程序元数据之前，阻止由 Gateway 网关支持的 Skills 安装。内置的危险代码扫描（用于插件安装）尚未接入 Skills 安装流程。
- 如果所有安装选项均为 `download`，Gateway 网关会呈现所有下载选项。
- 否则，Gateway 网关会根据当前安装偏好（`skills.install.preferBrew`、`skills.install.nodeManager`）和主机上的二进制文件选择一个首选安装程序：当启用 `preferBrew` 且存在 `brew` 时，首先选择 Homebrew；然后选择 `uv`；再选择已配置的 node 管理器；如果 Homebrew 可用，则再次选择它（即使没有 `preferBrew`）；然后选择 `go`；最后选择 `download`。
- Node 安装标签会反映已配置的 node 管理器，包括 `yarn`。

## 环境变量/API 密钥

- 应用将密钥存储在 `~/.openclaw/openclaw.json` 的 `skills.entries.<skillKey>` 下。
- `skills.update` 会修补 `enabled`、`apiKey` 和 `env`。

## 远程模式

- 安装和配置更新在 Gateway 网关主机上进行，而不是在本地 Mac 上进行。

## 相关内容

- [Skills](/zh-CN/tools/skills)
- [macOS 应用](/zh-CN/platforms/macos)
