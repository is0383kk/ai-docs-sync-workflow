---
read_when:
    - می‌خواهید یک عامل OpenClaw به جلسه Microsoft Teams بپیوندد
    - در حال پیکربندی Chrome،‏ BlackHole یا SoX برای پاسخ‌گویی صوتی در جلسه Microsoft Teams هستید
summary: 'Plugin جلسات Microsoft Teams: پیوستن به جلسات کاری یا مصرف‌کنندگان به‌عنوان مهمان مرورگر Chrome'
title: Plugin جلسات Microsoft Teams
x-i18n:
    generated_at: "2026-07-27T14:27:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f84e58d478185d026dd79a02a8500af48f51689ef6865d56badb0e27c6d2814
    source_path: plugins/teams-meetings.md
    workflow: 16
---

Plugin ‏`teams-meetings` به‌عنوان مهمان و با استفاده از نمایه Chrome متعلق به OpenClaw به پیوندهای Microsoft Teams می‌پیوندد. این Plugin پیوندهای کاری زیر `teams.microsoft.com/l/meetup-join/...` و پیوندهای مصرف‌کننده زیر `teams.live.com/meet/...` را می‌پذیرد. این Plugin جلسه ایجاد نمی‌کند، با شماره تلفن به جلسه متصل نمی‌شود، Microsoft Graph را فراخوانی نمی‌کند و صدای یا ویدیوی جلسه را ضبط نمی‌کند.

## راه‌اندازی

پاسخ صوتی از همان پیش‌نیازهای صوتی محلی [Plugin ‏Google Meet](/fa/plugins/google-meet) استفاده می‌کند: macOS، دستگاه صوتی مجازی `BlackHole 2ch` و SoX.

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

این Plugin به‌طور پیش‌فرض گنجانده و فعال شده است. فقط برای سفارشی‌سازی آن یک ورودی اضافه کنید، سپس راه‌اندازی را بررسی کنید:

```json5
{
  plugins: {
    entries: {
      "teams-meetings": {
        config: {
          defaultMode: "agent",
          chrome: { guestName: "OpenClaw Agent" },
        },
      },
    },
  },
}
```

اگر نمی‌خواهید Plugin فعال باشد، `openclaw plugins disable teams-meetings` را اجرا کنید.

```bash
openclaw teamsmeetings setup
openclaw teamsmeetings join 'https://teams.microsoft.com/l/meetup-join/...'
```

برای اجرای Chrome،‏ BlackHole و SoX روی یک Node جفت‌شده macOS، از `chromeNode.node` استفاده کنید. Node باید `teamsmeetings.chrome` و `browser.proxy` را مجاز کند.

## حالت‌ها

| حالت         | رفتار                                                                    |
| ------------ | --------------------------------------------------------------------------- |
| `agent`      | رونویسی بلادرنگ با عامل پیکربندی‌شده OpenClaw مشورت می‌کند؛ TTS پاسخ می‌دهد. |
| `bidi`       | یک مدل صوتی بلادرنگ مستقیماً گوش می‌دهد و پاسخ می‌دهد.                        |
| `transcribe` | پیوستن فقط برای مشاهده، همراه با عکس‌های فوری از رونویسی زیرنویس زنده.                   |

زیرنویس زنده Teams پس از پذیرش در همه حالت‌ها فعال می‌شود تا OpenClaw بتواند
یادداشت‌های منتسب به گوینده را نگه‌داری کند. کنش `transcript` همچنان فقط برای
نشست‌های `transcribe` بافر زنده محدود را برمی‌گرداند. هنگام خروج، OpenClaw
رونویسی ماندگار و خلاصه استخراج‌شده را در پایگاه داده وضعیت مشترک ذخیره می‌کند؛ آن‌ها را با
[`openclaw transcripts`](/fa/cli/transcripts) فهرست یا صادر کنید.

یادداشت‌های خودکار به‌طور پیش‌فرض فعال‌اند. برای غیرفعال‌کردن یادداشت‌های ماندگار در سراسر سامانه،
`transcripts.enabled: false` را تنظیم کنید؛ حالت صریح `transcribe` همچنان فقط
دنباله زنده محدود خود را ارائه می‌دهد.

## محدودیت‌های پیوستن به‌عنوان مهمان

آداپتور مرورگر صفحه واسط برنامه را می‌بندد، نام مهمان را وارد می‌کند، دوربین را خاموش می‌کند، میکروفون را برای حالت انتخاب‌شده پیکربندی می‌کند و روی دکمه پیوستن کلیک می‌کند. وضعیت درون تماس از کنترل قطع تماس استفاده می‌کند؛ وضعیت‌های لابی، ورود به مستأجر و مجوز دستگاه، دلایل صریحی را برای اقدام دستی برمی‌گردانند. تغییرمسیرهای راه‌انداز جلسه مصرف‌کننده و برچسب‌های `BlackHole 2ch (Virtual)` نمایش‌داده‌شده توسط Chrome پشتیبانی می‌شوند.

سیاست مستأجر Teams ممکن است ورود به حساب، تأیید ایمیل یا پذیرش سازمان‌دهنده را الزامی کند. آن مرحله را در نمایه Chrome متعلق به OpenClaw انجام دهید، سپس بررسی وضعیت یا گفتار را دوباره امتحان کنید. این Plugin سیاست مستأجر را دور نمی‌زند.

کلاینت وب Teams مصرف‌کننده برای صفحه واسط برنامه، ورود نام مهمان، کلیدهای تغییر وضعیت میکروفون/دوربین پیش از پیوستن، پیوستن، پذیرش در لابی، مجوزهای رسانه، تشخیص وضعیت درون تماس، زیرنویس زنده، مسیریابی ورودی/خروجی BlackHole، خروج و تشخیص پس از تماس به‌صورت زنده اعتبارسنجی شده است. مستأجرهای کاری می‌توانند سیاست‌های متفاوتی برای ورود به حساب، تأیید ایمیل، پذیرش و تأیید خروج اعمال کنند؛ هر اقدام دستی گزارش‌شده را در نمایه Chrome متعلق به OpenClaw انجام دهید.

## سطح ابزار و Gateway

ابزار عامل `teams_meetings` از `join`،‏ `leave`،‏ `status`،‏ `transcript` و `speak` پشتیبانی می‌کند. متدهای Gateway از پیشوند `teamsmeetings.*` استفاده می‌کنند. فرمان Node برابر با `teamsmeetings.chrome` است.

## مرتبط

- [نمای کلی Pluginهای جلسه](/plugins/meeting-plugins)
- [کانال Microsoft Teams](/fa/channels/msteams)
