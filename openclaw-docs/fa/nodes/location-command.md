---
read_when:
    - افزودن پشتیبانی Node از موقعیت مکانی یا رابط کاربری مجوزها
    - طراحی مجوزهای موقعیت مکانی یا رفتار پیش‌زمینه در Android
summary: فرمان موقعیت مکانی برای Nodeها، حالت‌های مجوز پلتفرم و راه‌اندازی GeoClue در Linux
title: فرمان موقعیت مکانی
x-i18n:
    generated_at: "2026-07-27T16:43:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 644229c1eafc8fc7b59bc23ba01d4ba95687ea66c4f9bd4a4cda98a87f2b6085
    source_path: nodes/location-command.md
    workflow: 16
---

## خلاصه

- `location.get` یک فرمان Node است که از طریق `node.invoke` یا `openclaw nodes location get` فراخوانی می‌شود.
- به‌طور پیش‌فرض خاموش است.
- بیلدهای شخص ثالث Android از یک انتخاب‌گر استفاده می‌کنند: خاموش / هنگام استفاده / همیشه. بیلدهای Play همچنان گزینه‌های خاموش / هنگام استفاده را دارند.
- مکان دقیق یک کلید جداگانه است.

## چرا انتخاب‌گر (و نه فقط یک کلید)

مجوزهای مکان سیستم‌عامل چندسطحی هستند. مکان دقیق نیز مجوز جداگانه‌ای در سیستم‌عامل است («Precise» در iOS 14+ و «fine» در برابر «coarse» در Android). انتخاب‌گر درون برنامه حالت درخواستی را تعیین می‌کند، اما تصمیم نهایی درباره مجوز اعطاشده همچنان با سیستم‌عامل است.

## مدل تنظیمات

برای هر دستگاه Node:

- `location.enabledMode`: `off | whileUsing | always`
- `location.preciseEnabled`: bool

رفتار رابط کاربری:

- انتخاب `whileUsing` مجوز پیش‌زمینه را درخواست می‌کند.
- انتخاب `always` در بیلد شخص ثالث Android ابتدا مجوز پیش‌زمینه را درخواست می‌کند، دسترسی پس‌زمینه را توضیح می‌دهد و سپس تنظیمات برنامه در Android را برای اعطای مجزای **Allow all the time** باز می‌کند.
- بیلدهای Android Play مجوز مکان پس‌زمینه را اعلام نمی‌کنند و `always` را نمایش نمی‌دهند.
- اگر سیستم‌عامل سطح درخواستی را رد کند، برنامه به بالاترین سطح اعطاشده بازمی‌گردد و وضعیت را نمایش می‌دهد.

## نگاشت مجوزها (node.permissions)

اختیاری است. Node در macOS مقدار `location` را از طریق نگاشت `permissions` در `node.list`/`node.describe` گزارش می‌کند؛ iOS/Android ممکن است آن را حذف کنند.

## فرمان: `location.get`

از طریق `node.invoke` یا ابزار کمکی CLI فراخوانی می‌شود:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

پارامترها:

```json
{
  "timeoutMs": 10000,
  "maxAgeMs": 15000,
  "desiredAccuracy": "coarse|balanced|precise"
}
```

پرچم‌های CLI مستقیماً نگاشت می‌شوند: `--location-timeout` -> `timeoutMs`، `--max-age` -> `maxAgeMs`، `--accuracy` -> `desiredAccuracy`.

بار پاسخ:

```json
{
  "lat": 48.20849,
  "lon": 16.37208,
  "accuracyMeters": 12.5,
  "altitudeMeters": 182.0,
  "speedMps": 0.0,
  "headingDeg": 270.0,
  "timestamp": "2026-01-03T12:34:56.000Z",
  "isPrecise": true,
  "source": "gps|wifi|cell|unknown"
}
```

خطاها (کدهای پایدار):

- `LOCATION_DISABLED`: انتخاب‌گر خاموش است.
- `LOCATION_PERMISSION_REQUIRED`: مجوز لازم برای حالت درخواستی وجود ندارد.
- `LOCATION_BACKGROUND_UNAVAILABLE`: برنامه در پس‌زمینه است، اما فقط مجوز هنگام استفاده اعطا شده است.
- `LOCATION_TIMEOUT`: موقعیت در زمان مقرر تثبیت نشد.
- `LOCATION_UNAVAILABLE`: خرابی سیستم یا نبود ارائه‌دهنده.

## رفتار در پس‌زمینه

- بیلدهای شخص ثالث Android فقط زمانی `location.get` را در پس‌زمینه می‌پذیرند که کاربر `Always` را انتخاب کرده باشد و Android مجوز مکان پس‌زمینه را اعطا کرده باشد. سرویس پایدار موجود Node نوع سرویس `location` را اضافه می‌کند و هنگام فعال بودن، `Location: Always` را اعلام می‌کند.
- بیلدهای Android Play و حالت `While Using` در زمان قرار داشتن در پس‌زمینه، `location.get` را رد می‌کنند.
- ممکن است سایر پلتفرم‌های Node رفتار متفاوتی داشته باشند.

## میزبان Node در Linux

Plugin همراه Node در Linux، `location.get` را به سرویس `openclaw node` در CLI اضافه می‌کند؛ این شامل میزبان‌های بدون رابط گرافیکی که برنامه دسکتاپ Linux را ندارند نیز می‌شود. مکان به‌طور پیش‌فرض خاموش است. آن را در ورودی Plugin فعال کنید و سپس سرویس Node را راه‌اندازی مجدد کنید:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          location: { enabled: true },
        },
      },
    },
  },
}
```

GeoClue2 و دموی `where-am-i` آن (`geoclue-2-demo` در Debian و Ubuntu) را نصب کنید. کاربر سرویس Node باید طبق سیاست GeoClue میزبان و عامل مجوزدهی آن اجازه دسترسی داشته باشد.

Plugin به‌جای مجموعه‌ای از فراخوانی‌های `busctl` از `where-am-i` استفاده می‌کند. GeoClue ایجاد کلاینت، ویژگی‌ها، شروع، به‌روزرسانی‌ها و توقف را به یک اتصال کلاینت D-Bus پیوند می‌دهد؛ دمو این چرخه عمر را یکپارچه نگه می‌دارد، درحالی‌که زیرفرایندهای جداگانه `busctl` چنین نمی‌کنند. هیچ وابستگی npm اضافه نمی‌شود.

Linux مقادیر `coarse`، `balanced` و `precise` را به‌ترتیب به سطوح دقت `4`، `6` و `8` در GeoClue نگاشت می‌کند. مقدار `maxAgeMs` را در برابر مُهر زمانی بازگشتی اعتبارسنجی می‌کند. دموی GeoClue ارائه‌دهنده انتخاب‌شده را آشکار نمی‌کند، بنابراین `source` برابر با `unknown` است؛ `isPrecise` فقط زمانی true است که دقت گزارش‌شده 100 متر یا بهتر باشد.

Linux از همان خطاهای پایدار استفاده می‌کند: `LOCATION_DISABLED`، `LOCATION_TIMEOUT` و `LOCATION_UNAVAILABLE`.

## یکپارچه‌سازی مدل و ابزارها

- ابزار عامل: کنش `location_get` در ابزار `nodes` (نیازمند Node).
- CLI: `openclaw nodes location get --node <id>`.
- رهنمودهای عامل: فقط زمانی فراخوانی شود که کاربر مکان را فعال کرده و از دامنه دسترسی آن آگاه باشد.

## متن پیشنهادی تجربه کاربری

- خاموش: «اشتراک‌گذاری مکان غیرفعال است.»
- هنگام استفاده: «فقط زمانی که OpenClaw باز است.»
- همیشه: «هنگامی که OpenClaw در پس‌زمینه است، بررسی‌های درخواستی مکان را مجاز کن.»
- دقیق: «از مکان دقیق GPS استفاده کن. برای اشتراک‌گذاری مکان تقریبی، این گزینه را خاموش کن.»

## مرتبط

- [نمای کلی Nodeها](/fa/nodes)
- [تجزیه مکان کانال](/fa/channels/location)
- [ثبت تصویر دوربین](/fa/nodes/camera)
- [حالت مکالمه](/fa/nodes/talk)
