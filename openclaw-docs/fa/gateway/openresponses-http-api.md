---
read_when:
    - یکپارچه‌سازی کلاینت‌هایی که با API ‏OpenResponses ارتباط برقرار می‌کنند
    - ورودی‌های مبتنی بر آیتم، فراخوانی‌های ابزار کلاینت یا رویدادهای SSE می‌خواهید
summary: یک نقطه پایانی HTTP سازگار با OpenResponses در مسیر /v1/responses از Gateway ارائه کنید
title: API پاسخ‌های باز
x-i18n:
    generated_at: "2026-07-27T16:33:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

Gateway می‌تواند یک نقطهٔ پایانی `POST /v1/responses` سازگار با OpenResponses ارائه کند. این قابلیت **به‌طور پیش‌فرض غیرفعال است** و پورت خود را با Gateway به اشتراک می‌گذارد (چندگانه‌سازی WS + HTTP): `http://<gateway-host>:<port>/v1/responses`.

درخواست‌ها مانند اجرای عادی عامل Gateway اجرا می‌شوند (همان مسیر کد `openclaw agent`)؛ بنابراین مسیریابی، مجوزها و پیکربندی با Gateway شما مطابقت دارند.

با `gateway.http.endpoints.responses.enabled` آن را فعال یا غیرفعال کنید. وقتی فعال باشد، همین سطح سازگاری همچنین `GET /v1/models`، `GET /v1/models/{id}`، `POST /v1/embeddings` و `POST /v1/chat/completions` را ارائه می‌کند.

## احراز هویت، امنیت و مسیریابی

رفتار عملیاتی با [تکمیل‌های گفت‌وگوی OpenAI](/fa/gateway/openai-http-api) مطابقت دارد:

- مسیر احراز هویت با `gateway.auth.mode` مطابقت دارد: حالت راز مشترک (`token`/`password`) از `Authorization: Bearer <token-or-password>` استفاده می‌کند؛ پراکسی مورداعتماد از سرآیندهای پراکسی آگاه از هویت استفاده می‌کند (پراکسی‌های لوپ‌بک روی همان میزبان به `gateway.auth.trustedProxy.allowLoopback = true` نیاز دارند و اگر هیچ‌یک از سرآیندهای `Forwarded`/`X-Forwarded-*`/`X-Real-IP` وجود نداشته باشد، بازگشت مستقیم روی همان میزبان از طریق `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` انجام می‌شود)؛ `none` در ورودی خصوصی به سرآیند احراز هویت نیاز ندارد. [احراز هویت پراکسی مورداعتماد](/fa/gateway/trusted-proxy-auth) را ببینید.
- این نقطهٔ پایانی را دارای دسترسی کامل اپراتور به نمونهٔ Gateway در نظر بگیرید.
- حالت‌های احراز هویت با راز مشترک، `x-openclaw-scopes` محدودتری را که در توکن حامل اعلام شده است نادیده می‌گیرند و مجموعهٔ کامل حوزه‌های پیش‌فرض اپراتور را بازیابی می‌کنند: `operator.admin`، `operator.approvals`، `operator.pairing`، `operator.read`، `operator.talk.secrets`، `operator.write`. نوبت‌های گفت‌وگو در این نقطهٔ پایانی به‌عنوان نوبت‌های فرستندهٔ مالک در نظر گرفته می‌شوند.
- حالت‌های HTTP مورداعتماد و حامل هویت (پراکسی مورداعتماد یا `gateway.auth.mode="none"`) در صورت وجود به `x-openclaw-scopes` احترام می‌گذارند؛ در غیر این صورت، به مجموعهٔ حوزه‌های پیش‌فرض اپراتور بازمی‌گردند. معنای مالک فقط زمانی از دست می‌رود که فراخواننده صراحتاً حوزه‌ها را محدود کند و `operator.admin` را حذف کند.
- عامل‌ها را با `model: "openclaw"`، `"openclaw/default"`، `"openclaw/<agentId>"` یا سرآیند `x-openclaw-agent-id` انتخاب کنید.
- برای بازنویسی مدل پشتیبان عامل انتخاب‌شده از `x-openclaw-model` استفاده کنید (در مسیرهای احراز هویت حامل هویت به `operator.admin` نیاز دارد).
- برای مسیریابی صریح نشست از `x-openclaw-session-key` استفاده کنید (اگر از فضای نام رزروشدهٔ `subagent:`، `cron:` یا `acp:` استفاده کند، با `400 invalid_request_error` رد می‌شود).
- برای زمینهٔ کانال ورودی مصنوعیِ غیراپیش‌فرض از `x-openclaw-message-channel` استفاده کنید.

برای توضیح مرجع دربارهٔ مدل‌های هدف عامل، `openclaw/default`، عبور مستقیم جاسازی‌ها و بازنویسی مدل پشتیبان، [تکمیل‌های گفت‌وگوی OpenAI](/fa/gateway/openai-http-api#agent-first-model-contract) را ببینید.

[حوزه‌های اپراتور](/fa/gateway/operator-scopes) و [امنیت](/fa/gateway/security) را ببینید.

## رفتار نشست

به‌طور پیش‌فرض، نقطهٔ پایانی **برای هر درخواست بدون حالت است** (در هر فراخوانی یک کلید نشست جدید ایجاد می‌شود).

اگر درخواست شامل رشتهٔ OpenResponses با نام `user` باشد، Gateway یک کلید نشست پایدار از آن استخراج می‌کند تا فراخوانی‌های تکراری بتوانند یک نشست عامل را به اشتراک بگذارند.

`previous_response_id` هنگامی نشست پاسخ قبلی را دوباره استفاده می‌کند که درخواست در همان محدودهٔ عامل/کاربر/نشست درخواستی باقی بماند (بر اساس موضوع احراز هویت، شناسهٔ عامل و `x-openclaw-session-key` تطبیق داده می‌شود).

## ساختار درخواست

| فیلد                                                            | پشتیبانی                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | رشته یا آرایه‌ای از اشیای آیتم.                                                                                               |
| `instructions`                                                   | با اعلان سیستم ادغام می‌شود.                                                                                                 |
| `tools`                                                          | تعریف ابزارهای کلاینت (ابزارهای تابع).                                                                                      |
| `tool_choice`                                                    | `"auto"`، `"none"`، `"required"` یا `{ "type": "function", "name": "..." }` برای پالایش یا الزامی‌کردن ابزارهای کلاینت.                |
| `stream`                                                         | جریان SSE را فعال می‌کند.                                                                                                         |
| `max_output_tokens`                                              | محدودیت خروجی با بیشترین تلاش ممکن (وابسته به ارائه‌دهنده).                                                                                 |
| `temperature`                                                    | دمای نمونه‌گیری با بیشترین تلاش ممکن. توسط پشتیبان Codex Responses مبتنی بر ChatGPT نادیده گرفته می‌شود، زیرا از نمونه‌گیری ثابت سمت سرور استفاده می‌کند. |
| `top_p`                                                          | نمونه‌گیری هسته‌ای با بیشترین تلاش ممکن. همان ملاحظهٔ Codex Responses مربوط به `temperature`.                                                    |
| `user`                                                           | مسیریابی پایدار نشست.                                                                                                        |
| `previous_response_id`                                           | تداوم نشست (بالا را ببینید).                                                                                                |
| `max_tool_calls`، `reasoning`، `metadata`، `store`، `truncation` | پذیرفته می‌شوند، اما در حال حاضر نادیده گرفته می‌شوند.                                                                                                |

## آیتم‌ها (ورودی)

### `message`

نقش‌ها: `system`، `developer`، `user`، `assistant`.

- `system` و `developer` به اعلان سیستم افزوده می‌شوند.
- جدیدترین آیتم `user` یا `function_call_output` به «پیام جاری» تبدیل می‌شود.
- پیام‌های پیشین کاربر/دستیار به‌عنوان تاریخچه برای زمینه گنجانده می‌شوند.

### `function_call_output` (ابزارهای مبتنی بر نوبت)

نتایج ابزار را به مدل بازگردانید:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` و `item_reference`

برای سازگاری طرح‌واره پذیرفته می‌شوند، اما هنگام ساخت اعلان نادیده گرفته می‌شوند.

## ابزارها (ابزارهای تابع سمت کلاینت)

ابزارها را با `tools: [{ type: "function", name, description?, parameters? }]` ارائه کنید.

اگر عامل ابزاری را فراخوانی کند، پاسخ یک آیتم خروجی `function_call` برمی‌گرداند. برای ادامهٔ نوبت، یک درخواست پیگیری با `function_call_output` ارسال کنید.

برای `tool_choice: "required"` و `tool_choice` سنجاق‌شده به تابع، نقطهٔ پایانی مجموعهٔ ابزارهای تابع کلاینتِ در معرض دسترس را محدود می‌کند، به زمان اجرا دستور می‌دهد پیش از پاسخ‌دادن یک ابزار کلاینت را فراخوانی کند و اگر نوبت شامل فراخوانی ساخت‌یافتهٔ منطبق با ابزار کلاینت نباشد، آن را مطابق قرارداد `/v1/chat/completions` رد می‌کند. درخواست‌های غیرجریانی `502` را همراه با `api_error` برمی‌گردانند؛ درخواست‌های جریانی یک رویداد `response.failed` منتشر می‌کنند.

## تصاویر (`input_image`)

از منابع base64 یا URL پشتیبانی می‌کند:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

انواع MIME مجاز (پیش‌فرض): `image/jpeg`، `image/png`، `image/gif`، `image/webp`، `image/heic`، `image/heif`. حداکثر اندازه (پیش‌فرض): 10MB.

## فایل‌ها (`input_file`)

از منابع base64 یا URL پشتیبانی می‌کند:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

انواع MIME مجاز (پیش‌فرض): `text/plain`، `text/markdown`، `text/html`، `text/csv`، `application/json`، `application/pdf`. حداکثر اندازه (پیش‌فرض): 5MB.

رفتار فعلی:

- محتوای فایل رمزگشایی و به **اعلان سیستم** افزوده می‌شود، نه پیام کاربر؛ بنابراین گذرا باقی می‌ماند (در تاریخچهٔ نشست ذخیره نمی‌شود).
- متن رمزگشایی‌شدهٔ فایل پیش از افزوده‌شدن به‌عنوان **محتوای خارجی نامطمئن** محصور می‌شود؛ بنابراین بایت‌های فایل به‌عنوان داده در نظر گرفته می‌شوند، نه دستورالعمل‌های مورداعتماد. بلوک تزریق‌شده از نشانگرهای مرزی صریح (`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`) و یک خط فرادادهٔ `Source: External` استفاده می‌کند. برای حفظ بودجهٔ اعلان، بنر طولانی `SECURITY NOTICE:` عمداً حذف می‌شود؛ نشانگرهای مرزی و فراداده همچنان اعمال می‌شوند.
- ابتدا PDFها برای استخراج متن تجزیه می‌شوند. اگر متن کمی یافت شود، صفحه‌های نخست به تصاویر شطرنجی تبدیل و به مدل ارسال می‌شوند و بلوک فایل تزریق‌شده از جای‌نگهدار `[PDF content rendered to images]` استفاده می‌کند.

تجزیهٔ PDF را Plugin همراه `document-extract` فراهم می‌کند که برای استخراج متن و رندر صفحه از `clawpdf` و زمان اجرای بسته‌بندی‌شدهٔ PDFium WebAssembly آن استفاده می‌کند.

پیش‌فرض‌های واکشی URL:

- `files.allowUrl`: `true`
- `images.allowUrl`: `true`
- `maxUrlParts`: `8` (مجموع بخش‌های مبتنی بر URL از نوع `input_file` + `input_image` در هر درخواست)
- درخواست‌ها محافظت می‌شوند (تفکیک DNS، مسدودسازی IP خصوصی، محدودیت تغییرمسیرها و مهلت‌های زمانی).
- فهرست‌های اختیاری میزبان‌های مجاز برای هر نوع ورودی (`files.urlAllowlist`، `images.urlAllowlist`) پشتیبانی می‌شوند: میزبان دقیق (`"cdn.example.com"`) یا زیردامنه‌های عام (`"*.assets.example.com"`، با دامنهٔ ریشه مطابقت ندارد). فهرست مجاز خالی یا حذف‌شده به‌معنای نبود محدودیت فهرست مجاز میزبان است.
- برای غیرفعال‌کردن کامل واکشی‌های مبتنی بر URL، `files.allowUrl: false` و/یا `images.allowUrl: false` را تنظیم کنید.

## محدودیت‌های فایل و تصویر

این نقطهٔ پایانی از محدودیت داخلی 20 MB برای بدنهٔ درخواست استفاده می‌کند. سیاست منبع فایل و تصویر
همچنان در `gateway.http.endpoints.responses` قابل پیکربندی است:

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

مقادیر پیش‌فرض در صورت حذف:

| کلید                      | پیش‌فرض   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

منابع HEIC/HEIF `input_image` پیش از تحویل به ارائه‌دهنده، از طریق پردازشگر مشترک تصویر OpenClaw ‏(Rastermill) به JPEG تبدیل و یکسان‌سازی می‌شوند. این پردازشگر برای قالب‌هایی که به پشتیبانی کدک خارجی نیاز دارند، از یک مبدل سیستمی (`sips`، ImageMagick، GraphicsMagick یا ffmpeg) به‌عنوان راهکار جایگزین استفاده می‌کند.

نکته امنیتی: فهرست‌های مجاز URL پیش از واکشی و در هر مرحله از تغییرمسیر اعمال می‌شوند. قرار دادن نام میزبان در فهرست مجاز، مسدودسازی IPهای خصوصی/داخلی را دور نمی‌زند. برای Gatewayهایی که در معرض اینترنت هستند، علاوه بر محافظ‌های سطح برنامه، کنترل‌های خروجی شبکه را نیز اعمال کنید. به [امنیت](/fa/gateway/security) مراجعه کنید.

## استریم (SSE)

برای دریافت رویدادهای ارسال‌شده از سرور، `stream: true` را تنظیم کنید:

- `Content-Type: text/event-stream`
- هر خط رویداد `event: <type>` و `data: <json>` است
- استریم با `data: [DONE]` پایان می‌یابد

انواع رویدادهایی که در حال حاضر منتشر می‌شوند: `response.created`، `response.in_progress`، `response.output_item.added`، `response.content_part.added`، `response.output_text.delta`، `response.output_text.done`، `response.content_part.done`، `response.output_item.done`، `response.completed`، `response.failed` (هنگام بروز خطا).

## میزان استفاده

هنگامی که ارائه‌دهنده زیربنایی تعداد توکن‌ها را گزارش کند، `usage` مقداردهی می‌شود. OpenClaw نام‌های مستعار رایج به سبک OpenAI، از جمله `input_tokens` / `output_tokens` و `prompt_tokens` / `completion_tokens` را پیش از رسیدن این شمارنده‌ها به سطوح وضعیت/نشست پایین‌دستی یکسان‌سازی می‌کند.

## خطاها

خطاها از یک شیء JSON مانند زیر استفاده می‌کنند:

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

موارد رایج: `400` بدنه درخواست نامعتبر، `401` احراز هویت مفقود/نامعتبر، `403` محدوده اپراتور مفقود، `405` متد نادرست، `429` تلاش‌های ناموفق بیش‌ازحد برای احراز هویت (همراه با `Retry-After`).

## مثال‌ها

بدون استریم:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

با استریم:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## مرتبط

- [تکمیل‌های گفت‌وگوی OpenAI](/fa/gateway/openai-http-api)
- [محدوده‌های اپراتور](/fa/gateway/operator-scopes)
- [OpenAI](/fa/providers/openai)
