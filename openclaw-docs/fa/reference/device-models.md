---
read_when:
    - به‌روزرسانی نگاشت‌های شناسه مدل دستگاه یا فایل‌های NOTICE/مجوز
    - تغییر نحوه نمایش نام دستگاه‌ها در رابط کاربری Instances
summary: OpenClaw چگونه شناسه‌های مدل دستگاه‌های Apple را برای نمایش نام‌های خوانا در برنامه macOS به‌صورت داخلی عرضه می‌کند.
title: پایگاه داده مدل دستگاه
x-i18n:
    generated_at: "2026-07-27T14:37:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 930cd330594072d9c986b8c85c5a68e02dd096e5f0c015e3ee86b767073b93e6
    source_path: reference/device-models.md
    workflow: 16
---

رابط کاربری **Instances** در اپ همراه macOS، شناسه‌های مدل Apple را به نام‌های خوانا نگاشت می‌کند (`iPad16,6` -> "iPad Pro 13-inch (M4)"، `Mac16,6` -> "MacBook Pro (14-inch, 2024)"). همچنین `DeviceModelCatalog` از پیشوند شناسه (با بازگشت به خانواده دستگاه در صورت نیاز) برای انتخاب یک SF Symbol برای هر دستگاه استفاده می‌کند.

فایل‌های موجود در `apps/macos/Sources/OpenClaw/Resources/DeviceModels/`:

| فایل                                   | کاربرد                               |
| -------------------------------------- | ------------------------------------- |
| `ios-device-identifiers.json`          | نگاشت شناسه iOS/iPadOS به نام |
| `mac-device-identifiers.json`          | نگاشت شناسه Mac به نام        |
| `NOTICE.md`                            | SHAهای کامیت بالادستی سنجاق‌شده           |
| `LICENSE.apple-device-identifiers.txt` | مجوز MIT بالادستی                  |

## منبع داده

از مخزن GitHub دارای مجوز MIT با نام `kyle-seongwoo-jun/apple-device-identifiers` به‌صورت کپی داخلی تهیه شده است. فایل‌های JSON به SHAهای کامیت ثبت‌شده در `NOTICE.md` سنجاق شده‌اند تا بیلدها قطعی باقی بمانند.

## به‌روزرسانی پایگاه داده

1. SHAهای کامیت بالادستی را برای سنجاق‌کردن انتخاب کنید (یکی برای iOS و یکی برای macOS).
2. فایل `apps/macos/Sources/OpenClaw/Resources/DeviceModels/NOTICE.md` را با SHAهای جدید به‌روزرسانی کنید.
3. فایل‌های JSON سنجاق‌شده به آن کامیت‌ها را دوباره دانلود کنید:

```bash
IOS_COMMIT="<commit sha for ios-device-identifiers.json>"
MAC_COMMIT="<commit sha for mac-device-identifiers.json>"

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${IOS_COMMIT}/ios-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/ios-device-identifiers.json

curl -fsSL "https://raw.githubusercontent.com/kyle-seongwoo-jun/apple-device-identifiers/${MAC_COMMIT}/mac-device-identifiers.json" \
  -o apps/macos/Sources/OpenClaw/Resources/DeviceModels/mac-device-identifiers.json
```

4. تأیید کنید که `LICENSE.apple-device-identifiers.txt` همچنان با نسخه بالادستی مطابقت دارد؛ اگر مجوز بالادستی تغییر کرده است، آن را جایگزین کنید.
5. بررسی کنید که اپ macOS بدون خطا بیلد می‌شود:

```bash
swift build --package-path apps/macos
```

## مرتبط

- [Nodeها](/fa/nodes)
- [عیب‌یابی Node](/fa/nodes/troubleshooting)
