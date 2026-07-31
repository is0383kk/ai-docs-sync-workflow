---
read_when:
    - راه‌اندازی استریم بی‌صدای Matrix برای Synapse یا Tuwunel خودمیزبان‌شده
    - کاربران اعلان‌ها را فقط پس از تکمیل بلوک‌ها می‌خواهند، نه با هر ویرایش پیش‌نمایش
summary: قواعد ارسال Matrix برای هر گیرنده جهت ویرایش‌های بی‌صدای پیش‌نمایش نهایی‌شده
title: قواعد ارسال Matrix برای پیش‌نمایش‌های بی‌صدا
x-i18n:
    generated_at: "2026-07-27T14:56:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c58e7e796c3ae6d1ee25de229e4592ab8b4fb4d0d50a9cf868ab5ef35b1dab5
    source_path: channels/matrix-push-rules.md
    workflow: 16
---

وقتی `channels.matrix.streaming.mode` برابر با `"quiet"` باشد، OpenClaw پاسخ را با ویرایش درجا‌ی یک رویداد پیش‌نمایش واحد پخش جریانی می‌کند. پیش‌نمایش‌ها به‌صورت رویدادهای بدون اعلان `m.notice` ارسال می‌شوند و ویرایش نهایی‌شده با `content["com.openclaw.finalized_preview"] = true` علامت‌گذاری می‌شود. کلاینت‌های Matrix فقط در صورتی برای آن ویرایش نهایی اعلان می‌دهند که یک قانون پوش مختص کاربر با نشانگر مطابقت داشته باشد. این صفحه برای اپراتورهایی است که Matrix را به‌صورت خودمیزبان اجرا می‌کنند و می‌خواهند این قانون را برای حساب هر گیرنده نصب کنند.

`streaming.mode: "progress"` پیش‌نویس‌های خود را از همان مسیر نهایی می‌کند، بنابراین همان قانون برای ویرایش‌های نهایی‌شده در حالت پیشرفت نیز فعال می‌شود.

اگر فقط رفتار پیش‌فرض اعلان Matrix را می‌خواهید، از `streaming.mode: "partial"` استفاده کنید یا پخش جریانی را خاموش نگه دارید. به [راه‌اندازی کانال Matrix](/fa/channels/matrix#streaming-previews) مراجعه کنید.

## پیش‌نیازها

- کاربر گیرنده = شخصی که باید اعلان را دریافت کند
- کاربر ربات = حساب Matrix متعلق به OpenClaw که پاسخ را ارسال می‌کند
- برای فراخوانی‌های API زیر از توکن دسترسی کاربر گیرنده استفاده کنید
- در قانون پوش، `sender` را با MXID کامل کاربر ربات مطابقت دهید
- حساب گیرنده باید از قبل پوشِرهای فعال داشته باشد؛ قوانین پیش‌نمایش بی‌صدا فقط زمانی کار می‌کنند که تحویل عادی پوش Matrix سالم باشد

## مراحل

<Steps>
  <Step title="پیش‌نمایش‌های بی‌صدا را پیکربندی کنید">

```json5
{
  channels: {
    matrix: {
      streaming: { mode: "quiet" },
    },
  },
}
```

  </Step>

  <Step title="توکن دسترسی گیرنده را دریافت کنید">
    در صورت امکان، از توکن یک نشست موجود کلاینت دوباره استفاده کنید. برای صدور یک توکن تازه:

```bash
curl -sS -X POST \
  "https://matrix.example.org/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "m.login.password",
    "identifier": { "type": "m.id.user", "user": "@alice:example.org" },
    "password": "REDACTED"
  }'
```

  </Step>

  <Step title="وجود پوشِرها را بررسی کنید">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushers"
```

اگر هیچ پوشِری برگردانده نشد، پیش از ادامه، تحویل عادی پوش Matrix را برای این حساب اصلاح کنید.

  </Step>

  <Step title="قانون پوش بازنویسی را نصب کنید">
    قانونی نصب کنید که با نشانگر پیش‌نمایش نهایی‌شده و MXID ربات به‌عنوان فرستنده مطابقت داشته باشد:

```bash
curl -sS -X PUT \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname" \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "conditions": [
      { "kind": "event_match", "key": "type", "pattern": "m.room.message" },
      {
        "kind": "event_property_is",
        "key": "content.m\\.relates_to.rel_type",
        "value": "m.replace"
      },
      {
        "kind": "event_property_is",
        "key": "content.com\\.openclaw\\.finalized_preview",
        "value": true
      },
      { "kind": "event_match", "key": "sender", "pattern": "@bot:example.org" }
    ],
    "actions": [
      "notify",
      { "set_tweak": "sound", "value": "default" },
      { "set_tweak": "highlight", "value": false }
    ]
  }'
```

    پیش از اجرا جایگزین کنید:

    - `https://matrix.example.org`: نشانی URL پایه homeserver شما
    - `$USER_ACCESS_TOKEN`: توکن دسترسی کاربر گیرنده
    - `openclaw-finalized-preview-botname`: یک شناسه قانون یکتا برای هر ربات و هر گیرنده (الگو: `openclaw-finalized-preview-<botname>`)
    - `@bot:example.org`: شناسه MXID ربات OpenClaw شما، نه شناسه گیرنده

  </Step>

  <Step title="بررسی کنید">

```bash
curl -sS \
  -H "Authorization: Bearer $USER_ACCESS_TOKEN" \
  "https://matrix.example.org/_matrix/client/v3/pushrules/global/override/openclaw-finalized-preview-botname"
```

سپس یک پاسخ پخش‌شده را آزمایش کنید. در حالت بی‌صدا، اتاق یک پیش‌نمایش بی‌صدای پیش‌نویس را نشان می‌دهد و پس از پایان بلوک یا نوبت، یک‌بار اعلان می‌دهد.

  </Step>
</Steps>

برای حذف قانون در آینده، نشانی URL همان قانون را با توکن گیرنده `DELETE` کنید.

## نکات مربوط به چند ربات

کلید قوانین پوش بر اساس `ruleId` تعیین می‌شود: اجرای دوباره `PUT` برای همان شناسه، یک قانون واحد را به‌روزرسانی می‌کند. برای چند ربات OpenClaw که به یک گیرنده اعلان می‌دهند، برای هر ربات یک قانون با تطبیق فرستنده متمایز ایجاد کنید.

قوانین جدید تعریف‌شده توسط کاربر از نوع `override` پیش از قوانین سرکوب پیش‌فرض سرور درج می‌شوند، بنابراین به پارامتر ترتیب دیگری نیاز نیست. این قانون فقط بر ویرایش‌های پیش‌نمایشِ صرفاً متنی اثر می‌گذارد که بتوان آن‌ها را درجا نهایی کرد؛ پاسخ‌های رسانه‌ای، بازگشت‌های جایگزینِ پیش‌نمایش منقضی و متن‌های نهایی‌ای که اشاره‌های Matrix را فعال می‌کنند، در عوض به‌صورت پیام‌های عادیِ دارای اعلان تحویل داده می‌شوند.

## نکات homeserver

<AccordionGroup>
  <Accordion title="Synapse">
    هیچ تغییر خاصی در `homeserver.yaml` لازم نیست. اگر اعلان‌های عادی Matrix از قبل به این کاربر می‌رسند، توکن گیرنده و فراخوانی `pushrules` بالا، مرحله اصلی راه‌اندازی هستند.

    اگر Synapse را پشت یک پروکسی معکوس یا workerها اجرا می‌کنید، مطمئن شوید `/_matrix/client/.../pushrules/` به‌درستی به Synapse می‌رسد. تحویل پوش توسط فرایند اصلی یا `synapse.app.pusher` / workerهای پیکربندی‌شده پوشِر انجام می‌شود؛ از سالم‌بودن آن‌ها اطمینان حاصل کنید.

    این قانون از شرط قانون پوش `event_property_is` ‏(MSC3758، قانون پوش v1.10) استفاده می‌کند که در سال 2023 به Synapse افزوده شد. نسخه‌های قدیمی‌تر Synapse فراخوانی `PUT pushrules/...` را می‌پذیرند، اما بدون هیچ هشداری هرگز شرط را مطابقت نمی‌دهند؛ اگر برای ویرایش پیش‌نمایش نهایی‌شده اعلانی دریافت نمی‌شود، Synapse را ارتقا دهید.

  </Accordion>

  <Accordion title="Tuwunel">
    جریان کار همانند Synapse است؛ برای نشانگر پیش‌نمایش نهایی‌شده به پیکربندی مختص Tuwunel نیازی نیست.

    اگر هنگامی که کاربر در دستگاه دیگری فعال است اعلان‌ها ناپدید می‌شوند، بررسی کنید که آیا `suppress_push_when_active` فعال است. Tuwunel این گزینه را در نسخه 1.4.2 (سپتامبر 2025) اضافه کرد و این گزینه می‌تواند وقتی یک دستگاه فعال است، پوش‌ها به دستگاه‌های دیگر را به‌عمد سرکوب کند.

  </Accordion>
</AccordionGroup>

## مرتبط

- [راه‌اندازی کانال Matrix](/fa/channels/matrix)
- [مفاهیم پخش جریانی](/fa/concepts/streaming)
