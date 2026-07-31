---
read_when:
    - می‌خواهید OpenClaw را به WeChat یا Weixin متصل کنید
    - در حال نصب یا عیب‌یابی Plugin کانال openclaw-weixin هستید
    - باید درک کنید که Pluginهای کانال خارجی چگونه در کنار Gateway اجرا می‌شوند
summary: راه‌اندازی کانال WeChat از طریق Plugin خارجی openclaw-weixin
title: وی‌چت
x-i18n:
    generated_at: "2026-07-27T13:54:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98faf95f9fb76deedb7df9adf3092083722a77bdd793de98c41a6f715cc0d14a
    source_path: channels/wechat.md
    workflow: 16
---

OpenClaw از طریق Plugin خارجی کانال
`@tencent-weixin/openclaw-weixin` متعلق به Tencent به WeChat متصل می‌شود.

وضعیت: Plugin خارجی، نگه‌داری‌شده توسط تیم Tencent Weixin. گفت‌وگوهای مستقیم و
رسانه پشتیبانی می‌شوند. گفت‌وگوهای گروهی در فراداده قابلیت‌های Plugin اعلام
نشده‌اند (این فراداده فقط گفت‌وگوهای مستقیم را اعلام می‌کند).

## نام‌گذاری

- **WeChat** نامی است که در این مستندات به کاربران نمایش داده می‌شود.
- **Weixin** نامی است که بسته Tencent و شناسه Plugin از آن استفاده می‌کنند.
- `openclaw-weixin` شناسه کانال OpenClaw است (`weixin` و `wechat` به‌عنوان نام مستعار کار می‌کنند).
- `@tencent-weixin/openclaw-weixin` بسته npm است.

در فرمان‌های CLI و مسیرهای پیکربندی از `openclaw-weixin` استفاده کنید.

## نحوه کار

کد WeChat در مخزن هسته OpenClaw قرار ندارد. OpenClaw قرارداد عمومی Plugin
کانال را فراهم می‌کند و Plugin خارجی، زمان اجرای ویژه WeChat را ارائه می‌دهد:

1. `openclaw plugins install`، `@tencent-weixin/openclaw-weixin` را نصب می‌کند.
2. Gateway مانیفست Plugin را شناسایی و نقطه ورود Plugin را بارگذاری می‌کند.
3. Plugin شناسه کانال `openclaw-weixin` را ثبت می‌کند.
4. `openclaw channels login --channel openclaw-weixin` ورود با کد QR را آغاز می‌کند.
5. Plugin اعتبارنامه‌های حساب را در پوشه وضعیت OpenClaw ذخیره می‌کند
   (به‌طور پیش‌فرض `~/.openclaw`).
6. هنگام راه‌اندازی Gateway، Plugin پایشگر Weixin خود را برای هر
   حساب پیکربندی‌شده راه‌اندازی می‌کند.
7. پیام‌های ورودی WeChat از طریق قرارداد کانال نرمال‌سازی می‌شوند، به
   عامل منتخب OpenClaw هدایت می‌شوند و از مسیر خروجی Plugin بازگردانده می‌شوند.

این جداسازی اهمیت دارد: هسته OpenClaw مستقل از کانال باقی می‌ماند. ورود WeChat،
فراخوانی‌های API مربوط به Tencent iLink، بارگذاری و دریافت رسانه، توکن‌های زمینه و
پایش حساب بر عهده Plugin خارجی هستند.

## نصب

نصب سریع:

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
```

نصب دستی:

```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
```

پس از نصب، Gateway را بازراه‌اندازی کنید:

```bash
openclaw gateway restart
```

## ورود

ورود با کد QR را روی همان دستگاهی اجرا کنید که Gateway روی آن اجرا می‌شود:

```bash
openclaw channels login --channel openclaw-weixin
```

کد QR را با WeChat روی تلفن خود اسکن و ورود را تأیید کنید. پس از اسکن موفق،
Plugin توکن حساب را به‌صورت محلی ذخیره می‌کند.

برای افزودن یک حساب WeChat دیگر، همان فرمان ورود را دوباره اجرا کنید. برای چند
حساب، نشست‌های پیام مستقیم را بر اساس حساب، کانال و فرستنده تفکیک کنید:

```bash
openclaw config set session.dmScope per-account-channel-peer
```

## کنترل دسترسی

پیام‌های مستقیم از مدل معمول جفت‌سازی و فهرست مجاز OpenClaw برای Pluginهای
کانال استفاده می‌کنند.

فرستندگان جدید را تأیید کنید:

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

برای مشاهده مدل کامل کنترل دسترسی، به [جفت‌سازی](/fa/channels/pairing) مراجعه کنید.

## سازگاری

Plugin هنگام راه‌اندازی، نسخه OpenClaw میزبان را بررسی می‌کند.

| سری Plugin | نسخه OpenClaw                                                | برچسب npm  |
| ----------- | --------------------------------------------------------------- | -------- |
| `2.x`       | `>=2026.5.12` (نسخه کنونی 2.4.6؛ نسخه‌های اولیه 2.x، مقدار `>=2026.3.22` را می‌پذیرفتند) | `latest` |
| `1.x`       | `>=2026.1.0 <2026.3.22`                                         | `legacy` |

اگر Plugin گزارش داد که نسخه OpenClaw شما بیش‌ازحد قدیمی است، OpenClaw را
به‌روزرسانی کنید یا سری قدیمی Plugin را نصب کنید:

```bash
openclaw plugins install @tencent-weixin/openclaw-weixin@legacy
```

## فرایند جانبی

Plugin مربوط به WeChat می‌تواند هنگام پایش API مربوط به Tencent iLink، کارهای
کمکی را در کنار Gateway اجرا کند. در ایشوی #68451، این مسیر کمکی یک باگ را در
پاک‌سازی عمومی Gateway منقضی‌شده در OpenClaw آشکار کرد: یک فرایند فرزند
می‌توانست برای پاک‌سازی فرایند والد Gateway تلاش کند و در مدیران فرایندی مانند
systemd باعث ایجاد چرخه‌های بازراه‌اندازی شود.

پاک‌سازی کنونی هنگام راه‌اندازی OpenClaw، فرایند فعلی و فرایندهای والد آن را
مستثنا می‌کند؛ بنابراین یک فرایند کمکی کانال نمی‌تواند Gateway راه‌انداز خود را
متوقف کند. این اصلاح عمومی است و مسیر ویژه WeChat در هسته نیست.

## عیب‌یابی

نصب و وضعیت را بررسی کنید:

```bash
openclaw plugins list
openclaw channels status --probe
openclaw --version
```

اگر کانال نصب‌شده نمایش داده می‌شود اما متصل نمی‌شود، مطمئن شوید Plugin فعال
است و آن را بازراه‌اندازی کنید:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
```

اگر Gateway پس از فعال‌سازی WeChat به‌طور مکرر بازراه‌اندازی می‌شود، هم OpenClaw
و هم Plugin را به‌روزرسانی کنید:

```bash
npm view @tencent-weixin/openclaw-weixin version
openclaw plugins install "@tencent-weixin/openclaw-weixin" --force
openclaw gateway restart
```

اگر هنگام راه‌اندازی گزارش شد که بسته Plugin نصب‌شده `requires compiled runtime
output for TypeScript entry`، بسته npm بدون فایل‌های کامپایل‌شده
زمان اجرای JavaScript موردنیاز OpenClaw منتشر شده است. پس از انتشار بسته
اصلاح‌شده توسط ناشر Plugin، آن را به‌روزرسانی یا دوباره نصب کنید؛ یا موقتاً
Plugin را غیرفعال یا حذف کنید.

غیرفعال‌سازی موقت:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled false
openclaw gateway restart
```

## مستندات مرتبط

- نمای کلی کانال: [کانال‌های گفت‌وگو](/fa/channels)
- جفت‌سازی: [جفت‌سازی](/fa/channels/pairing)
- مسیریابی کانال: [مسیریابی کانال](/fa/channels/channel-routing)
- معماری Plugin: [معماری Plugin](/fa/plugins/architecture)
- SDK مربوط به Plugin کانال: [SDK مربوط به Plugin کانال](/fa/plugins/sdk-channel-plugins)
- بسته خارجی: [@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
