---
read_when:
    - بررسی خرابی بارگذار tsx/esbuild که به نبود helper با نام __name اشاره می‌کند
summary: خرابی تاریخی Node + tsx با خطای «__name is not a function» و علت آن
title: کرش Node + tsx
x-i18n:
    generated_at: "2026-07-27T14:03:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97d2f62d24860cee65753027ba84c14c8d4ffb910ee17bb0032cf0409c427589
    source_path: debug/node-issue.md
    workflow: 16
---

# ازکارافتادگی Node + tsx با خطای "\_\_name is not a function"

## وضعیت

برطرف شده است. این ازکارافتادگی در نسخهٔ فعلی `tsx` که در
`package.json` (`4.22.3`) پین شده، یا در نسخه‌های فعلی Node بازتولید نمی‌شود. این مطلب برای حالتی نگه داشته شده است که
ارتقای آیندهٔ `tsx`/esbuild دوباره آن را ایجاد کند.

## نشانهٔ اولیه

اجرای اسکریپت‌های توسعهٔ OpenClaw از طریق `tsx` هنگام راه‌اندازی با خطای زیر مواجه می‌شد:

```text
[openclaw] راه‌اندازی CLI ناموفق بود: TypeError: __name is not a function
    در createSubsystemLogger (src/logging/subsystem.ts)
    در <caller> (src/agents/auth-profiles/constants.ts)
```

شماره‌خط‌ها حذف شده‌اند؛ هر دو فایل از زمان ازکارافتادگی اولیه تغییر کرده‌اند
و خطوط مشخص‌شده دیگر مطابقت ندارند.

این مشکل پس از آن ظاهر شد که اسکریپت‌های توسعه از Bun به `tsx` (`2871657e`،
2026-01-06) تغییر کردند تا Bun اختیاری شود. مسیر معادل مبتنی بر Bun دچار ازکارافتادگی نمی‌شد.
این مشکل نخستین‌بار در Node v25.3.0 روی macOS مشاهده شد؛ احتمال می‌رفت سایر پلتفرم‌هایی که
Node 25 را اجرا می‌کنند نیز تحت تأثیر قرار گیرند.

## علت

`tsx` کد TS/ESM را از طریق esbuild و با `keepNames: true` که در
گزینه‌های تبدیل آن به‌صورت ثابت تعریف شده است، تبدیل می‌کند. این تنظیم باعث می‌شود esbuild اعلان‌های نام‌دار تابع/کلاس را
در فراخوانی یک تابع کمکی `__name` قرار دهد تا `fn.name` پس از کوچک‌سازی
و بسته‌بندی حفظ شود. ازکارافتادگی به این معناست که در محل فراخوانی آن ماژول، در ترکیب تحت تأثیر
`tsx`/Node، تابع کمکی وجود نداشت یا تحت‌الشعاع قرار گرفته بود؛ بنابراین `__name(...)`
به‌جای بازگرداندن مقدار پوشش‌داده‌شده، خطا ایجاد کرد.

## بررسی فعلی بازتولید

```bash
node --version
pnpm install
node --import tsx src/entry.ts status
```

بازتولید حداقلی و ایزوله‌شده (فقط ماژول موجود در ردپای پشتهٔ اولیه را بارگذاری می‌کند):

```bash
node --import tsx scripts/repro/tsx-name-repro.ts
```

در حال حاضر هر دو فرمان بدون خطا خاتمه می‌یابند. اگر هرکدام دوباره `__name is not a
function` ایجاد کرد، پیش از ثبت گزارش در پروژهٔ بالادستی، نسخهٔ دقیق Node، نسخهٔ `tsx`
(`node_modules/tsx/package.json`) و ردپای کامل پشته را ثبت کنید.

## راهکارهای موقت (اگر ازکارافتادگی بازگردد)

- اسکریپت‌های توسعه را به‌جای `node --import tsx` با Bun اجرا کنید.
- برای بررسی نوع‌ها، `pnpm tsgo` را اجرا کنید؛ سپس به‌جای اجرای کد منبع از طریق
  `tsx`، خروجی ساخته‌شده را اجرا کنید:

  ```bash
  pnpm tsgo
  node openclaw.mjs status
  ```

- نسخهٔ دیگری از `tsx` را امتحان کنید (`pnpm add -D tsx@<version>` تغییری در وابستگی است
  و طبق خط‌مشی مخزن به تأیید نیاز دارد) تا با دونیم‌سازی مشخص شود آیا نسخهٔ esbuild
  همراه آن دوباره باگ را ایجاد کرده است.
- روی نسخهٔ اصلی/فرعی دیگری از Node آزمایش کنید تا مشخص شود آیا خرابی
  مختص نسخه است.

## منابع

- [https://esbuild.github.io/api/#keep-names](https://esbuild.github.io/api/#keep-names)
- [https://github.com/evanw/esbuild/issues/1031](https://github.com/evanw/esbuild/issues/1031)

## مرتبط

- [نصب Node.js](/fa/install/node)
- [عیب‌یابی Gateway](/fa/gateway/troubleshooting)
