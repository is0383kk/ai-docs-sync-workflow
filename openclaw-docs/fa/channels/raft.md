---
read_when:
    - می‌خواهید OpenClaw را به یک فضای کاری Raft متصل کنید
    - در حال پیکربندی یک عامل خارجی Raft هستید
    - در حال اشکال‌زدایی تحویل بیدارباش Raft هستید
sidebarTitle: Raft
summary: پشتیبانی از عامل خارجی Raft از طریق پل بیدارسازی CLI‏ Raft
title: Raft
x-i18n:
    generated_at: "2026-07-27T16:14:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 454d92d764a4ec3b0ec52467cba254dcad795870e04d1d32d4cf65d8b451a0de
    source_path: channels/raft.md
    workflow: 16
---

Raft یک عامل OpenClaw را از طریق CLI محلی Raft به یک عامل خارجی Raft
متصل می‌کند. Raft اعلان‌های بیدارباش احراز هویت‌شده را به Gateway می‌فرستد؛ سپس عامل
از CLI ‏Raft برای بررسی و ارسال پیام‌ها استفاده می‌کند. فقط گفت‌وگوی مستقیم (بدون گروه).

## نصب

Raft یک Plugin خارجی رسمی است. آن را روی میزبان Gateway نصب کنید:

```bash
openclaw plugins install @openclaw/raft
openclaw gateway restart
```

جزئیات: [Pluginها](/fa/tools/plugin)

## پیش‌نیازها

- یک فضای کاری Raft با یک عامل خارجی.
- CLI ‏Raft نصب‌شده روی همان میزبان Gateway ‏OpenClaw، در `PATH` سرویس.
- یک پروفایل CLI ‏Raft که از قبل وارد شده و با آن
  عامل خارجی مرتبط است.

Plugin اعتبارنامه‌های Raft را ذخیره نمی‌کند؛ CLI ‏Raft آن
احراز هویت را در پروفایل خودش نگه می‌دارد.

## پیکربندی

پروفایل را در پیکربندی تنظیم کنید:

```json5
{
  channels: {
    raft: {
      enabled: true,
      profile: "openclaw",
    },
  },
}
```

برای حساب پیش‌فرض، در عوض می‌توانید `RAFT_PROFILE` را در محیط Gateway
تنظیم کنید:

```bash
RAFT_PROFILE=openclaw
```

هنگامی که یک Gateway به بیش از یک عامل خارجی Raft متصل می‌شود، از یک حساب نام‌گذاری‌شده استفاده کنید:

```json5
{
  channels: {
    raft: {
      accounts: {
        support: {
          profile: "support-agent",
        },
        engineering: {
          profile: "engineering-agent",
        },
      },
    },
  },
}
```

راه‌اندازی تعاملی همان پروفایل را ثبت می‌کند:

```bash
openclaw channels add --channel raft
```

## نحوه کار

هنگام شروع Gateway، ‏Plugin:

1. یک نقطه پایانی HTTP بیدارباش را که فقط روی loopback در دسترس است، روی یک درگاه موقت باز می‌کند.
2. `raft --profile <profile> agent bridge` را با آن نقطه پایانی و یک
   توکن مختص هر فرایند راه‌اندازی می‌کند.
3. فقط اعلان‌های بیدارباش احراز هویت‌شده و بدون محتوا را که دارای شناسه بازپخش
   از پل محلی هستند، می‌پذیرد.
4. وجود یکی از `eventId`، `attemptId`، `messageId`، `delivery_id`،
   `wake_id` یا `id` را در هر محموله بیدارباش الزامی می‌کند.
5. تحویل‌های مجدد بیدارباش را بر اساس شناسه رویداد پل به‌مدت 24 ساعت،
   حتی در میان راه‌اندازی‌های مجدد Gateway، حذف تکراری می‌کند.
6. یک نشست پایدار زمان اجرا برای پل فعلی و یک دسته خالی
   تخلیه فعالیت برای پروتکل CLI ‏Raft برمی‌گرداند.
7. برای هر بیدارباش پذیرفته‌شده، یک نوبت سریالی عامل OpenClaw را آغاز می‌کند.

پل مسئول تلاش‌های مجدد تحویل و اتصال‌های مجدد Raft است. نوبت OpenClaw
فقط یک اعلان بیدارباش دریافت می‌کند، نه نسخه‌ای از بدنه پیام Raft. عامل از CLI
برای خواندن پیام‌های در انتظار و ارسال پاسخ خود استفاده می‌کند:

```bash
raft --profile openclaw message check
raft --profile openclaw message send
```

<Note>
Raft یک انتقال‌دهنده پیام فشاری نیست. OpenClaw متن نهایی مدل را به‌طور خودکار از طریق پل بازنمی‌فرستد؛ بنابراین عامل باید پس از پردازش بیدارباش از CLI ‏Raft استفاده کند.
</Note>

## تأیید

بررسی کنید که OpenClaw می‌تواند CLI را پیدا کند و یک پروفایل پیکربندی‌شده دارد:

```bash
openclaw channels status --probe
openclaw plugins inspect raft --runtime --json
```

سپس پیامی به عامل خارجی Raft بفرستید. گزارش Gateway باید ابتدا
راه‌اندازی پل Raft و سپس یک بیدارباش ورودی را نشان دهد. عامل باید از
پروفایل پیکربندی‌شده Raft برای بررسی پیام‌های در انتظار خود استفاده کند.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="CLI ‏Raft موجود نیست">
    CLI ‏Raft را روی میزبان Gateway نصب کنید و `raft` را در
    `PATH` سرویس در دسترس قرار دهید. آن را با `raft --help` تأیید کنید، سپس Gateway را دوباره راه‌اندازی کنید.
  </Accordion>
  <Accordion title="پل بلافاصله خارج می‌شود">
    تأیید کنید که پروفایل پیکربندی‌شده وارد شده و متعلق به عامل خارجی
    Raft موردنظر است. برای مشاهده عیب‌یابی CLI، ‏`raft --profile <profile> agent bridge` را مستقیماً
    اجرا کنید.
  </Accordion>
  <Accordion title="یک بیدارباش می‌رسد، اما هیچ پاسخ Raft ارسال نمی‌شود">
    وقتی عامل CLI ‏Raft را فراخوانی نمی‌کند، این رفتار مورد انتظار است. پل
    بیدارباش بدنه پیام‌ها یا پاسخ‌های نهایی خودکار را حمل نمی‌کند. سیاست ابزار
    عامل را بررسی کنید و مطمئن شوید که می‌تواند `raft --profile <profile>
    message check` و `message send` را اجرا کند.
  </Accordion>
</AccordionGroup>

## منابع

- [Raft](https://raft.build/)
- [مستندات Raft](https://docs.raft.build/welcome/)
- [یکپارچه‌سازی Raft در Hermes](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/raft)
