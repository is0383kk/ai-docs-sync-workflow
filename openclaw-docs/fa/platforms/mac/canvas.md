---
read_when:
    - پیاده‌سازی پنل Canvas در macOS
    - افزودن کنترل‌های عامل برای فضای کاری بصری
    - اشکال‌زدایی بارگذاری‌های canvas در WKWebView
summary: پنل Canvas کنترل‌شده توسط عامل، تعبیه‌شده از طریق WKWebView و طرح URL سفارشی
title: بوم
x-i18n:
    generated_at: "2026-07-27T15:48:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 56532246bc06601aa753a59f85f33bfa8d6599deecade591a03972e8b9b16fc2
    source_path: platforms/mac/canvas.md
    workflow: 16
---

برنامه macOS یک **پنل Canvas** تحت کنترل عامل را با استفاده از `WKWebView` تعبیه می‌کند؛
فضای کاری بصری سبکی برای HTML/CSS/JS،‏ A2UI و سطوح کوچک رابط کاربری
تعاملی.

## محل قرارگیری Canvas

وضعیت Canvas در Application Support ذخیره می‌شود:

- `~/Library/Application Support/OpenClaw/canvas/<session>/...`

پنل Canvas آن فایل‌ها را از طریق یک طرح‌واره URL سفارشی،
`openclaw-canvas://<session>/<path>`، ارائه می‌کند:

- `openclaw-canvas://main/` -> `<canvasRoot>/main/index.html`
- `openclaw-canvas://main/assets/app.css` -> `<canvasRoot>/main/assets/app.css`
- `openclaw-canvas://main/widgets/todo/` -> `<canvasRoot>/main/widgets/todo/index.html`

اگر هیچ `index.html` در ریشه وجود نداشته باشد، برنامه یک صفحه داربست داخلی نمایش می‌دهد.

## رفتار پنل

- پنلی بدون حاشیه و با اندازه قابل تغییر که نزدیک نوار منو (یا نشانگر ماوس) جای می‌گیرد.
- نمایش Canvas باعث جابه‌جایی بین برنامه‌ها یا گرفتن تمرکز صفحه‌کلید نمی‌شود.
- اندازه/موقعیت را برای هر نشست به خاطر می‌سپارد.
- با تغییر فایل‌های محلی Canvas، به‌طور خودکار بارگذاری مجدد می‌شود.
- در هر لحظه فقط یک پنل Canvas قابل مشاهده است (در صورت نیاز، نشست تغییر می‌کند).

Canvas را می‌توان از Settings -> **Allow Canvas** غیرفعال کرد. در حالت غیرفعال،
فرمان‌های نود Canvas مقدار `CANVAS_DISABLED` را برمی‌گردانند.

## سطح API عامل

Canvas از طریق WebSocket مربوط به Gateway در دسترس قرار می‌گیرد؛ بنابراین عامل می‌تواند
پنل را نمایش دهد یا پنهان کند، به یک مسیر یا URL برود، JavaScript را ارزیابی کند و
یک تصویر لحظه‌ای ثبت کند:

```bash
openclaw nodes canvas present --node <id>
openclaw nodes canvas navigate --node <id> "/"
openclaw nodes canvas eval --node <id> --js "document.title"
openclaw nodes canvas snapshot --node <id>
```

`eval` و `a2ui.*` محتوا را بدون باز کردن یا آشکار کردن پنل به‌روزرسانی می‌کنند. فقط
`present`،‏ `navigate` یا اقدام کاربر آن را نمایش می‌دهد؛ پس از پنهان‌سازی، به‌روزرسانی‌های محتوا
همچنان روی پنل پنهان اعمال می‌شوند. `snapshot` به پنلی قابل مشاهده نیاز دارد و
در غیر این صورت `CANVAS_HIDDEN` را برمی‌گرداند؛ ابتدا `present` را اجرا کنید.

`canvas.navigate` مسیرهای محلی Canvas،‏ URLهای `http(s)` و URLهای `file://` را
می‌پذیرد. ارسال `"/"` داربست محلی یا `index.html` را نمایش می‌دهد.

مقصدهای میزبانی‌شده توسط Gateway در `/__openclaw__/canvas/` و
`/__openclaw__/a2ui/` از طریق URL محدوده‌بندی‌شده فعلی Canvas در نشست نود
تفکیک می‌شوند. برنامه این قابلیت کوتاه‌عمر را پیش از پیمایش تازه‌سازی می‌کند؛
لازم نیست URL قابلیت را خودتان بسازید یا کپی کنید.

## A2UI در Canvas

A2UI توسط میزبان Canvas در Gateway میزبانی و درون پنل Canvas
رندر می‌شود. وقتی Gateway وجود میزبان Canvas را اعلام می‌کند، برنامه macOS هنگام نخستین باز شدن
به‌طور خودکار به صفحه میزبان A2UI می‌رود.

URL اعلام‌شده به قابلیت محدود است؛ برای مثال
`http://<gateway-host>:18789/__openclaw__/cap/<token>/__openclaw__/a2ui/?platform=macos`.
آن را اعتبارنامه‌ای موقت در نظر بگیرید، نه پیوندی پایدار.

### فرمان‌های A2UI (v0.8)

Canvas پیام‌های سرور به کلاینت A2UI v0.8 را می‌پذیرد: `beginRendering`،
`surfaceUpdate`،‏ `dataModelUpdate`،‏ `deleteSurface`. ‏`createSurface` (v0.9)
هنوز پشتیبانی نمی‌شود.

```bash
cat > /tmp/a2ui-v0.8.jsonl <<'EOFA2'
{"surfaceUpdate":{"surfaceId":"main","components":[{"id":"root","component":{"Column":{"children":{"explicitList":["title","content"]}}}},{"id":"title","component":{"Text":{"text":{"literalString":"اگر می‌توانید این متن را بخوانید، ارسال A2UI کار می‌کند."},"usageHint":"h1"}}},{"id":"content","component":{"Text":{"text":{"literalString":"اگر می‌توانید این متن را بخوانید، ارسال A2UI کار می‌کند."},"usageHint":"body"}}}]}}
{"beginRendering":{"surfaceId":"main","root":"root"}}
EOFA2

openclaw nodes canvas a2ui push --jsonl /tmp/a2ui-v0.8.jsonl --node <id>
```

آزمون سریع اولیه:

```bash
openclaw nodes canvas a2ui push --node <id> --text "سلام از A2UI"
```

## راه‌اندازی اجرای عامل از Canvas

Canvas می‌تواند از طریق پیوندهای عمیق `openclaw://agent?...` اجرای جدید عامل را راه‌اندازی کند:

```js
window.location.href = "openclaw://agent?message=Review%20this%20design";
```

پارامترهای پرس‌وجوی پشتیبانی‌شده:

| پارامتر                    | معنا                                                  |
| -------------------------- | ----------------------------------------------------- |
| `message`                  | اعلان ازپیش‌پرشده عامل.                               |
| `sessionKey`               | شناسه پایدار نشست.                                    |
| `thinking`                 | پروفایل اختیاری تفکر.                                 |
| `deliver`، `to`، `channel` | مقصد تحویل.                                           |
| `timeoutSeconds`           | مهلت زمانی اختیاری اجرا.                              |
| `key`                      | توکن ایمنی تولیدشده توسط برنامه برای فراخوان‌های محلی مورد اعتماد. |

برنامه، مگر در صورت ارائه کلیدی معتبر، درخواست تأیید می‌کند. پیوندهای
بدون کلید، پیام و URL را پیش از تأیید نمایش می‌دهند و فیلدهای مسیریابی تحویل را
نادیده می‌گیرند؛ پیوندهای کلیددار از مسیر عادی اجرای Gateway استفاده می‌کنند.

## نکات امنیتی

- طرح‌واره Canvas پیمایش دایرکتوری را مسدود می‌کند؛ فایل‌ها باید زیر ریشه نشست قرار داشته باشند.
- محتوای محلی Canvas از طرح‌واره‌ای سفارشی استفاده می‌کند (به سرور loopback نیازی نیست).
- URLهای خارجی `http(s)` فقط زمانی مجاز هستند که صراحتاً به آن‌ها پیمایش شود.
- صفحه‌های وب عادی فقط قابل رندر هستند. اقدام‌های عامل تنها از طرح‌واره Canvas
  متعلق به برنامه یا سند دقیق A2UI در Gateway که به قابلیت محدود شده و برنامه آن را
  انتخاب کرده است پذیرفته می‌شوند؛ فریم‌های فرعی، تغییرمسیرها، قابلیت‌های منقضی‌شده و پرس‌وجوهای
  تغییریافته نمی‌توانند اقدام‌ها را ارسال کنند.

## مرتبط

- [برنامه macOS](/fa/platforms/macos)
- [WebChat](/fa/web/webchat)
