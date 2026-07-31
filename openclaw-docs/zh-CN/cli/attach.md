---
read_when:
    - 你希望 Claude Code 使用 OpenClaw Gateway 网关 MCP 工具
    - 你需要为外部 harness 获取一个临时的、与会话绑定的 MCP 授权
summary: '`openclaw attach` 的 CLI 参考（使用限定范围的 Gateway 网关 MCP 授权启动 Claude Code）'
title: 附加 CLI
x-i18n:
    generated_at: "2026-07-26T06:08:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d8ac60724adef1439af09179806af537b8f2925f06b3715850e4dd3b83b080f
    source_path: cli/attach.md
    workflow: 16
---

`openclaw attach` 使用绑定到单个 Gateway 网关会话的严格临时 MCP 配置启动 Claude Code。

```sh
openclaw attach
openclaw attach --session agent:main:telegram:123 --ttl 600000
openclaw attach --print-config
```

选项：

- `--session <key>` 将授权绑定到一个 Gateway 网关会话。默认为主会话。
- `--ttl <ms>` 请求以毫秒为单位的正数授权 TTL。Gateway 网关会应用自身的上限。
- `--bin <path>` 选择 Claude Code 二进制文件。默认值：`claude`。
- `--print-config` 写入临时 `.mcp.json`，输出启动命令和环境变量，并使授权保持有效直至 TTL 到期（它不会启动 Claude Code，也不会撤销授权）。

持有者令牌通过环境变量而非 argv 传递。OpenClaw 使用 `--strict-mcp-config --mcp-config <path>` 启动 Claude Code，因此环境中已有的 Claude MCP 服务器不会加入所附加的会话。正常启动（不使用 `--print-config`）会在 Claude Code 进程退出时撤销授权。

另请参阅：[Gateway CLI](/zh-CN/cli/gateway)、[MCP CLI](/zh-CN/cli/mcp) 和 [ACP CLI](/zh-CN/cli/acp)。
