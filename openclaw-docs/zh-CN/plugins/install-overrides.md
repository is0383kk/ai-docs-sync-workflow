---
read_when:
    - 使用本地打包的插件测试新手引导或设置流程
    - 发布前验证插件包
    - 使用测试工件替代自动安装的插件
sidebarTitle: Install overrides
summary: 测试打包插件在设置时安装流程中的覆盖行为
title: 插件安装覆盖设置
x-i18n:
    generated_at: "2026-07-26T06:20:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: adc823f49ea9f8fa86e6a89933e43fdc309d808ac24397770495dbe81cb4b0d7
    source_path: plugins/install-overrides.md
    workflow: 16
---

插件安装覆盖允许维护者将设置期间的插件安装指向特定的 npm 软件包或本地 npm-pack tarball，而不是目录、内置或默认 npm 源。它们仅用于 E2E 和软件包验证；普通用户使用
[`openclaw plugins install`](/zh-CN/cli/plugins)
安装插件。

<Warning>
覆盖会执行你所提供来源中的插件代码。只能在隔离的状态目录或一次性测试机器中使用它们。
</Warning>

## 环境

除非同时设置以下两个变量，否则覆盖功能将被禁用：

```bash
export OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1
export OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{
  "codex": "npm-pack:/tmp/openclaw-codex-2026.5.8.tgz",
  "openclaw-web-search": "npm:@openclaw/web-search@2026.5.8"
}'
```

覆盖映射采用 JSON 格式，并以插件 ID 为键。值支持：

| 前缀                  | 来源                                                                                                  |
| --------------------- | ----------------------------------------------------------------------------------------------------- |
| `npm:<registry-spec>` | 注册表软件包、精确版本或标签                                                                          |
| `npm-pack:<path.tgz>` | 由 `npm pack` 生成的本地 tarball；相对路径从当前工作目录解析 |

## 行为

当设置期间的流程安装 ID 出现在映射中的插件时，OpenClaw 会使用覆盖源，而不是目录、内置或默认 npm 源。这适用于新手引导以及使用共享设置期间插件安装器的任何其他流程。

- 覆盖仍会强制执行预期的插件 ID：映射到 `codex` 的 tarball
  必须安装清单 ID 为 `codex` 的插件。
- 覆盖不会继承官方可信来源状态。即使目录条目通常代表 OpenClaw 所有的软件包，覆盖也会被视为操作员提供的测试输入。
- 工作区 `.env` 文件无法启用安装覆盖；这两个环境变量均位于工作区 dotenv 阻止列表中。请在启动 OpenClaw 的可信 shell、CI 作业或远程测试命令中设置它们。

## 软件包 E2E

使用隔离的状态目录，避免软件包安装和安装记录影响正常的 OpenClaw 状态：

```bash
npm pack extensions/codex --pack-destination /tmp

OPENCLAW_STATE_DIR="$(mktemp -d)" \
OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1 \
OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{"codex":"npm-pack:/tmp/openclaw-codex-2026.5.8.tgz"}' \
pnpm openclaw onboard --mode local
```

验证状态目录下安装的软件包：

```bash
find "$OPENCLAW_STATE_DIR/npm/projects" -path '*/node_modules/@openclaw/codex/package.json' -print
grep -R '"@openclaw/codex"' "$OPENCLAW_STATE_DIR/npm/projects"/*/package-lock.json
```

对于实时提供商 E2E，请在启动测试命令前，从可信 shell 或 CI 密钥中载入真实 API 密钥。不要打印密钥；仅报告来源以及密钥是否存在。
