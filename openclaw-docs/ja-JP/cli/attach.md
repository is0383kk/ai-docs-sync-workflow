---
read_when:
    - Claude Code で OpenClaw Gateway の MCP ツールを使用する場合
    - 外部ハーネス用に、セッションに紐づく一時的な MCP 権限付与が必要です
summary: '`openclaw attach` の CLI リファレンス（スコープ指定された Gateway MCP 権限で Claude Code を起動）'
title: CLI を接続
x-i18n:
    generated_at: "2026-07-26T09:15:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d8ac60724adef1439af09179806af537b8f2925f06b3715850e4dd3b83b080f
    source_path: cli/attach.md
    workflow: 16
---

`openclaw attach` は、1 つの Gateway セッションにバインドされた厳格な一時 MCP 設定で Claude Code を起動します。

```sh
openclaw attach
openclaw attach --session agent:main:telegram:123 --ttl 600000
openclaw attach --print-config
```

オプション:

- `--session <key>` は、許可を Gateway セッションにバインドします。デフォルトはメインセッションです。
- `--ttl <ms>` は、ミリ秒単位の正の許可 TTL を要求します。Gateway は独自の上限を適用します。
- `--bin <path>` は、Claude Code バイナリを選択します。デフォルト: `claude`。
- `--print-config` は、一時的な `.mcp.json` を書き込み、起動コマンドと環境変数を表示し、TTL が期限切れになるまで許可を有効なままにします（Claude Code のプロセスは生成せず、許可も取り消しません）。

Bearer トークンは argv ではなく、環境変数を介して渡されます。OpenClaw は `--strict-mcp-config --mcp-config <path>` を指定して Claude Code を起動するため、環境内の Claude MCP サーバーが接続済みセッションに参加することはありません。通常の起動（`--print-config` なし）では、Claude Code プロセスの終了時に許可が取り消されます。

関連項目: [Gateway CLI](/ja-JP/cli/gateway)、[MCP CLI](/ja-JP/cli/mcp)、[ACP CLI](/ja-JP/cli/acp)。
