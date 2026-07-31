---
read_when:
    - راه‌اندازی پشتیبانی از Signal
    - اشکال‌زدایی ارسال/دریافت Signal
summary: پشتیبانی از Signal از طریق signal-cli (دیمون بومی یا کانتینر bbernhard)، مسیرهای راه‌اندازی و مدل شماره تلفن
title: Signal
x-i18n:
    generated_at: "2026-07-27T13:44:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 744f817e425d378e9f3e160df534019a6fc865227eb3fc68959a12ad46c0b714
    source_path: channels/signal.md
    workflow: 16
---

Signal یک plugin کانال قابل‌دانلود است (`@openclaw/signal`). Gateway از طریق HTTP با `signal-cli` ارتباط برقرار می‌کند: یا دیمن بومی (JSON-RPC + SSE) یا کانتینر [bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api)‏ (REST + WebSocket). ‏OpenClaw، ‏libsignal را درون خود تعبیه نمی‌کند.

## مدل شماره (ابتدا این بخش را بخوانید)

- Gateway به یک **دستگاه Signal** متصل می‌شود: حساب `signal-cli`.
- اجرای ربات روی **حساب شخصی Signal شما** باعث می‌شود پیام‌های خودتان را نادیده بگیرد (محافظت در برابر حلقه).
- برای حالت «به ربات پیام می‌دهم و پاسخ می‌دهد»، از یک **شماره جداگانه برای ربات** استفاده کنید.

## نصب

```bash
openclaw plugins install @openclaw/signal
```

مشخصات ساده plugin ابتدا ClawHub و سپس npm را به‌عنوان مسیر جایگزین امتحان می‌کند. با `openclaw plugins install clawhub:@openclaw/signal` یا `npm:@openclaw/signal` یک منبع را اجباراً انتخاب کنید. `plugins install`، ‏plugin را ثبت و فعال می‌کند؛ به مرحله جداگانه `enable` نیازی نیست. برای قواعد عمومی نصب، به [Pluginها](/fa/tools/plugin) مراجعه کنید.

## راه‌اندازی سریع

<Steps>
  <Step title="انتخاب شماره">
    برای ربات از یک **شماره جداگانه Signal** استفاده کنید (توصیه می‌شود).
  </Step>
  <Step title="نصب plugin">
    ```bash
    openclaw plugins install @openclaw/signal
    ```
  </Step>
  <Step title="اجرای راه‌اندازی هدایت‌شده">
    ```bash
    openclaw channels add
    ```
    راهنما تشخیص می‌دهد که آیا `signal-cli` در `PATH` وجود دارد و اگر موجود نباشد، نصب آن را پیشنهاد می‌کند: در Linux x86-64 بیلد بومی رسمی GraalVM را دانلود می‌کند، یا در macOS و معماری‌های دیگر آن را از طریق Homebrew نصب می‌کند. سپس شماره ربات و مسیر `signal-cli` را درخواست می‌کند.

    برای راه‌اندازی غیرتعاملی، `openclaw channels add --channel signal` همچنین `--signal-number <e164>` را برای شماره تلفن ربات و نیز `--http-host <host>` و `--http-port <port>` را برای نقطه پایانی دیمن Signal می‌پذیرد (پیش‌فرض `127.0.0.1:8080`).

  </Step>
  <Step title="پیوند دادن یا ثبت حساب">
    - **پیوند QR (سریع‌ترین):** ‏`signal-cli link -n "OpenClaw"`، سپس با Signal اسکن کنید. به [مسیر A](#setup-path-a-link-existing-signal-account-qr) مراجعه کنید.
    - **ثبت‌نام پیامکی:** شماره اختصاصی همراه با کپچا و تأیید پیامکی. به [مسیر B](#setup-path-b-register-dedicated-bot-number-sms-linux) مراجعه کنید.

  </Step>
  <Step title="تأیید و جفت‌سازی">
    ```bash
    openclaw gateway call channels.status --params '{"probe":true}'
    ```
    نخستین پیام مستقیم را ارسال و جفت‌سازی را تأیید کنید: `openclaw pairing approve signal <CODE>`.
  </Step>
</Steps>

پیکربندی حداقلی:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "managed-native",
        cliPath: "signal-cli",
      },
      dmPolicy: "pairing",
      allowFrom: ["+15557654321"],
    },
  },
}
```

| فیلد       | توضیحات                                       |
| ----------- | ------------------------------------------------- |
| `account`   | شماره تلفن ربات با قالب E.164 ‏(`+15551234567`) |
| `transport` | اتصال Signal متعلق به حساب و حالت فرایند  |
| `dmPolicy`  | سیاست دسترسی پیام مستقیم (‏`pairing` توصیه می‌شود)          |
| `allowFrom` | شماره‌های تلفن یا مقادیر `uuid:<id>` مجاز برای ارسال پیام مستقیم |

پشتیبانی از چند حساب: از `channels.signal.accounts` همراه با پیکربندی مختص هر حساب و `name` اختیاری استفاده کنید. هر حساب نام‌گذاری‌شده مالک `transport` خود است و آن را از انتقال سطح بالا به ارث نمی‌برد. انتقال سطح بالا فقط به حساب ضمنی `default` تعلق دارد. برای الگوی مشترک، به [کانال‌های چندحسابی](/fa/gateway/config-channels#multi-account-all-channels) مراجعه کنید.

## ماهیت آن

- مسیریابی قطعی: پاسخ‌ها همیشه به Signal بازمی‌گردند.
- پیام‌های مستقیم نشست اصلی عامل را به‌اشتراک می‌گذارند؛ گروه‌ها ایزوله هستند (`agent:<agentId>:signal:group:<groupId>`).
- به‌طور پیش‌فرض، Signal ممکن است به‌روزرسانی‌های پیکربندی فعال‌شده توسط `/config set|unset` را بنویسد (به `commands.config: true` نیاز دارد). با `channels.signal.configWrites: false` غیرفعال کنید.

## مسیر راه‌اندازی A: پیوند دادن حساب موجود Signal ‏(QR)

1. `signal-cli` را نصب کنید (بیلد JVM یا بومی)، یا اجازه دهید `openclaw channels add` آن را برای شما نصب کند.
2. یک حساب ربات را پیوند دهید: `signal-cli link -n "OpenClaw"`، سپس کد QR را در Signal اسکن کنید.
3. Signal را پیکربندی و Gateway را راه‌اندازی کنید.

## مسیر راه‌اندازی B: ثبت شماره اختصاصی ربات (پیامک، Linux)

از این روش برای یک شماره اختصاصی ربات، به‌جای پیوند دادن حساب موجود برنامه Signal، استفاده کنید. جریان زیر روی Ubuntu 24 آزمایش شده است.

1. شماره‌ای تهیه کنید که بتواند پیامک دریافت کند (یا برای تلفن ثابت، تأیید صوتی). شماره اختصاصی ربات از تداخل حساب/نشست جلوگیری می‌کند.
2. `signal-cli` را روی میزبان Gateway نصب کنید:

```bash
VERSION=$(curl -Ls -o /dev/null -w %{url_effective} https://github.com/AsamK/signal-cli/releases/latest | sed -e 's/^.*\/v//')
curl -L -O "https://github.com/AsamK/signal-cli/releases/download/v${VERSION}/signal-cli-${VERSION}-Linux-native.tar.gz"
sudo tar xf "signal-cli-${VERSION}-Linux-native.tar.gz" -C /opt
sudo ln -sf /opt/signal-cli /usr/local/bin/
signal-cli --version
```

اگر از بیلد JVM ‏(`signal-cli-${VERSION}.tar.gz`) استفاده می‌کنید، ابتدا یک JRE نصب کنید. `signal-cli` را به‌روز نگه دارید؛ در بالادست ذکر شده است که با تغییر APIهای سرور Signal، نسخه‌های قدیمی ممکن است از کار بیفتند.

3. شماره را ثبت و تأیید کنید:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register
```

اگر کپچا لازم است (برای تکمیل این مرحله دسترسی به مرورگر لازم است):

1. `https://signalcaptchas.org/registration/generate.html` را باز کنید.
2. کپچا را تکمیل کنید و مقصد پیوند `signalcaptcha://...` را از "Open Signal" کپی کنید.
3. در صورت امکان، دستور را از همان IP خارجی نشست مرورگر اجرا کنید (توکن‌های کپچا به‌سرعت منقضی می‌شوند).
4. بلافاصله ثبت و تأیید کنید:

```bash
signal-cli -a +<BOT_PHONE_NUMBER> register --captcha '<SIGNALCAPTCHA_URL>'
signal-cli -a +<BOT_PHONE_NUMBER> verify <VERIFICATION_CODE>
```

4. ‏OpenClaw را پیکربندی کنید، Gateway را دوباره راه‌اندازی کنید و کانال را تأیید کنید:

```bash
# اگر Gateway را به‌عنوان سرویس systemd کاربر اجرا می‌کنید:
systemctl --user restart openclaw-gateway.service

# سپس تأیید کنید:
openclaw doctor
openclaw channels status --probe
```

5. فرستنده پیام مستقیم خود را جفت کنید:
   - هر پیامی را به شماره ربات ارسال کنید.
   - روی سرور تأیید کنید: `openclaw pairing approve signal <PAIRING_CODE>`.
   - برای جلوگیری از نمایش "Unknown contact"، شماره ربات را به‌عنوان مخاطب در تلفن خود ذخیره کنید.

<Warning>
ثبت حساب شماره تلفن با `signal-cli` می‌تواند نشست برنامه اصلی Signal را برای آن شماره از احراز هویت خارج کند. یک شماره اختصاصی ربات را ترجیح دهید، یا برای حفظ تنظیمات فعلی برنامه تلفن خود از حالت پیوند QR استفاده کنید.
</Warning>

منابع بالادست:

- ‏README مربوط به `signal-cli`: ‏`https://github.com/AsamK/signal-cli`
- جریان کپچا: `https://github.com/AsamK/signal-cli/wiki/Registration-with-captcha`
- جریان پیوند دادن: `https://github.com/AsamK/signal-cli/wiki/Linking-other-devices-(Provisioning)`

## حالت دیمن بومی خارجی

برای مدیریت `signal-cli` توسط خودتان (شروع سرد کُند JVM، مقداردهی اولیه کانتینر، CPUهای اشتراکی)، دیمن را جداگانه اجرا کنید و OpenClaw را به آن هدایت کنید:

برای راه‌اندازی غیرتعاملی، در صورت نیاز نوع نقطه پایانی را صریحاً انتخاب کنید:

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://127.0.0.1:8080 --signal-transport external-native
```

```json5
{
  channels: {
    signal: {
      transport: {
        kind: "external-native",
        url: "http://127.0.0.1:8080",
      },
    },
  },
}
```

این کار ایجاد خودکار و انتظار هنگام راه‌اندازی OpenClaw را رد می‌کند. برای دیمن مدیریت‌شده‌ای که آهسته شروع می‌شود، `channels.signal.transport.startupTimeoutMs` را تنظیم کنید.

## حالت کانتینر (bbernhard/signal-cli-rest-api)

به‌جای اجرای بومی `signal-cli`، از کانتینر Docker ‏[bbernhard/signal-cli-rest-api](https://github.com/bbernhard/signal-cli-rest-api) استفاده کنید که `signal-cli` را پشت رابط REST + WebSocket قرار می‌دهد.

```bash
openclaw channels add --channel signal --signal-number +15551234567 \
  --http-url http://signal-cli:8080 --signal-transport container
```

الزامات:

- کانتینر برای دریافت بلادرنگ پیام‌ها **باید** با `MODE=json-rpc` اجرا شود.
- پیش از اتصال OpenClaw، حساب Signal خود را داخل کانتینر ثبت یا پیوند دهید.

نمونه سرویس `docker-compose.yml`:

```yaml
signal-cli:
  image: bbernhard/signal-cli-rest-api:latest
  environment:
    MODE: json-rpc
  ports:
    - "8080:8080"
  volumes:
    - signal-cli-data:/home/.local/share/signal-cli
```

پیکربندی OpenClaw:

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15551234567",
      transport: {
        kind: "container",
        url: "http://signal-cli:8080",
      },
    },
  },
}
```

`transport.kind` تعیین می‌کند OpenClaw از کدام پروتکل و چرخه‌عمر فرایند استفاده کند:

| مقدار               | رفتار                                                                                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `"managed-native"`  | signal-cli بومی را اجرا می‌کند و از JSON-RPC در `/api/v1/rpc` به‌همراه SSE در `/api/v1/events` استفاده می‌کند؛ `url` ممکن است نقطه پایانی اتصالی متمایز از نشانی اتصال دیمن انتخاب کند |
| `"external-native"` | به دیمن بومی signal-cli که از قبل در حال اجرا است متصل می‌شود                                                                                                       |
| `"container"`       | به REST مربوط به bbernhard در `/v2/send` و WebSocket در `/v1/receive/{account}` متصل می‌شود                                                                             |

راه‌اندازی و `openclaw doctor --fix` ممکن است برای شناسایی نوع دقیق یک نقطه پایانی موجود، آن را یک‌بار کاوش کنند. عملیات زمان اجرا پروتکل‌ها را به‌طور خودکار تشخیص نمی‌دهند یا تغییر نمی‌دهند.

حالت کانتینر، در مواردی که کانتینر APIهای متناظر را ارائه کند، از همان عملیات Signal در حالت بومی پشتیبانی می‌کند: ارسال، دریافت، پیوست‌ها، نشانگرهای تایپ، رسیدهای خواندن/مشاهده‌شدن، واکنش‌ها، گروه‌ها و متن سبک‌دهی‌شده. OpenClaw فراخوانی‌های RPC بومی Signal را به بارهای REST کانتینر ترجمه می‌کند، از جمله شناسه‌های گروه `group.{base64(internal_id)}` و `text_mode: "styled"` برای متن قالب‌بندی‌شده.

نکات عملیاتی:

- برای دریافت از `MODE=json-rpc` استفاده کنید. `MODE=normal` می‌تواند باعث شود `/v1/about` سالم به‌نظر برسد، اما `/v1/receive/{account}` به WebSocket ارتقا نمی‌یابد؛ بنابراین کاوش جریان دریافت کانتینر شکست می‌خورد.
- `kind: "container"` را برای API ‏REST مربوط به bbernhard و `kind: "external-native"` را برای JSON-RPC/SSE بومی `signal-cli` تنظیم کنید.
- دانلود پیوست در کانتینر از همان محدودیت‌های بایتی رسانه در حالت بومی پیروی می‌کند. وقتی سرور `Content-Length` را ارسال کند، پاسخ‌های بیش‌ازحد بزرگ پیش از بافر شدن کامل رد می‌شوند؛ در غیر این صورت هنگام جریان‌یابی رد می‌شوند.

## کنترل دسترسی (پیام‌های مستقیم + گروه‌ها)

پیام‌های مستقیم:

- پیش‌فرض: `channels.signal.dmPolicy = "pairing"`.
- فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند؛ پیام‌ها تا زمان تأیید نادیده گرفته می‌شوند (کدها پس از 1 ساعت منقضی می‌شوند).
- از طریق `openclaw pairing list signal` و `openclaw pairing approve signal <CODE>` تأیید کنید.
- جفت‌سازی، تبادل توکن پیش‌فرض برای پیام‌های مستقیم Signal است. جزئیات: [جفت‌سازی](/fa/channels/pairing)
- فرستندگان فقط دارای UUID (از `sourceUuid`) به‌شکل `uuid:<id>` در `channels.signal.allowFrom` ذخیره می‌شوند.

گروه‌ها:

- `channels.signal.groupPolicy = open | allowlist | disabled`.
- `channels.signal.groupAllowFrom` تعیین می‌کند وقتی `allowlist` تنظیم شده است، کدام گروه‌ها یا فرستندگان می‌توانند پاسخ‌های گروهی را فعال کنند؛ ورودی‌ها می‌توانند شناسه‌های گروه Signal (خام، `group:<id>`، یا `signal:group:<id>`)، شماره‌تلفن‌های فرستنده، مقادیر `uuid:<id>`، یا `*` باشند.
- `channels.signal.groups["<group-id>" | "*"]` می‌تواند رفتار گروه را با `requireMention`، `tools`، و `toolsBySender` بازنویسی کند.
- برای بازنویسی‌های مختص هر حساب در پیکربندی‌های چندحسابی، از `channels.signal.accounts.<id>.groups` استفاده کنید.
- افزودن یک گروه Signal به فهرست مجاز از طریق `groupAllowFrom`، به‌خودی‌خود محدودسازی بر پایه اشاره را غیرفعال نمی‌کند. یک ورودی `channels.signal.groups["<group-id>"]` که به‌طور مشخص پیکربندی شده باشد، همه پیام‌های گروه را پردازش می‌کند، مگر اینکه `requireMention=true` تنظیم شده باشد.
- با `requireMention=true`، اشاره‌های بومی @ در Signal با استفاده از فراداده ساختاریافته اشاره و بر اساس شماره‌تلفن حساب ربات یا `accountUuid` تطبیق داده می‌شوند. `mentionPatterns` پیکربندی‌شده همچنان به‌عنوان جایگزین متن ساده باقی می‌مانند.
- نکته زمان اجرا: اگر `channels.signal` کاملاً وجود نداشته باشد، زمان اجرا برای بررسی‌های گروه به `groupPolicy="allowlist"` بازمی‌گردد (حتی اگر `channels.defaults.groupPolicy` تنظیم شده باشد).

گروه محدودشده بر پایه اشاره با زمینه محدود:

```json5
{
  channels: {
    signal: {
      account: "+15551234567",
      accountUuid: "bot-signal-uuid",
      groupPolicy: "allowlist",
      groupAllowFrom: ["group:<signal-group-id>"],
      historyLimit: 8,
      groups: {
        "<signal-group-id>": { requireMention: true },
      },
    },
  },
  messages: {
    groupChat: {
      mentionPatterns: ["\\bopenclaw\\b"],
    },
  },
}
```

پیام‌های مجاز گروه که به ربات اشاره نمی‌کنند بی‌پاسخ می‌مانند و فقط در پنجره محدود تاریخچه در انتظار نگه‌داری می‌شوند. هنگامی که بعداً یک اشاره بومی @ یا اشاره متنی جایگزین ربات را فعال کند، OpenClaw آن زمینه اخیر را در نظر می‌گیرد و به همان گروه پاسخ می‌دهد. بدنه پیوست‌های نادیده‌گرفته‌شده دانلود نمی‌شود؛ ممکن است آن‌ها فقط به‌شکل جای‌نگهدارهای فشرده رسانه در زمینه در انتظار ظاهر شوند.

## نحوه کار (رفتار)

- حالت بومی: `signal-cli` به‌عنوان یک دیمن اجرا می‌شود؛ Gateway رویدادها را از طریق SSE می‌خواند.
- حالت کانتینر: Gateway از طریق REST API ارسال و از طریق WebSocket دریافت می‌کند.
- پیام‌های ورودی در پوش پیام مشترک کانال نرمال‌سازی می‌شوند.
- پاسخ‌ها همیشه به همان شماره یا گروه بازگردانده می‌شوند.
- پاسخ به پیام‌های ورودی، هنگامی که بک‌اند مهر زمانی و نویسنده پیام ورودی را بپذیرد، شامل فراداده بومی نقل‌قول Signal است؛ اگر فراداده نقل‌قول وجود نداشته باشد یا رد شود، OpenClaw پاسخ را به‌صورت یک پیام عادی ارسال می‌کند.
- استفاده از نقل‌قول بومی را با `channels.signal.replyToMode = off | first | all | batched`، یا برای بازنویسی مختص نوع گفت‌وگو با `channels.signal.replyToModeByChatType.direct/group` پیکربندی کنید. مقادیر سطح حساب زیر `channels.signal.accounts.<id>` اولویت دارند.

## رسانه + محدودیت‌ها

- متن خروجی بر اساس `channels.signal.textChunkLimit` به قطعه‌ها تقسیم می‌شود (پیش‌فرض 4000).
- تقسیم اختیاری بر اساس خط جدید: `channels.signal.streaming.chunkMode="newline"` را تنظیم کنید تا پیش از تقسیم بر اساس طول، متن در خطوط خالی (مرز پاراگراف‌ها) تقسیم شود.
- پیوست‌ها پشتیبانی می‌شوند (base64 از `signal-cli` دریافت می‌شود).
- پیوست‌های یادداشت صوتی، هنگامی که `contentType` وجود ندارد، از نام فایل `signal-cli` به‌عنوان جایگزین MIME استفاده می‌کنند تا رونویسی صوت همچنان بتواند یادداشت‌های صوتی AAC را طبقه‌بندی کند.
- سقف پیش‌فرض رسانه: `channels.signal.mediaMaxMb` (پیش‌فرض 8).
- برای رد کردن دانلود رسانه در هر انتقالی، از `channels.signal.ignoreAttachments` استفاده کنید.
- زمینه تاریخچه گروه از `channels.signal.historyLimit` (یا `channels.signal.accounts.*.historyLimit`) استفاده می‌کند و در صورت نبود آن به `messages.groupChat.historyLimit` بازمی‌گردد. برای غیرفعال‌سازی، `0` را تنظیم کنید (پیش‌فرض 50).

## نشانگر تایپ + رسید خواندن

- **نشانگرهای تایپ**: OpenClaw سیگنال‌های تایپ را از طریق `signal-cli sendTyping` ارسال می‌کند و تا زمانی که پاسخ در حال اجرا است آن‌ها را تازه‌سازی می‌کند.
- **رسیدهای خواندن**: وقتی `channels.signal.sendReadReceipts` درست باشد، OpenClaw رسیدهای خواندن را برای پیام‌های مستقیم مجاز ارسال می‌کند.
- `signal-cli` رسیدهای خواندن گروه‌ها را ارائه نمی‌کند.

## واکنش‌های وضعیت چرخه عمر

`messages.statusReactions.enabled: true` را تنظیم کنید تا Signal چرخه واکنش مشترک صف‌شده/در حال فکر/ابزار/Compaction/انجام‌شده/خطا را در نوبت‌های ورودی نشان دهد. Signal از مهر زمانی پیام ورودی به‌عنوان هدف واکنش استفاده می‌کند؛ واکنش‌های گروه با شناسه گروه Signal به‌همراه فرستنده اصلی به‌عنوان نویسنده هدف ارسال می‌شوند.

واکنش‌های وضعیت همچنین به یک واکنش تأیید دریافت و یک `messages.ackReactionScope` منطبق (`direct`، `group-all`، `group-mentions`، یا `all`) نیاز دارند. برای غیرفعال‌سازی واکنش‌های وضعیت Signal، `channels.signal.reactionLevel: "off"` را تنظیم کنید.

Signal پس از وضعیت نهایی انجام‌شده/خطا، واکنش تأیید دریافت اولیه را بازیابی می‌کند.

## واکنش‌ها (ابزار پیام)

از `message action=react` به‌همراه `channel=signal` استفاده کنید.

- هدف‌ها: شماره E.164 یا UUID فرستنده (از `uuid:<id>` در خروجی جفت‌سازی استفاده کنید؛ UUID بدون پیشوند نیز کار می‌کند).
- `messageId` مهر زمانی Signal برای پیامی است که به آن واکنش نشان می‌دهید.
- واکنش‌های گروه به `targetAuthor` یا `targetAuthorUuid` نیاز دارند.

```text
message action=react channel=signal target=uuid:123e4567-e89b-12d3-a456-426614174000 messageId=1737630212345 emoji=🔥
message action=react channel=signal target=+15551234567 messageId=1737630212345 emoji=🔥 remove=true
message action=react channel=signal target=signal:group:<groupId> targetAuthor=uuid:<sender-uuid> messageId=1737630212345 emoji=✅
```

پیکربندی:

- `channels.signal.actions.reactions`: فعال/غیرفعال‌کردن کنش‌های واکنش (پیش‌فرض true).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (پیش‌فرض `minimal`).
  - `off`/`ack` واکنش‌های عامل را غیرفعال می‌کند (ابزار پیام `react` خطا می‌دهد).
  - `minimal`/`extensive` واکنش‌های عامل را فعال و سطح راهنمایی را تعیین می‌کند.
- بازنویسی‌های مختص هر حساب: `channels.signal.accounts.<id>.actions.reactions`، `channels.signal.accounts.<id>.reactionLevel`.

## واکنش‌های تأیید

درخواست‌های تأیید اجرای Signal و Plugin از بلوک‌های مسیریابی سطح بالای `approvals.exec` و `approvals.plugin` استفاده می‌کنند. Signal بلوک `channels.signal.execApprovals` ندارد.

- `👍` یک‌بار تأیید می‌کند.
- `👎` رد می‌کند.
- وقتی یک درخواست تأیید دائمی ارائه می‌دهد، از `/approve <id> allow-always` استفاده کنید.

حل واکنش تأیید به تأییدکنندگان صریح Signal از `channels.signal.allowFrom`، `channels.signal.defaultTo`، یا فیلدهای منطبق سطح حساب نیاز دارد. درخواست‌های تأیید اجرای مستقیم در همان گفت‌وگو همچنان می‌توانند جایگزین محلی تکراری `/approve` را بدون تأییدکنندگان صریح پنهان کنند؛ تأییدهای گروهی بدون تأییدکننده، جایگزین محلی را قابل‌مشاهده نگه می‌دارند.

## واکنش‌های پرسش

برای یک درخواست `ask_user` شامل یک پرسش غیرمحرمانه تک‌گزینشی و یک تا چهار گزینه، Signal در کنار برچسب گزینه‌ها `1️⃣` تا `4️⃣` را نشان می‌دهد. برای پاسخ، با شماره منطبق به درخواست تحویل‌شده واکنش نشان دهید. OpenClaw تأیید می‌کند که واکنش، پیام نوشته‌شده توسط ربات را هدف گرفته است، سپس شماره را از طریق Gateway به گزینه متعارف نگاشت می‌کند. لمس‌های منقضی یا تکراری نادیده گرفته می‌شوند. درخواست‌های چندپرسشی، چندگزینشی و متن آزاد همچنان فقط با پاسخ متنی قابل پاسخ‌گویی هستند؛ قواعد عادی پذیرش پیام مستقیم/گروه Signal، فرستنده را مجاز می‌کنند.

## هدف‌های تحویل (CLI/Cron)

- پیام‌های مستقیم: `signal:+15551234567` (یا E.164 ساده).
- پیام‌های مستقیم UUID: `uuid:<id>` (یا UUID بدون پیشوند).
- گروه‌ها: `signal:group:<groupId>`.
- نام‌های کاربری: `username:<name>` (اگر حساب Signal شما پشتیبانی کند).

## نام‌های مستعار

برای نام‌های پایدار در هدف‌های تکرارشونده Signal، نام مستعار پیکربندی کنید. نام‌های مستعار فقط پیکربندی سمت OpenClaw هستند؛ آن‌ها مخاطبان Signal را ایجاد یا ویرایش نمی‌کنند.

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
        jane: "uuid:123e4567-e89b-12d3-a456-426614174000",
        ops: "group:<groupId>",
      },
      defaultTo: "signal:me",
    },
  },
}
```

در هر جایی که هدف‌های تحویل Signal پذیرفته می‌شوند، از نام‌های مستعار استفاده کنید:

```bash
openclaw message send --channel signal --target signal:ops --message "استقرار کامل شد"
```

نام‌های مستعار مختص هر حساب، نام‌های مستعار سطح بالا را به ارث می‌برند و می‌توانند نام‌هایی را اضافه یا بازنویسی کنند:

```json5
{
  channels: {
    signal: {
      aliases: {
        me: "+15557654321",
      },
      accounts: {
        work: {
          aliases: {
            ops: "group:<workGroupId>",
          },
        },
      },
    },
  },
}
```

`openclaw directory peers list --channel signal` و `openclaw directory groups list --channel signal` نام‌های مستعار پیکربندی‌شده را فهرست می‌کنند. فهرست Signal مبتنی بر پیکربندی است؛ مخاطبان Signal را به‌صورت زنده واکشی نمی‌کند و حساب Signal را تغییر نمی‌دهد.

## عیب‌یابی

ابتدا این مراحل را اجرا کنید:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

سپس در صورت نیاز، وضعیت جفت‌سازی پیام مستقیم را تأیید کنید:

```bash
openclaw pairing list signal
```

خرابی‌های رایج:

- دیمن در دسترس است اما پاسخی نمی‌آید: `account`، `transport.kind`، نشانی انتقال، و حالت دریافت را بررسی کنید.
- پیام‌های مستقیم نادیده گرفته می‌شوند: فرستنده در انتظار تأیید جفت‌سازی است.
- پیام‌های گروه نادیده گرفته می‌شوند: محدودسازی بر پایه فرستنده/اشاره گروه، تحویل را مسدود می‌کند.
- خطاهای اعتبارسنجی پیکربندی پس از ویرایش: `openclaw doctor --fix` را اجرا کنید.
- Signal در تشخیص‌ها وجود ندارد: `channels.signal.enabled: true` را تأیید کنید.

بررسی‌های بیشتر:

```bash
openclaw pairing list signal
pgrep -af signal-cli
openclaw logs --plain --limit 500 | grep -i "signal" | tail -20
```

برای روند بررسی مشکل: [عیب‌یابی کانال‌ها](/fa/channels/troubleshooting).

## نکات امنیتی

- `signal-cli` کلیدهای حساب را به‌صورت محلی ذخیره می‌کند (معمولاً `~/.local/share/signal-cli/data/`).
- پیش از مهاجرت یا بازسازی سرور، از وضعیت حساب Signal پشتیبان تهیه کنید.
- مگر اینکه صریحاً دسترسی گسترده‌تر به پیام مستقیم بخواهید، `channels.signal.dmPolicy: "pairing"` را حفظ کنید.
- تأیید پیامکی فقط برای فرایندهای ثبت‌نام یا بازیابی لازم است، اما ازدست‌دادن کنترل شماره/حساب می‌تواند ثبت‌نام مجدد را پیچیده کند.

## مرجع پیکربندی (Signal)

پیکربندی کامل: [پیکربندی](/fa/gateway/configuration)

گزینه‌های ارائه‌دهنده:

- `channels.signal.enabled`: راه‌اندازی کانال را فعال/غیرفعال می‌کند.
- `channels.signal.account`: قالب E.164 برای حساب ربات.
- `channels.signal.accountUuid`: UUID اختیاری حساب ربات برای تشخیص بومی @mention و محافظت در برابر حلقه.
- `channels.signal.transport`: انتقال متعلق به حساب. برای پیش‌فرض‌های بومی مدیریت‌شده، آن را حذف کنید.
- `channels.signal.transport.kind`: `managed-native | external-native | container`.
- `channels.signal.transport.url`: برای `external-native` و `container` الزامی است؛ برای `managed-native` زمانی اختیاری است که نقطه پایانی اتصال آن با آدرس اتصال daemon متفاوت باشد.
- `channels.signal.transport.cliPath`: مسیر بومی مدیریت‌شده به `signal-cli`.
- `channels.signal.transport.configPath`: پوشه اختیاری بومی مدیریت‌شده `signal-cli --config`.
- `channels.signal.transport.httpHost`, `channels.signal.transport.httpPort`: آدرس اتصال daemon بومی مدیریت‌شده (پیش‌فرض `127.0.0.1:8080`).
- `channels.signal.transport.startupTimeoutMs`: زمان انتظار راه‌اندازی بومی مدیریت‌شده برحسب میلی‌ثانیه (حداقل 1000، حداکثر 120000؛ پیش‌فرض 30000).
- `channels.signal.transport.receiveMode`: `on-start | manual` بومی مدیریت‌شده.
- `channels.signal.ignoreAttachments`: بارگیری پیوست‌های ورودی را برای این حساب نادیده می‌گیرد.
- `channels.signal.transport.ignoreStories`: کلید تغییر وضعیت استوری بومی مدیریت‌شده.
- `channels.signal.sendReadReceipts`: رسیدهای خواندن را ارسال می‌کند.
- `channels.signal.dmPolicy`: `pairing | allowlist | open | disabled` (پیش‌فرض: جفت‌سازی).
- `channels.signal.allowFrom`: فهرست مجاز DM‏ (E.164 یا `uuid:<id>`). ‏`open` به `"*"` نیاز دارد. Signal نام کاربری ندارد؛ از شناسه‌های تلفن/UUID استفاده کنید.
- `channels.signal.aliases`: نام‌های مستعار سمت OpenClaw برای مقصدهای تحویل DM یا گروه.
- `channels.signal.groupPolicy`: `open | allowlist | disabled` (پیش‌فرض: فهرست مجاز).
- `channels.signal.groupAllowFrom`: فهرست مجاز گروه؛ شناسه‌های گروه Signal (خام، `group:<id>` یا `signal:group:<id>`)، شماره‌های E.164 فرستنده یا مقادیر `uuid:<id>` را می‌پذیرد.
- `channels.signal.groups`: بازنویسی‌های هر گروه با کلید شناسه گروه Signal (یا `"*"`). فیلدهای پشتیبانی‌شده: `requireMention`، `tools`، `toolsBySender`.
- `channels.signal.accounts.<id>.groups`: نسخه هر حساب از `channels.signal.groups` برای پیکربندی‌های چندحسابی.
- `channels.signal.accounts.<id>.aliases`: نام‌های مستعار هر حساب که با نام‌های مستعار سطح بالا ادغام می‌شوند.
- `channels.signal.replyToMode`: حالت نقل‌قول پاسخ بومی، `off | first | all | batched` (پیش‌فرض: `all`).
- `channels.signal.replyToModeByChatType.direct`, `channels.signal.replyToModeByChatType.group`: بازنویسی‌های نقل‌قول پاسخ بومی برای هر نوع گفت‌وگو.
- `channels.signal.accounts.<id>.replyToMode`, `channels.signal.accounts.<id>.replyToModeByChatType.direct`, `channels.signal.accounts.<id>.replyToModeByChatType.group`: بازنویسی‌های نقل‌قول پاسخ برای هر حساب.
- `channels.signal.historyLimit`: حداکثر تعداد پیام‌های گروه که به‌عنوان زمینه گنجانده می‌شوند (0 غیرفعال می‌کند).
- `channels.signal.dmHistoryLimit`: محدودیت تاریخچه DM برحسب نوبت‌های کاربر. بازنویسی‌های هر کاربر: `channels.signal.dms["<phone_or_uuid>"].historyLimit`.
- `channels.signal.textChunkLimit`: اندازه قطعه خروجی برحسب نویسه (پیش‌فرض 4000).
- `channels.signal.streaming.chunkMode`: ‏`length` (پیش‌فرض) یا `newline` برای تقسیم در خطوط خالی (مرز پاراگراف‌ها) پیش از قطعه‌بندی بر اساس طول.
- `channels.signal.mediaMaxMb`: سقف رسانه ورودی/خروجی برحسب MB (پیش‌فرض 8).
- `channels.signal.reactionLevel`: `off | ack | minimal | extensive` (پیش‌فرض `minimal`). [واکنش‌ها](#reactions-message-tool) را ببینید.
- `channels.signal.reactionNotifications`: `off | own | all | allowlist` (پیش‌فرض `own`) - زمانی که عامل از واکنش‌های ورودی دیگران مطلع می‌شود.
- `channels.signal.reactionAllowlist`: فرستندگانی که واکنش‌هایشان در حالت `reactionNotifications: "allowlist"` عامل را مطلع می‌کند.
- `channels.signal.streaming.block.enabled`, `channels.signal.streaming.block.coalesce`: کنترل‌های استریم حالت بلوکی که میان کانال‌ها مشترک‌اند. [استریم](/fa/concepts/streaming) را ببینید.

گزینه‌های سراسری مرتبط:

- `agents.entries.*.groupChat.mentionPatterns` (راهکار جایگزین متن ساده؛ @mentionهای بومی Signal زمانی از فراداده ساخت‌یافته تشخیص داده می‌شوند که هویت حساب ربات پیکربندی شده باشد).
- `messages.groupChat.mentionPatterns` (راهکار جایگزین سراسری).
- `channels.signal.responsePrefix` یا یک `responsePrefix` در سطح حساب.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) - همه کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت DM و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) - رفتار گفت‌وگوی گروهی و کنترل بر اساس اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) - مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) - مدل دسترسی و مقاوم‌سازی
