---
read_when:
    - می‌خواهید یک عامل OpenClaw به جلسه Zoom بپیوندد
    - در حال پیکربندی Chrome، BlackHole یا SoX برای پاسخ‌گویی در جلسه Zoom هستید
summary: 'Plugin جلسات Zoom: پیوستن به جلسات به‌عنوان مهمان مرورگر Chrome'
title: Plugin جلسات Zoom
x-i18n:
    generated_at: "2026-07-27T14:28:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d91e57cccb163f634c6eaee71dd3832fc7b9e783fc5cd02601572b302d0d25e8
    source_path: plugins/zoom-meetings.md
    workflow: 16
---

Plugin `zoom-meetings` از طریق Zoom Web App در پروفایل Chrome متعلق به OpenClaw، به‌عنوان مهمان به پیوندهای جلسه Zoom می‌پیوندد. این Plugin پیوندهای جلسه در `zoom.us/j/...` و زیردامنه‌های حساب مانند `example.zoom.us/j/...` را می‌پذیرد. جلسه ایجاد نمی‌کند، با شماره‌گیری وارد نمی‌شود، از Zoom Meeting SDK استفاده نمی‌کند و فایل‌های ضبط‌شدهٔ صوتی/تصویری را ثبت نمی‌کند.

## راه‌اندازی

پاسخ‌گویی صوتی از همان پیش‌نیازهای صوتی محلی [Plugin مربوط به Google Meet](/fa/plugins/google-meet) استفاده می‌کند: macOS، دستگاه صوتی مجازی `BlackHole 2ch` و SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

این Plugin به‌صورت پیش‌فرض گنجانده و فعال شده است. فقط برای سفارشی‌سازی آن یک ورودی اضافه کنید، سپس راه‌اندازی را بررسی کنید:

```json5
{
  plugins: {
    entries: {
      "zoom-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

اگر نمی‌خواهید Plugin فعال باشد، `openclaw plugins disable zoom-meetings` را اجرا کنید.

```bash
openclaw zoommeetings setup
openclaw zoommeetings join 'https://zoom.us/j/1234567890'
```

برای اجرای Chrome، BlackHole و SoX روی یک Node جفت‌شدهٔ macOS، از `chromeNode.node` استفاده کنید. Node باید `zoommeetings.chrome` و `browser.proxy` را مجاز کند.

## حالت‌ها

| حالت         | رفتار                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | رونویسی بلادرنگ با عامل پیکربندی‌شدهٔ OpenClaw مشورت می‌کند؛ TTS پاسخ می‌دهد. |
| `bidi`       | یک مدل صوتی بلادرنگ مستقیماً گوش می‌دهد و پاسخ می‌دهد.                        |
| `transcribe` | پیوستن فقط برای مشاهده، همراه با نماهای لحظه‌ای رونویسی زیرنویس زنده.                   |

زیرنویس زندهٔ Zoom پس از پذیرش در همهٔ حالت‌ها فعال می‌شود تا OpenClaw بتواند
یادداشت‌های جلسه را ماندگار کند. کنش `transcript` همچنان بافر زندهٔ محدود را
فقط برای نشست‌های `transcribe` برمی‌گرداند. هنگام ترک جلسه، OpenClaw رونویسی
ماندگار و خلاصهٔ استخراج‌شده را در پایگاه دادهٔ وضعیت مشترک ذخیره می‌کند؛ آن‌ها را
با [`openclaw transcripts`](/fa/cli/transcripts) فهرست یا صادر کنید.

یادداشت‌های خودکار به‌صورت پیش‌فرض فعال‌اند. برای غیرفعال‌کردن یادداشت‌های ماندگار
در سطح سراسری، `transcripts.enabled: false` را تنظیم کنید؛ حالت صریح `transcribe`
همچنان فقط دنبالهٔ زندهٔ محدود خود را ارائه می‌دهد.

## محدودیت‌های پیوستن مهمان

آداپتور مرورگر **Join from browser** را انتخاب می‌کند، نام مهمان را وارد می‌کند، دوربین را خاموش می‌کند، میکروفون را برای حالت انتخاب‌شده پیکربندی می‌کند و روی **Join** کلیک می‌کند. Zoom Web App تحت `app.zoom.us` اجرا می‌شود؛ Plugin پیش از پیمایش، مجوزهای میکروفون و انتخاب بلندگو را به آن مبدأ اعطا می‌کند. وضعیت حین تماس از کنترل Leave در Zoom استفاده می‌کند. وضعیت‌های اتاق انتظار، ورود به سیستم، رمز عبور جلسه، CAPTCHA و مجوز دستگاه، دلایل صریحی برای اقدام دستی برمی‌گردانند.

خط‌مشی میزبان و حساب Zoom می‌تواند پیوستن از مرورگر را غیرفعال کند، احراز هویت یا تأیید ایمیل را الزامی کند، CAPTCHA نمایش دهد یا پذیرش میزبان را لازم بداند. آن مرحله را در پروفایل Chrome متعلق به OpenClaw تکمیل کنید، سپس وضعیت یا گفتار را دوباره امتحان کنید. این Plugin خط‌مشی Zoom را دور نمی‌زند.

Zoom Web App با یک جلسهٔ آزمایشی رسمی Zoom برای میان‌صفحهٔ برنامه، ورود نام مهمان در iframe، کنترل‌های میکروفون و دوربین پیش از پیوستن، پیوستن، مجوزهای رسانه‌ای مرورگر و macOS، تشخیص وضعیت حین تماس، فعال‌سازی زیرنویس زنده و تشخیص پایان جلسه توسط میزبان، به‌صورت زنده اعتبارسنجی شده است. وضعیت‌های اتاق انتظار و احراز هویت به خط‌مشی میزبان وابسته‌اند و هرگاه شناسهٔ پایدار DOM در دسترس نباشد، جایگزین‌های متنی را حفظ می‌کنند.

## سطح ابزار و Gateway

ابزار عامل `zoom_meetings` از `join`، `leave`، `status`، `transcript` و `speak` پشتیبانی می‌کند. متدهای Gateway از پیشوند `zoommeetings.*` استفاده می‌کنند. فرمان Node عبارت است از `zoommeetings.chrome`.

## مرتبط

- [نمای کلی Pluginهای جلسه](/plugins/meeting-plugins)
