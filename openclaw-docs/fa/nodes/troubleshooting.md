---
read_when:
    - Node متصل است، اما ابزارهای دوربین/بوم/صفحه‌نمایش/اجرا کار نمی‌کنند
    - به درک مدل ذهنی جفت‌سازی Node در برابر تأییدها نیاز دارید
summary: عیب‌یابی جفت‌سازی Node، الزامات اجرای پیش‌زمینه، مجوزها و خطاهای ابزارها
title: عیب‌یابی Node
x-i18n:
    generated_at: "2026-07-27T15:29:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7ee9e48985805e91cd5acfa1b9f6b676b7e67236ce29fe91e2c8d03002e5c4
    source_path: nodes/troubleshooting.md
    workflow: 16
---

وقتی یک Node در وضعیت قابل‌مشاهده است اما ابزارهای Node کار نمی‌کنند، از این صفحه استفاده کنید.

## سلسله‌مراتب فرمان‌ها

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

سپس بررسی‌های مختص Node را اجرا کنید:

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
```

نشانه‌های وضعیت سالم:

- Node برای نقش `node` متصل و جفت شده است.
- `nodes describe` قابلیت مورد فراخوانی را شامل می‌شود.
- تأییدهای اجرا حالت/فهرست مجاز مورد انتظار را نشان می‌دهند.

## الزامات پیش‌زمینه

`canvas.*`، `camera.*` و `screen.*` در Nodeهای iOS/Android فقط در پیش‌زمینه کار می‌کنند.

بررسی و رفع سریع:

```bash
openclaw nodes describe --node <idOrNameOrIp>
openclaw nodes canvas snapshot --node <idOrNameOrIp>
openclaw logs --follow
```

اگر `NODE_BACKGROUND_UNAVAILABLE` را مشاهده کردید، برنامه Node را به پیش‌زمینه بیاورید و دوباره تلاش کنید.

## ماتریس مجوزها

| قابلیت                   | iOS                                     | Android                                      | برنامه Node در macOS                   | کد خطای معمول                          |
| ---------------------------- | --------------------------------------- | -------------------------------------------- | -------------------------------- | --------------------------------------------- |
| `camera.snap`، `camera.clip` | دوربین (+ میکروفون برای صدای کلیپ)           | دوربین (+ میکروفون برای صدای کلیپ)                | دوربین (+ میکروفون برای صدای کلیپ)    | `*_PERMISSION_REQUIRED`                       |
| `screen.record`              | ضبط صفحه (+ میکروفون اختیاری)       | درخواست ضبط صفحه (+ میکروفون اختیاری)       | ضبط صفحه                 | `*_PERMISSION_REQUIRED`                       |
| `computer.act`               | قابل‌اعمال نیست                                     | قابل‌اعمال نیست                                          | دسترسی‌پذیری + ضبط صفحه | `COMPUTER_DISABLED`، `ACCESSIBILITY_REQUIRED` |
| `location.get`               | هنگام استفاده یا همیشه (بسته به حالت) | موقعیت مکانی پیش‌زمینه/پس‌زمینه بر اساس حالت | مجوز موقعیت مکانی              | `LOCATION_PERMISSION_REQUIRED`                |
| `system.run`                 | قابل‌اعمال نیست (مسیر میزبان Node)                    | قابل‌اعمال نیست (مسیر میزبان Node)                         | تأییدهای اجرا الزامی‌اند          | `SYSTEM_RUN_DENIED`                           |

## جفت‌سازی در برابر تأییدها

سه دروازهٔ مجزا موفقیت یک فرمان Node را کنترل می‌کنند:

1. **جفت‌سازی دستگاه**: آیا این Node می‌تواند به Gateway متصل شود؟
2. **خط‌مشی فرمان Node در Gateway**: آیا شناسهٔ فرمان RPC طبق `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny` و پیش‌فرض‌های پلتفرم مجاز است؟
3. **تأییدهای اجرا**: آیا این Node می‌تواند یک فرمان پوستهٔ مشخص را به‌صورت محلی اجرا کند؟

جفت‌سازی Node یک دروازهٔ هویت/اعتماد است، نه یک سطح تأیید برای هر فرمان. برای `system.run`، خط‌مشی هر Node در فایل تأییدهای اجرای همان Node (`openclaw approvals get --node ...`) قرار دارد، نه در رکورد جفت‌سازی Gateway.

بررسی‌های سریع:

```bash
openclaw devices list
openclaw nodes status
openclaw approvals get --node <idOrNameOrIp>
openclaw approvals allowlist add --node <idOrNameOrIp> "/usr/bin/uname"
```

- جفت‌سازی موجود نیست: ابتدا دستگاه Node را تأیید کنید.
- فرمانی در `nodes describe` موجود نیست: خط‌مشی فرمان Node در Gateway را بررسی کنید و مطمئن شوید Node هنگام اتصال واقعاً آن فرمان را اعلام کرده است.
- جفت‌سازی درست است، اما `system.run` ناموفق می‌شود: تأییدهای اجرا/فهرست مجاز را در آن Node اصلاح کنید.

برای اجرای `host=node` مبتنی بر تأیید، Gateway اجرا را به `systemRunPlan` متعارف و آماده‌شده نیز مقید می‌کند. اگر فراخواننده‌ای بعداً فرمان، cwd یا فرادادهٔ نشست را پیش از ارسال اجرای تأییدشده تغییر دهد، Gateway به‌جای اعتماد به بار دادهٔ ویرایش‌شده، اجرا را به‌دلیل عدم تطابق تأیید رد می‌کند.

## کدهای خطای رایج Node

| کد                                   | معنی                                                                                                                                                                                 |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `NODE_BACKGROUND_UNAVAILABLE`          | برنامه در پس‌زمینه است؛ آن را به پیش‌زمینه بیاورید.                                                                                                                                        |
| `CAMERA_DISABLED`                      | کلید دوربین در تنظیمات Node غیرفعال است.                                                                                                                                                |
| `*_PERMISSION_REQUIRED`                | مجوز سیستم‌عامل موجود نیست یا رد شده است.                                                                                                                                                           |
| `LOCATION_DISABLED`                    | حالت موقعیت مکانی خاموش است.                                                                                                                                                                   |
| `LOCATION_PERMISSION_REQUIRED`         | حالت موقعیت مکانی درخواستی اعطا نشده است.                                                                                                                                                    |
| `LOCATION_BACKGROUND_UNAVAILABLE`      | برنامه در پس‌زمینه است، اما فقط مجوز هنگام استفاده وجود دارد.                                                                                                                             |
| `COMPUTER_DISABLED`                    | **Allow Computer Control** را در برنامه macOS فعال کنید، سپس به‌روزرسانی جفت‌سازی را تأیید کنید.                                                                                                    |
| `ACCESSIBILITY_REQUIRED`               | در تنظیمات سیستم macOS، به بستهٔ فعلی برنامه OpenClaw مجوز دسترسی‌پذیری بدهید.                                                                                                        |
| `SYSTEM_RUN_DENIED: approval required` | درخواست اجرا به تأیید صریح نیاز دارد.                                                                                                                                                   |
| `SYSTEM_RUN_DENIED: allowlist miss`    | فرمان در حالت فهرست مجاز مسدود شده است. در میزبان‌های Node ویندوز، شکل‌های پوشش‌دهندهٔ پوسته مانند `cmd.exe /c ...` در حالت فهرست مجاز به‌عنوان موارد ناموجود در فهرست مجاز تلقی می‌شوند، مگر اینکه از طریق جریان پرسش تأیید شده باشند. |

## چرخهٔ بازیابی سریع

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
```

اگر همچنان مشکل باقی بود:

- جفت‌سازی دستگاه را دوباره تأیید کنید.
- برنامه Node را دوباره باز کنید (در پیش‌زمینه).
- مجوزهای سیستم‌عامل را دوباره اعطا کنید.
- خط‌مشی تأیید اجرا را دوباره ایجاد یا تنظیم کنید.

برای کنترل رایانه، همچنین بررسی کنید که یک عامل دارای قابلیت بینایی ابزار `computer` را ارائه می‌کند، `screen.snapshot` با مجوز ضبط صفحه با موفقیت انجام می‌شود و `/phone status` مجوز موقت یا پایدار Gateway موردنظر شما را نشان می‌دهد. ورودی `gateway.nodes.commands.deny` همیشه `gateway.nodes.commands.allow` را لغو می‌کند.

## مرتبط

- [نمای کلی Nodeها](/fa/nodes)
- [Nodeهای دوربین](/fa/nodes/camera)
- [فرمان موقعیت مکانی](/fa/nodes/location-command)
- [استفاده از رایانه](/fa/nodes/computer-use)
- [تأییدهای اجرا](/fa/tools/exec-approvals)
- [جفت‌سازی Gateway](/fa/gateway/pairing)
- [عیب‌یابی Gateway](/fa/gateway/troubleshooting)
- [عیب‌یابی کانال](/fa/channels/troubleshooting)
