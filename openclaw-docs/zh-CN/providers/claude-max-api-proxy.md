---
read_when:
    - 你想将 Claude Max 订阅与 OpenAI 兼容工具搭配使用
    - 你需要一个封装 Claude Code CLI 的本地 API 服务器
    - 你想评估基于订阅与基于 API key 的 Anthropic 访问方式
summary: 社区代理，用于将 Claude 订阅凭据公开为 OpenAI 兼容端点
title: Claude Max API 代理
x-i18n:
    generated_at: "2026-07-26T06:56:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5d0d9a70e14d7d444e57e9bcf169816fec4013a2680dfc9b1761e6ab32109e9f
    source_path: providers/claude-max-api-proxy.md
    workflow: 16
---

**claude-max-api-proxy** 是一个社区 npm 软件包（不是 OpenClaw 插件），它将 Claude Max/Pro 订阅公开为兼容 OpenAI 的 API 端点，因此你可以将任何兼容 OpenAI 的工具指向你的订阅，而无需使用 Anthropic API key。

<Warning>
这仅表示技术上兼容，并非官方认可的使用方式。Anthropic 过去曾阻止在 Claude Code 之外使用某些订阅；在依赖此方式之前，请确认 Anthropic 当前的计费规则。

Anthropic 的 Claude Code 文档将 `claude -p` 描述为 Agent SDK/编程式用法。根据 Anthropic 于 2026 年 6 月 15 日发布的支持更新，Claude Agent SDK、`claude -p` 和第三方应用的使用量均计入已登录订阅的使用限额（此前宣布的独立 Agent SDK 点数计划已暂停）。请参阅 Anthropic 的 [Agent SDK 套餐文章](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)、[Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan) 和 [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan) 套餐文章，以及 [Anthropic 提供商](/zh-CN/providers/anthropic)中有关 OpenClaw 自身 Claude CLI 计费的说明。
</Warning>

## 为什么使用此方式

| 方式                      | 费用途径                                        | 最适合                                     |
| ------------------------- | ----------------------------------------------- | ------------------------------------------ |
| Anthropic API key         | 通过 Claude Console 按 token 付费               | 生产应用、共享自动化、大用量场景           |
| Claude 订阅代理           | Claude Code / `claude -p` 套餐和点数规则 | 使用兼容工具进行个人实验                   |

此代理让 Claude Max 或 Pro 订阅能够与兼容 OpenAI 的工具配合使用。它并非不限量的固定费率方案，而是沿用 Claude Code 的使用限额。对于生产用途，API key 仍然是计费方式更明确的选择。

## 工作原理

```text
你的应用 -> claude-max-api-proxy -> Claude Code CLI / claude -p -> Anthropic
       （OpenAI 格式）              （转换格式）               （使用你的登录状态）
```

代理会为每个请求将 Claude Code CLI 作为子进程启动，把 OpenAI 格式的聊天请求转换为 CLI 提示词，并以 OpenAI 格式将响应流式传回（或直接返回）。

## 入门指南

<Steps>
  <Step title="安装代理">
    需要 Node.js 20+，以及已完成身份验证的 Claude Code CLI。

    ```bash
    npm install -g claude-max-api-proxy

    # 确认 Claude CLI 已完成身份验证
    claude --version
    claude auth login   # 如果尚未完成身份验证
    ```

  </Step>
  <Step title="启动服务器">
    ```bash
    claude-max-api
    # 服务器运行于 http://localhost:3456
    ```
  </Step>
  <Step title="测试代理">
    ```bash
    curl http://localhost:3456/health
    curl http://localhost:3456/v1/models

    curl http://localhost:3456/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "claude-opus-4",
        "messages": [{"role": "user", "content": "你好！"}]
      }'
    ```

  </Step>
  <Step title="配置 OpenClaw">
    将 OpenClaw 指向该代理，并将其用作自定义的兼容 OpenAI 端点：

    ```json5
    {
      env: {
        OPENAI_API_KEY: "not-needed",
        OPENAI_BASE_URL: "http://localhost:3456/v1",
      },
      agents: {
        defaults: {
          model: { primary: "openai/claude-opus-4" },
        },
      },
    }
    ```

  </Step>
</Steps>

<Note>
下列模型 ID 属于代理自身的目录，而不是 OpenClaw 的 Anthropic 模型引用。每个 ID 都映射到一个 Claude Code CLI 模型别名（`opus`、`sonnet`、`haiku`），因此每当 Anthropic 在 CLI 中更新该别名时，底层模型也会随之变化。在依赖特定映射之前，请查看代理当前的 README。
</Note>

| 模型 ID           | CLI 别名  | 当前映射        |
| ----------------- | --------- | --------------- |
| `claude-opus-4`   | `opus`    | Claude Opus 4.5 |
| `claude-sonnet-4` | `sonnet`  | Claude Sonnet 4 |
| `claude-haiku-4`  | `haiku`   | Claude Haiku 4  |

## 高级配置

<AccordionGroup>
  <Accordion title="代理式兼容 OpenAI 端点说明">
    此方式使用 OpenClaw 的通用自定义 `/v1` 兼容 OpenAI 路由，与其他任何自托管的兼容 OpenAI 后端使用相同路径：

    - 仅适用于原生 OpenAI 的请求整形不会生效。
    - `/fast` 和 `service_tier` 仅适用于直接的 `api.anthropic.com` 流量；代理路由不会修改 `service_tier`（请参阅 [Anthropic 提供商快速模式](/zh-CN/providers/anthropic#advanced-configuration)）。
    - 不进行 Responses `store`、提示词缓存提示或 OpenAI 推理兼容载荷整形。
    - OpenClaw 的 OpenAI/Codex 归属标头（`originator`、`version`、`User-Agent`）仅随原生 `api.openai.com` OAuth 流量发送，不会发送到此代理之类的自定义 `OPENAI_BASE_URL` 目标。

  </Accordion>

  <Accordion title="在 macOS 上使用 LaunchAgent 自动启动">
    ```bash
    cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
      <key>Label</key>
      <string>com.claude-max-api</string>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
      <key>ProgramArguments</key>
      <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
      </array>
      <key>EnvironmentVariables</key>
      <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
      </dict>
    </dict>
    </plist>
    EOF

    launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
    ```

  </Accordion>
</AccordionGroup>

## 注意事项

- 沿用 Claude Code 的 `claude -p` 计费、使用点数和速率限制行为。
- 仅绑定到 `127.0.0.1`；除了 CLI 自身对 Anthropic 的调用外，不会向任何第三方服务器发送数据。
- 支持流式响应。
- 启动时不会检查身份验证失败，只有实际运行聊天请求时才会显现；如果 CLI 未完成身份验证，预期首次请求会失败，而不是服务器拒绝启动。

<Note>
如需通过 Claude CLI 或 API key 使用原生 Anthropic 集成，请参阅 [Anthropic 提供商](/zh-CN/providers/anthropic)。如需使用 OpenAI/Codex 订阅，请参阅 [OpenAI provider](/zh-CN/providers/openai)。
</Note>

## 相关内容

<CardGroup cols={2}>
  <Card title="Anthropic 提供商" href="/zh-CN/providers/anthropic" icon="bolt">
    通过 Claude CLI 或 API key 使用原生 OpenClaw 集成。
  </Card>
  <Card title="OpenAI provider" href="/zh-CN/providers/openai" icon="robot">
    用于 OpenAI/Codex 订阅。
  </Card>
  <Card title="模型选择" href="/zh-CN/concepts/model-providers" icon="layers">
    所有提供商、模型引用和故障转移行为的概览。
  </Card>
  <Card title="配置" href="/zh-CN/gateway/configuration" icon="gear">
    完整配置参考。
  </Card>
</CardGroup>
