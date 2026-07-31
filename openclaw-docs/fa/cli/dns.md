---
read_when:
    - می‌خواهید کشف در شبکه گسترده (DNS-SD) را از طریق Tailscale + CoreDNS انجام دهید
    - You're setting up split DNS for a custom discovery domain (example: openclaw.internal)
summary: مرجع CLI برای `openclaw dns` (ابزارهای کمکی کشف در گسترهٔ وسیع)
title: DNS
x-i18n:
    generated_at: "2026-07-27T16:19:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bb07353df03f9d169e1aede2da0b711ffb68e8c9d21d51359e93e92cc0818ca2
    source_path: cli/dns.md
    workflow: 16
---

# `openclaw dns`

ابزارهای کمکی DNS برای کشف در شبکه گسترده (Tailscale + CoreDNS). در حال حاضر فقط از macOS + Homebrew CoreDNS پشتیبانی می‌شود.

مرتبط:

- کشف Gateway: [کشف](/fa/gateway/discovery)
- پیکربندی کشف در شبکه گسترده: [پیکربندی](/fa/gateway/configuration)

## `dns setup`

راه‌اندازی CoreDNS را برای کشف DNS-SD تک‌پخشی برنامه‌ریزی یا اعمال کنید.

```bash
openclaw dns setup
openclaw dns setup --domain openclaw.internal
openclaw dns setup --apply
```

| گزینه              | اثر                                                                              |
| ------------------- | ----------------------------------------------------------------------------------- |
| `--domain <domain>` | دامنه کشف در شبکه گسترده (برای مثال `openclaw.internal`).                       |
| `--apply`           | پیکربندی CoreDNS را نصب/به‌روزرسانی می‌کند و سرویس را (دوباره) راه‌اندازی می‌کند. به sudo نیاز دارد و فقط برای macOS است. |

بدون `--domain`، OpenClaw از `discovery.wideArea.domain` موجود در پیکربندی استفاده می‌کند.

بدون `--apply`، فرمان فقط موارد زیر را چاپ می‌کند:

- دامنه کشف حل‌شده و مسیر فایل ناحیه
- IPهای فعلی tailnet
- پیکربندی پیشنهادی کشف `openclaw.json`
- مقادیر کارساز نام/دامنه Split DNS در Tailscale که باید در کنسول مدیریت Tailscale تنظیم شوند

با `--apply` (فقط macOS، نیازمند Homebrew CoreDNS):

- اگر فایل ناحیه وجود نداشته باشد، آن را راه‌اندازی اولیه می‌کند
- اگر بند import مربوط به CoreDNS وجود نداشته باشد، آن را اضافه می‌کند
- سرویس brew با نام `coredns` را دوباره راه‌اندازی می‌کند

## مرتبط

- [مرجع CLI](/fa/cli)
- [کشف](/fa/gateway/discovery)
