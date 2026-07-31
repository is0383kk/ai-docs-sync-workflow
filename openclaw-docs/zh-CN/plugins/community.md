---
doc-schema-version: 1
read_when:
    - 你想查找第三方 OpenClaw 插件
    - 你想在 ClawHub 上发布或上架自己的插件
summary: 查找并发布由社区维护的 OpenClaw 插件
title: 社区插件
x-i18n:
    generated_at: "2026-07-26T06:49:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

社区插件是第三方软件包，可通过渠道、工具、提供商、Hooks 或其他能力扩展 OpenClaw。使用 [ClawHub](/clawhub) 作为发现公共社区插件的主要平台。

## 查找插件

通过 CLI 搜索 ClawHub：

```bash
openclaw plugins search "calendar"
```

使用明确的来源前缀安装 ClawHub 插件：

```bash
openclaw plugins install clawhub:<package-name>
```

在发布切换期间，npm 仍是受支持的直接安装方式：

```bash
openclaw plugins install npm:<package-name>
```

有关常见的安装、更新、检查和卸载示例，请参阅[管理插件](/zh-CN/plugins/manage-plugins)。有关完整的命令参考和来源选择规则，请参阅 [`openclaw plugins`](/zh-CN/cli/plugins)。

## 发布插件

在 ClawHub 上发布公共社区插件，以便 OpenClaw 用户发现并安装它们。ClawHub 负责实时软件包列表、发布历史、扫描状态和安装提示；文档不维护静态的第三方插件目录。

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

发布前，请确保插件具有软件包元数据、插件清单、设置文档和明确的维护负责人。ClawHub 会在创建版本前验证所有者权限范围、软件包名称、版本、文件限制和源代码元数据，然后在审查和验证完成前，对常规安装和下载入口隐藏新版本。

发布前检查清单：

| 要求                 | 原因                                                |
| -------------------- | --------------------------------------------------- |
| 已发布到 ClawHub     | 用户需要 `openclaw plugins install` 提示才能正常使用 |
| 公开的 GitHub 仓库   | 便于源代码审查、问题跟踪并确保透明度                |
| 设置和使用文档       | 用户需要了解如何配置它                              |
| 积极维护             | 近期有更新或及时处理问题                            |

完整的发布约定：

- [ClawHub 发布](/zh-CN/clawhub/publishing) - 所有者、权限范围、版本、审查、软件包验证和软件包转移
- [构建插件](/zh-CN/plugins/building-plugins) - 插件软件包结构和首次发布工作流
- [插件清单](/zh-CN/plugins/manifest) - 原生插件清单字段

## 相关内容

- [插件](/zh-CN/tools/plugin) - 安装、配置、重启和故障排查
- [管理插件](/zh-CN/plugins/manage-plugins) - 命令示例
- [ClawHub 发布](/zh-CN/clawhub/publishing) - 发布和版本规则
