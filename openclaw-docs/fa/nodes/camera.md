---
read_when:
    - افزودن یا تغییر تصویربرداری با دوربین در پلتفرم‌های Node
    - گسترش جریان‌های کاری فایل موقت MEDIA قابل‌دسترسی برای عامل‌ها
summary: ثبت تصویر با دوربین در Nodeهای iOS، Android، macOS و Linux برای عکس‌ها و کلیپ‌های ویدیویی کوتاه
title: ثبت تصویر با دوربین
x-i18n:
    generated_at: "2026-07-27T14:17:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b819f7ff3fc9b51757ae998d27f540975bf6c1194ed32fd36b1fbe909e79400c
    source_path: nodes/camera.md
    workflow: 16
---

OpenClaw از ثبت تصویر دوربین برای گردش‌کارهای عامل در Nodeهای جفت‌شدهٔ **iOS**، **Android**، **macOS** و **Linux** پشتیبانی می‌کند: از طریق Gateway `node.invoke` یک عکس (`jpg`) یا کلیپ ویدیویی کوتاه (`mp4`، با صدای اختیاری) ثبت کنید.

تمام دسترسی‌های دوربین در هر پلتفرم پشت یک تنظیم تحت کنترل کاربر قرار دارند.

## Node در iOS

### تنظیم کاربر در iOS

- زبانهٔ iOS Settings → **Camera** → **Allow Camera** (`camera.enabled`).
  - پیش‌فرض: **روشن** (نبود کلید به‌معنای فعال‌بودن در نظر گرفته می‌شود).
  - در حالت خاموش: فرمان‌های `camera.*` مقدار `CAMERA_DISABLED` را برمی‌گردانند.

### فرمان‌های iOS (از طریق Gateway `node.invoke`)

- `camera.list`
  - بار پاسخ: `devices` — آرایه‌ای از `{ id, name, position, deviceType }`.

- `camera.snap`
  - پارامترها:
    - `facing`: `front|back` (پیش‌فرض: `front`)
    - `maxWidth`: عدد (اختیاری؛ پیش‌فرض `1600`)
    - `quality`: `0..1` (اختیاری؛ پیش‌فرض `0.9`، محدودشده به `[0.05, 1.0]`)
    - `format`: در حال حاضر `jpg`
    - `delayMs`: عدد (اختیاری؛ پیش‌فرض `0`، با سقف داخلی `10000`)
    - `deviceId`: رشته (اختیاری؛ از `camera.list`)
  - بار پاسخ: `format: "jpg"`، `base64`، `width`، `height`.
  - محافظ بار: عکس‌ها دوباره فشرده می‌شوند تا بار کدگذاری‌شده با base64 زیر 5MB باقی بماند.

- `camera.clip`
  - پارامترها:
    - `facing`: `front|back` (پیش‌فرض: `front`)
    - `durationMs`: عدد (پیش‌فرض `3000`، محدودشده به `[250, 60000]`)
    - `includeAudio`: بولی (پیش‌فرض `true`)
    - `format`: در حال حاضر `mp4`
    - `deviceId`: رشته (اختیاری؛ از `camera.list`)
  - بار پاسخ: `format: "mp4"`، `base64`، `durationMs`، `hasAudio`.

### الزام پیش‌زمینه در iOS

مانند `canvas.*`، Node در iOS فرمان‌های `camera.*` را فقط در **پیش‌زمینه** مجاز می‌داند. فراخوانی‌های پس‌زمینه `NODE_BACKGROUND_UNAVAILABLE` را برمی‌گردانند.

### ابزار کمکی CLI

ساده‌ترین راه برای دریافت فایل‌های رسانه‌ای، استفاده از ابزار کمکی CLI است که رسانهٔ رمزگشایی‌شده را در یک فایل موقت می‌نویسد و مسیر ذخیره‌شده را چاپ می‌کند.

```bash
openclaw nodes camera snap --node <id>                 # پیش‌فرض: هر دو دوربین جلو و پشت (2 خط MEDIA)
openclaw nodes camera snap --node <id> --facing front
openclaw nodes camera clip --node <id> --duration 3000
openclaw nodes camera clip --node <id> --no-audio
```

`nodes camera snap` به‌طور پیش‌فرض `--facing both` است و هر دو نمای جلو و پشت را ثبت می‌کند تا عامل به هر دو نما دسترسی داشته باشد؛ `--device-id` را با یک جهت صریح واحد ارسال کنید (`both` هنگام تنظیم `--device-id` رد می‌شود). فایل‌های خروجی موقت هستند (در پوشهٔ موقت سیستم‌عامل)، مگر اینکه پوشش اختصاصی خود را بسازید.

## Node در Android

### تنظیم کاربر در Android

- صفحهٔ Android Settings → **Camera** → **Allow Camera** (`camera.enabled`).
  - **در نصب‌های جدید، پیش‌فرض خاموش است.** نصب‌های موجودی که پیش از معرفی این تنظیم انجام شده‌اند به حالت **روشن** منتقل می‌شوند تا ارتقا باعث ازدست‌رفتن بی‌سروصدای دسترسی دوربینی که قبلاً کار می‌کرد نشود.
  - در حالت خاموش: فرمان‌های `camera.*` مقدار `CAMERA_DISABLED: enable Camera in Settings` را برمی‌گردانند.

### مجوزها

- `CAMERA` برای هر دو `camera.snap` و `camera.clip` الزامی است؛ نبودن یا ردشدن مجوز باعث بازگشت `CAMERA_PERMISSION_REQUIRED` می‌شود.
- `RECORD_AUDIO` برای `camera.clip` هنگامی الزامی است که `includeAudio` برابر با `true` باشد؛ نبودن یا ردشدن مجوز باعث بازگشت `MIC_PERMISSION_REQUIRED` می‌شود.

برنامه در صورت امکان، مجوزهای زمان اجرا را درخواست می‌کند.

### الزام پیش‌زمینه در Android

مانند `canvas.*`، Node در Android فرمان‌های `camera.*` را فقط در **پیش‌زمینه** مجاز می‌داند. فراخوانی‌های پس‌زمینه `NODE_BACKGROUND_UNAVAILABLE: command requires foreground` را برمی‌گردانند.

### فرمان‌های Android (از طریق Gateway `node.invoke`)

- `camera.list`
  - بار پاسخ: `devices` — آرایه‌ای از `{ id, name, position, deviceType }`.

- `camera.snap`
  - پارامترها: `facing` (`front|back`، پیش‌فرض `front`)، `quality` (پیش‌فرض `0.95`، محدودشده به `[0.1, 1.0]`)، `maxWidth` (پیش‌فرض `1600`)، `deviceId` (اختیاری؛ شناسهٔ ناشناخته با `INVALID_REQUEST` ناموفق می‌شود).
  - بار پاسخ: `format: "jpg"`، `base64`، `width`، `height`.
  - محافظ بار: برای نگه‌داشتن base64 زیر 5MB دوباره فشرده می‌شود (همان بودجهٔ iOS).

- `camera.clip`
  - پارامترها: `facing` (پیش‌فرض `front`)، `durationMs` (پیش‌فرض `3000`، محدودشده به `[200, 60000]`)، `includeAudio` (پیش‌فرض `true`)، `deviceId` (اختیاری).
  - بار پاسخ: `format: "mp4"`، `base64`، `durationMs`، `hasAudio`.
  - محافظ بار: حجم MP4 خام پیش از کدگذاری base64 به 18MB محدود است؛ کلیپ‌های بزرگ‌تر با `PAYLOAD_TOO_LARGE` ناموفق می‌شوند (`durationMs` را کاهش دهید و دوباره تلاش کنید).

## برنامهٔ macOS

### تنظیم کاربر در macOS

برنامهٔ همراه macOS یک کادر انتخاب ارائه می‌کند:

- **Settings → General → Allow Camera** (`openclaw.cameraEnabled`).
  - پیش‌فرض: **خاموش**.
  - در حالت خاموش: درخواست‌های دوربین `CAMERA_DISABLED: enable Camera in Settings` را برمی‌گردانند.

### ابزار کمکی CLI (فراخوانی Node)

برای فراخوانی فرمان‌های دوربین در Node مربوط به macOS، از CLI اصلی `openclaw` استفاده کنید.

```bash
openclaw nodes camera list --node <id>                     # فهرست شناسه‌های دوربین
openclaw nodes camera snap --node <id>                     # مسیر ذخیره‌شده را چاپ می‌کند
openclaw nodes camera snap --node <id> --max-width 1280
openclaw nodes camera snap --node <id> --delay-ms 2000
openclaw nodes camera snap --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --duration 10s       # مسیر ذخیره‌شده را چاپ می‌کند
openclaw nodes camera clip --node <id> --duration-ms 3000   # مسیر ذخیره‌شده را چاپ می‌کند (پرچم قدیمی)
openclaw nodes camera clip --node <id> --device-id <id>
openclaw nodes camera clip --node <id> --no-audio
```

- `openclaw nodes camera snap` به‌طور پیش‌فرض `maxWidth=1600` است، مگر اینکه بازنویسی شود.
- `camera.snap` پس از گرم‌شدن و تثبیت نوردهی، پیش از ثبت به‌مدت `delayMs` (پیش‌فرض 2000ms، محدودشده به `[0, 10000]`) منتظر می‌ماند.
- بارهای عکس دوباره فشرده می‌شوند تا base64 زیر 5MB باقی بماند.

## میزبان Node در Linux

Plugin همراه Linux Node، ثبت تصویر دوربین را به سرویس CLI ‏`openclaw node` اضافه می‌کند. این قابلیت روی میزبان بدون رابط گرافیکی کار می‌کند و به برنامهٔ دسکتاپ Linux نیازی ندارد.

دسترسی دوربین به‌طور پیش‌فرض خاموش است. آن را زیر ورودی Plugin فعال کنید، سپس سرویس Node را راه‌اندازی مجدد کنید تا اعلان Gateway آن دوباره ساخته شود:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          camera: { enabled: true },
        },
      },
    },
  },
}
```

الزامات:

- FFmpeg با ورودی V4L2، ‏`libx264` و پشتیبانی AAC
- یک دستگاه `/dev/video*` که برای کاربر سرویس Node قابل خواندن باشد؛ در توزیع‌های رایج، آن کاربر را به گروه `video` اضافه کنید
- برای کلیپ‌هایی با `includeAudio: true` پیش‌فرض، یک سرور PulseAudio فعال یا لایهٔ سازگاری PulseAudio در PipeWire با یک منبع پیش‌فرض

Linux مسیرهای دستگاه V4L2 خواندنی و دارای قابلیت ثبت را از `camera.list` برمی‌گرداند؛ FFmpeg هر نامزد `/dev/video*` را بررسی می‌کند و Nodeهای فاقد فراداده یا صرفاً خروجی را کنار می‌گذارد. مقدار `position` دستگاه برابر با `unknown` است، بنابراین درخواست‌های جهت بدون `deviceId` به‌جای ادعای وجود دوربین جلو یا پشت، یک عکس یا کلیپ با موقعیت `unknown` تولید می‌کنند. وقتی میزبانی چند دوربین دارد، از `deviceId` استفاده کنید. `camera.snap` برای `delayMs` از گرم‌سازی ورودی FFmpeg استفاده می‌کند و ضمن محدودکردن عرض، نسبت ابعاد را حفظ می‌کند. `camera.clip` صدای میکروفون را به‌عنوان ترک صوتی MP4 ضبط می‌کند؛ OpenClaw عمداً هیچ فرمان مستقل میکروفونی ارائه نمی‌دهد.

این Plugin برای ویدیوی MP4 از `libx264` استفاده می‌کند و کدک‌ها را بی‌سروصدا تغییر نمی‌دهد. یک ساخت FFmpeg بدون ورودی یا رمزگذارهای موردنیاز، `CAMERA_UNAVAILABLE` را برمی‌گرداند. عکس‌ها و کلیپ‌هایی که از بودجهٔ بار base64 برابر با 25MB فراتر بروند، با `PAYLOAD_TOO_LARGE` ناموفق می‌شوند.

`camera.snap` و `camera.clip` همچنان فرمان‌هایی خطرناک هستند. فقط هنگامی آن‌ها را به `gateway.nodes.commands.allow` اضافه کنید که قصد فعال‌کردن ثبت را دارید؛ فعال‌کردن Plugin به‌تنهایی سیاست Gateway را دور نمی‌زند.

## ایمنی و محدودیت‌های عملی

- دسترسی به دوربین و میکروفون درخواست‌های معمول مجوز سیستم‌عامل را فعال می‌کند (و به رشته‌های کاربرد در `Info.plist` نیاز دارد).
- کلیپ‌های ویدیویی برای جلوگیری از بارهای بیش‌ازحد بزرگ Node به 60s محدود شده‌اند (سربار base64 به‌علاوهٔ محدودیت‌های پیام).

## ویدیوی صفحه‌نمایش در macOS (سطح سیستم‌عامل)

برای ویدیوی _صفحه‌نمایش_ (نه دوربین)، از برنامهٔ همراه macOS استفاده کنید:

```bash
openclaw nodes screen record --node <id> --duration 10s --fps 15   # مسیر ذخیره‌شده را چاپ می‌کند
```

به مجوز **Screen Recording** در macOS ‏(TCC) نیاز دارد.

## مرتبط

- [پشتیبانی از تصویر و رسانه](/fa/nodes/images)
- [درک رسانه](/fa/nodes/media-understanding)
- [فرمان موقعیت مکانی](/fa/nodes/location-command)
