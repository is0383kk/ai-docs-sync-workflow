---
read_when:
    - می‌خواهید Claude Code از ابزارهای MCP مربوط به OpenClaw Gateway استفاده کند
    - برای یک چارچوب آزمایشی خارجی به یک مجوز موقت MCP وابسته به نشست نیاز دارید
summary: مرجع CLI برای `openclaw attach` (اجرای Claude Code با مجوز محدودشدهٔ Gateway MCP)
title: اتصال CLI
x-i18n:
    generated_at: "2026-07-27T14:58:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d8ac60724adef1439af09179806af537b8f2925f06b3715850e4dd3b83b080f
    source_path: cli/attach.md
    workflow: 16
---

`openclaw attach`، Claude Code را با یک پیکربندی موقت و سخت‌گیرانهٔ MCP که به یک نشست Gateway متصل است، اجرا می‌کند.

```sh
openclaw attach
openclaw attach --session agent:main:telegram:123 --ttl 600000
openclaw attach --print-config
```

گزینه‌ها:

- `--session <key>` مجوز را به یک نشست Gateway متصل می‌کند. مقدار پیش‌فرض، نشست اصلی است.
- `--ttl <ms>` یک TTL مثبت برای مجوز، برحسب میلی‌ثانیه، درخواست می‌کند. Gateway سقف محدودیت خود را اعمال می‌کند.
- `--bin <path>` فایل اجرایی Claude Code را انتخاب می‌کند. پیش‌فرض: `claude`.
- `--print-config` فایل موقت `.mcp.json` را می‌نویسد، فرمان اجرا و متغیرهای محیطی را چاپ می‌کند و مجوز را تا پایان TTL فعال نگه می‌دارد (Claude Code را اجرا نمی‌کند و مجوز را لغو نمی‌کند).

توکن حامل از طریق متغیرهای محیطی منتقل می‌شود، نه argv. OpenClaw، Claude Code را با `--strict-mcp-config --mcp-config <path>` اجرا می‌کند تا سرورهای محیطی Claude MCP به نشست متصل‌شده نپیوندند. اجراهای عادی (بدون `--print-config`) هنگام خروج فرایند Claude Code، مجوز را لغو می‌کنند.

همچنین ببینید: [CLI مربوط به Gateway](/fa/cli/gateway)، [CLI مربوط به MCP](/fa/cli/mcp) و [CLI مربوط به ACP](/fa/cli/acp).
