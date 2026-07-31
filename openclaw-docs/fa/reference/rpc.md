---
read_when:
    - افزودن یا تغییر یکپارچه‌سازی‌های CLI خارجی
    - اشکال‌زدایی آداپتورهای RPC ‏(signal-cli، imsg)
summary: آداپتورهای RPC برای CLIهای خارجی (signal-cli، imsg) و الگوهای Gateway
title: آداپتورهای RPC
x-i18n:
    generated_at: "2026-07-27T15:47:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw از طریق JSON-RPC با CLIهای خارجی یکپارچه می‌شود. امروزه از دو الگو استفاده می‌شود.

## الگوی A: دیمن HTTP (signal-cli)

- `signal-cli` به‌صورت دیمن با JSON-RPC روی HTTP اجرا می‌شود.
- جریان رویداد SSE است (`/api/v1/events`).
- پروب سلامت: `/api/v1/check`.
- هنگامی که `channels.signal.transport.kind="managed-native"` باشد (حالت پیش‌فرض)، OpenClaw چرخهٔ حیات را مدیریت می‌کند.

برای راه‌اندازی و نقاط پایانی، [Signal](/fa/channels/signal) را ببینید.

## الگوی B: فرایند فرزند stdio (imsg)

- OpenClaw برای [iMessage](/fa/channels/imessage)، ‏`imsg rpc` را به‌عنوان فرایند فرزند اجرا می‌کند.
- JSON-RPC روی stdin/stdout به‌صورت خط‌به‌خط جدا می‌شود (یک شیء JSON در هر خط).
- به درگاه TCP یا دیمن نیازی نیست.

متدهای اصلی مورداستفاده:

- `watch.subscribe` ← اعلان‌ها (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (پروب/عیب‌یابی)

برای راه‌اندازی و آدرس‌دهی، [iMessage](/fa/channels/imessage) را ببینید (`chat_id` بر رشته‌های نمایشی ترجیح دارد).

## دستورالعمل‌های آداپتور

- Gateway فرایند را مدیریت می‌کند (شروع/توقف به چرخهٔ حیات ارائه‌دهنده وابسته است).
- کلاینت‌های RPC را تاب‌آور نگه دارید: مهلت‌های زمانی و راه‌اندازی مجدد هنگام خروج.
- شناسه‌های پایدار (برای مثال، `chat_id`) را بر رشته‌های نمایشی ترجیح دهید.

## مرتبط

- [پروتکل Gateway](/fa/gateway/protocol)
