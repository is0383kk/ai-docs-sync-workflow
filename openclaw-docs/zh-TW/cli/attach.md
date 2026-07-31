---
read_when:
    - 你想要 Claude Code 使用 OpenClaw 閘道 MCP 工具
    - 你需要為外部測試框架提供暫時且綁定工作階段的 MCP 授權
summary: '`openclaw attach` 的命令列介面參考（使用範圍限定的閘道 MCP 授權啟動 Claude Code）'
title: 附加命令列介面
x-i18n:
    generated_at: "2026-07-26T07:46:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d8ac60724adef1439af09179806af537b8f2925f06b3715850e4dd3b83b080f
    source_path: cli/attach.md
    workflow: 16
---

`openclaw attach` 會使用綁定至單一閘道工作階段的嚴格臨時 MCP 設定來啟動 Claude Code。

```sh
openclaw attach
openclaw attach --session agent:main:telegram:123 --ttl 600000
openclaw attach --print-config
```

選項：

- `--session <key>` 會將授權綁定至閘道工作階段。預設為主要工作階段。
- `--ttl <ms>` 會要求以毫秒為單位的正值授權存續時間。閘道會套用自身的上限。
- `--bin <path>` 會選取 Claude Code 執行檔。預設值：`claude`。
- `--print-config` 會寫入臨時的 `.mcp.json`、輸出啟動命令與環境變數，並讓授權持續有效至存續時間屆滿（不會產生 Claude Code 程序或撤銷授權）。

持有者權杖會透過環境變數而非 argv 傳遞。OpenClaw 會使用 `--strict-mcp-config --mcp-config <path>` 啟動 Claude Code，因此環境中的 Claude MCP 伺服器不會加入已附加的工作階段。一般啟動（不含 `--print-config`）會在 Claude Code 程序結束時撤銷授權。

另請參閱：[閘道命令列介面](/zh-TW/cli/gateway)、[MCP 命令列介面](/zh-TW/cli/mcp)和 [ACP 命令列介面](/zh-TW/cli/acp)。
