---
read_when:
    - شما خواهان پشتیبانی از Zalo Personal (غیررسمی) در OpenClaw هستید
    - در حال پیکربندی یا توسعه Plugin ‏zalouser هستید
summary: 'Plugin شخصی Zalo: ورود با کد QR + پیام‌رسانی از طریق zca-js بومی (نصب Plugin + پیکربندی کانال + ابزار)'
title: Plugin شخصی Zalo
x-i18n:
    generated_at: "2026-07-27T17:02:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb0bdaa10340b5d78dc32abf6b0520fda6cf5f65e2e17b551b4e9bd72acfbbf2
    source_path: plugins/zalouser.md
    workflow: 16
---

پشتیبانی از Zalo Personal برای OpenClaw از طریق Pluginی که با استفاده از `zca-js` بومی، یک حساب کاربری عادی Zalo را
خودکارسازی می‌کند. هیچ فایل اجرایی CLI خارجی `zca`/`openzca`
لازم نیست.

<Warning>
خودکارسازی غیررسمی ممکن است به تعلیق یا مسدودشدن حساب منجر شود. مسئولیت استفاده بر عهده خودتان است.
</Warning>

## نام‌گذاری

شناسه کانال `zalouser` است تا به‌صراحت مشخص شود که این کانال یک **حساب کاربری شخصی
Zalo** را خودکارسازی می‌کند (غیررسمی). شناسه کانال جداگانه `zalo` مربوط به یکپارچه‌سازی رسمی و
همراه Zalo Bot/Webhook است — به [Zalo](/fa/channels/zalo) مراجعه کنید.

## محل اجرا

این Plugin **درون فرایند Gateway** اجرا می‌شود. برای یک Gateway راه‌دور،
آن را روی همان میزبان نصب و پیکربندی کنید، سپس Gateway را دوباره راه‌اندازی کنید.

## نصب

### از npm

```bash
openclaw plugins install @openclaw/zalouser
```

برای دنبال‌کردن برچسب انتشار رسمی فعلی، از نام بسته بدون نسخه استفاده کنید؛ فقط زمانی یک
نسخه دقیق را تثبیت کنید که به نصب تکرارپذیر نیاز دارید. سپس Gateway را
دوباره راه‌اندازی کنید.

### از یک پوشه محلی (توسعه)

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

سپس Gateway را دوباره راه‌اندازی کنید.

## پیکربندی

پیکربندی کانال در `channels.zalouser` قرار دارد (نه `plugins.entries.*`):

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

برای کنترل دسترسی پیام مستقیم/گروه، راه‌اندازی چندحسابی، متغیرهای محیطی و عیب‌یابی، به
[پیکربندی کانال شخصی Zalo](/fa/channels/zalouser) مراجعه کنید.

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels login --channel zalouser --account <name>
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hello from OpenClaw"
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "name"
openclaw directory groups members --channel zalouser --group-id <id>
```

## ابزار عامل

نام ابزار: `zalouser`

کنش‌ها: `send`، `image`، `link`، `friends`، `groups`، `me`، `status`

کنش‌های پیام کانال (نه ابزار عامل) همچنین از `react` برای واکنش به
پیام‌ها پشتیبانی می‌کنند.

## مرتبط

- [پیکربندی کانال شخصی Zalo](/fa/channels/zalouser)
- [Zalo (کانال رسمی Bot/Webhook)](/fa/channels/zalo)
- [ساخت Pluginها](/fa/plugins/building-plugins)
- [ClawHub](/clawhub)
