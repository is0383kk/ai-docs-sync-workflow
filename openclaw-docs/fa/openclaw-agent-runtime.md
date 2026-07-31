---
read_when:
    - کار روی کد یا آزمون‌های زمان اجرای عامل OpenClaw
    - اجرای جریان‌های lint، بررسی نوع و آزمون زندهٔ زمان‌اجرای عامل
summary: 'گردش‌کار توسعه‌دهنده برای زمان اجرای عامل OpenClaw: ساخت، آزمایش و اعتبارسنجی زنده'
title: گردش کار زمان اجرای عامل OpenClaw
x-i18n:
    generated_at: "2026-07-27T15:48:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 044f05779bef4ad18478081ba44d84356723c8a0be764440aa9d2b976d167324
    source_path: openclaw-agent-runtime.md
    workflow: 16
---

گردش‌کار توسعه‌دهنده برای زمان‌اجرای عامل (`src/agents/`) در مخزن OpenClaw.

## بررسی نوع و لینت

- گیت محلی پیش‌فرض: `pnpm check` (بررسی نوع، لینت، محافظ‌های سیاست)
- گیت ساخت: `pnpm build` هنگامی که تغییر می‌تواند بر خروجی ساخت، بسته‌بندی یا مرزهای بارگذاری تنبل/ماژول اثر بگذارد
- گیت کامل پیش از پوش: `pnpm build && pnpm check && pnpm check:test-types && pnpm test`

## اجرای آزمون‌های زمان‌اجرای عامل

مجموعه‌های آزمون واحد زمان‌اجرای عامل را اجرا کنید:

```bash
pnpm test \
  "src/agents/agent-*.test.ts" \
  "src/agents/embedded-agent-*.test.ts" \
  "src/agents/agent-hooks/**/*.test.ts"
```

الگوی glob نخست، مجموعه‌های `agent-tools*`، `agent-settings` و
`agent-tool-definition-adapter*` را نیز پوشش می‌دهد.

آزمون‌های زنده از پیکربندی واحد مستثنا هستند؛ آن‌ها را از طریق پوشش‌دهنده زنده
اجرا کنید (`OPENCLAW_LIVE_TEST=1` را تنظیم می‌کند و به اعتبارنامه‌های ارائه‌دهنده نیاز دارد):

```bash
pnpm test:live src/agents/embedded-agent-runner-extraparams.live.test.ts
```

## آزمون دستی

- Gateway را در حالت توسعه اجرا کنید (اتصال‌های کانال از طریق `OPENCLAW_SKIP_CHANNELS=1` نادیده گرفته می‌شوند): `pnpm gateway:dev`
- یک نوبت عامل را از طریق Gateway راه‌اندازی کنید: `pnpm openclaw agent --message "Hello" --thinking low`
- برای اشکال‌زدایی تعاملی از TUI استفاده کنید: `pnpm tui`

برای بررسی رفتار فراخوانی ابزار، یک کنش `read` یا `exec` را درخواست کنید تا بتوانید
جریان ابزار و مدیریت بار داده را مشاهده کنید.

## بازنشانی کامل

وضعیت به‌طور پیش‌فرض در پوشه وضعیت OpenClaw یعنی `~/.openclaw` قرار دارد، یا در صورت
تنظیم‌شدن `$OPENCLAW_STATE_DIR`. مسیرهای نسبی به آن پوشه:

| مسیر                                           | محتوا                                                              |
| ---------------------------------------------- | ------------------------------------------------------------------ |
| `openclaw.json`                                | پیکربندی                                                             |
| `state/openclaw.sqlite`                        | پایگاه داده وضعیت مشترک زمان‌اجرا                                      |
| `agents/<agentId>/agent/openclaw-agent.sqlite` | نمایه‌های احراز هویت مدل برای هر عامل (کلیدهای API + OAuth) و وضعیت زمان‌اجرا |
| `credentials/`                                 | اعتبارنامه‌های ارائه‌دهنده/کانال خارج از مخزن نمایه احراز هویت        |
| `agents/<agentId>/sessions/`                   | تاریخچه رونوشت و منابع مهاجرت نشست قدیمی            |
| `sessions/`                                    | مخزن قدیمی نشست تک‌عاملی (فقط نصب‌های قدیمی)              |
| `workspace/`                                   | فضای کاری عامل پیش‌فرض (عامل‌های اضافی از `workspace-<agentId>` استفاده می‌کنند)   |

برای بازنشانی کامل، آن مسیرها را حذف کنید. بازنشانی‌های محدودتر:

- فقط نشست‌ها: `agents/<agentId>/agent/openclaw-agent.sqlite` را حذف نکنید؛ ردیف‌های نشست در کنار سایر وضعیت‌های هر عامل در آن قرار دارند. برای شروع یک نشست تازه برای یک گفت‌وگو، از `/new` یا `/reset` و برای نگه‌داری نشست از `openclaw sessions cleanup` استفاده کنید.
- حفظ احراز هویت: `agents/<agentId>/agent/openclaw-agent.sqlite` و `credentials/` را در جای خود نگه دارید.

فایل‌های قدیمی `auth-profiles.json` دیگر در زمان اجرا خوانده نمی‌شوند؛
`openclaw doctor --fix` آن‌ها را به مخزن SQLite وارد می‌کند.

## منابع

- [آزمون](/fa/help/testing)
- [شروع به کار](/fa/start/getting-started)

## مرتبط

- [معماری زمان‌اجرای عامل OpenClaw](/fa/agent-runtime-architecture)
