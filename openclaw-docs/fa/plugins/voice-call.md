---
read_when:
    - می‌خواهید از OpenClaw یک تماس صوتی خروجی برقرار کنید
    - در حال پیکربندی یا توسعه Plugin تماس صوتی هستید
    - در تلفن به صدای بلادرنگ یا رونویسی جریانی نیاز دارید
sidebarTitle: Voice call
summary: برقراری تماس‌های صوتی خروجی و پذیرش تماس‌های ورودی از طریق Twilio، Telnyx یا Plivo، با امکان اختیاری صدای بلادرنگ و رونویسی جریانی
title: Plugin تماس صوتی
x-i18n:
    generated_at: "2026-07-27T15:33:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 79f09f7b5cb99aace0960e283723d4f4408afa5f5dacd71f3c527fa62859f56f
    source_path: plugins/voice-call.md
    workflow: 16
---

تماس‌های صوتی برای OpenClaw از طریق یک Plugin: اعلان‌های خروجی، مکالمه‌های
چندنوبتی، صدای بلادرنگ تمام‌دوطرفه، رونویسی جریانی و تماس‌های ورودی با
سیاست‌های فهرست مجاز.

**ارائه‌دهندگان:** `mock` (توسعه، بدون شبکه)، `plivo` (Voice API + انتقال XML +
گفتار GetInput)، `telnyx` (Call Control v2)، `twilio` (Programmable Voice +
Media Streams).

<Note>
Plugin تماس صوتی **درون فرایند Gateway** اجرا می‌شود. اگر از یک
Gateway راه‌دور استفاده می‌کنید، Plugin را روی دستگاهی که Gateway را اجرا می‌کند
نصب و پیکربندی کنید، سپس Gateway را برای بارگذاری آن مجدداً راه‌اندازی کنید.
</Note>

## شروع سریع

<Steps>
  <Step title="نصب Plugin">
    <Tabs>
      <Tab title="از npm">
        ```bash
        openclaw plugins install @openclaw/voice-call
        ```
      </Tab>
      <Tab title="از یک پوشه محلی (توسعه)">
        ```bash
        PLUGIN_SRC=./path/to/local/voice-call-plugin
        openclaw plugins install "$PLUGIN_SRC"
        cd "$PLUGIN_SRC" && pnpm install
        ```
      </Tab>
    </Tabs>

    برای دنبال‌کردن برچسب انتشار فعلی، از بسته بدون نسخه استفاده کنید. فقط زمانی
    یک نسخه دقیق را ثابت کنید که به نصبی تکرارپذیر نیاز دارید. پس از آن Gateway
    را مجدداً راه‌اندازی کنید تا Plugin بارگذاری شود.

  </Step>
  <Step title="پیکربندی ارائه‌دهنده و Webhook">
    پیکربندی را زیر `plugins.entries.voice-call.config` تنظیم کنید (بخش
    [پیکربندی](#configuration) را در ادامه ببینید). حداقل موارد لازم: `provider`، اعتبارنامه‌های
    ارائه‌دهنده، `fromNumber` و یک نشانی URL عمومی و قابل‌دسترسی برای Webhook.
  </Step>
  <Step title="اعتبارسنجی راه‌اندازی">
    ```bash
    openclaw voicecall setup
    openclaw voicecall setup --json
    ```

    فعال‌بودن Plugin، اعتبارنامه‌های ارائه‌دهنده، در معرض دسترس بودن Webhook و
    فعال‌بودن فقط یکی از حالت‌های صوتی (`streaming` یا `realtime`) را بررسی می‌کند.

  </Step>
  <Step title="آزمایش دود">
    ```bash
    openclaw voicecall smoke
    openclaw voicecall smoke --to "+15555550123"
    ```

    هر دو به‌طور پیش‌فرض اجرای آزمایشی هستند. برای برقراری یک تماس اعلان خروجی
    کوتاه، `--yes` را اضافه کنید:

    ```bash
    openclaw voicecall smoke --to "+15555550123" --yes
    ```

  </Step>
</Steps>

<Warning>
برای Twilio، Telnyx و Plivo، راه‌اندازی باید به یک **نشانی URL عمومی Webhook** منتهی شود.
اگر `publicUrl`، نشانی URL تونل، نشانی URL مربوط به Tailscale یا مسیر جایگزین serve
به حلقه بازگشتی یا فضای شبکه خصوصی منتهی شود، راه‌اندازی به‌جای
اجرای ارائه‌دهنده‌ای که نمی‌تواند Webhookهای اپراتور را دریافت کند، شکست می‌خورد.
</Warning>

## پیکربندی

اگر `enabled: true` باشد اما ارائه‌دهنده انتخاب‌شده اعتبارنامه نداشته باشد، هنگام
راه‌اندازی Gateway هشداری درباره ناقص‌بودن راه‌اندازی همراه با کلیدهای مفقود ثبت می‌شود و
اجرای زمان‌اجرا نادیده گرفته می‌شود. فرمان‌ها، فراخوانی‌های RPC و ابزارهای عامل همچنان هنگام
استفاده، پیکربندی مفقود را دقیقاً برمی‌گردانند.

<Note>
اعتبارنامه‌های تماس صوتی SecretRef را می‌پذیرند. `plugins.entries.voice-call.config.twilio.authToken`، `plugins.entries.voice-call.config.realtime.providers.*.apiKey`، `plugins.entries.voice-call.config.streaming.providers.*.apiKey` و `plugins.entries.voice-call.config.tts.providers.*.apiKey` از طریق سطح استاندارد SecretRef رفع می‌شوند؛ [سطح اعتبارنامه SecretRef](/fa/reference/secretref-credential-surface) را ببینید.
</Note>

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio", // یا "telnyx" | "plivo" | "mock"
          fromNumber: "+15550001234", // یا TWILIO_FROM_NUMBER برای Twilio
          toNumber: "+15550005678",
          sessionScope: "per-phone", // per-phone | per-call
          numbers: {
            "+15550009999": {
              inboundGreeting: "کارت‌های Silver Fox، چطور می‌توانم کمک کنم؟",
              responseSystemPrompt: "شما متخصصی موجز در زمینه کارت‌های بیسبال هستید.",
              tts: {
                providers: {
                  openai: { speakerVoice: "alloy" },
                },
              },
            },
          },

          twilio: {
            accountSid: "ACxxxxxxxx",
            authToken: "...",
            // region: "ie1", // اختیاری: us1 | ie1 | au1؛ مقدار پیش‌فرض us1 است
          },
          telnyx: {
            apiKey: "...",
            connectionId: "...",
            // کلید عمومی Webhook مربوط به Telnyx از Mission Control Portal
            // (Base64؛ همچنین می‌توان آن را از طریق TELNYX_PUBLIC_KEY تنظیم کرد).
            publicKey: "...",
          },
          plivo: {
            authId: "MAxxxxxxxxxxxxxxxxxxxx",
            authToken: "...",
          },

          // سرور Webhook
          serve: {
            port: 3334,
            path: "/voice/webhook",
          },

          // امنیت Webhook (برای تونل‌ها/پراکسی‌ها توصیه می‌شود)
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
            trustedProxyIPs: ["100.64.0.1"],
          },

          // دسترسی عمومی (یکی را انتخاب کنید)
          // publicUrl: "https://example.ngrok.app/voice/webhook",
          // tunnel: { provider: "ngrok" },
          // tailscale: { mode: "funnel", path: "/voice/webhook" },

          outbound: {
            defaultMode: "notify", // notify | conversation
          },

          streaming: { enabled: true /* فقط Twilio؛ بخش رونویسی جریانی را ببینید */ },
          realtime: { enabled: false /* بخش مکالمه‌های صوتی بلادرنگ را ببینید */ },
        },
      },
    },
  },
}
```

### مرجع پیکربندی

کلیدهای سطح‌بالا زیر `plugins.entries.voice-call.config` که در بالا نشان داده نشده‌اند:

| کلید                             | پیش‌فرض      | توضیحات                                                                                              |
| ------------------------------- | ------------ | -------------------------------------------------------------------------------------------------- |
| `enabled`                       | `false`      | کلید اصلی روشن/خاموش.                                                                              |
| `inboundPolicy`                 | `"disabled"` | `disabled` \| `allowlist` \| `pairing` \| `open`. بخش [تماس‌های ورودی](#inbound-calls) را ببینید.             |
| `allowFrom`                     | `[]`         | فهرست مجاز E.164 برای `inboundPolicy: "allowlist"`.                                                  |
| `maxDurationSeconds`            | `300`        | سقف قطعی مدت هر تماس که صرف‌نظر از وضعیت پاسخ‌گویی اعمال می‌شود.                                 |
| `staleCallReaperSeconds`        | `120`        | بخش [پاک‌ساز تماس‌های کهنه](#stale-call-reaper) را ببینید. `0` آن را غیرفعال می‌کند.                                      |
| `silenceTimeoutMs`              | `800`        | تشخیص سکوت پایان گفتار برای جریان کلاسیک (غیربلادرنگ).                               |
| `transcriptTimeoutMs`           | `180000`     | حداکثر زمان انتظار برای رونویسی تماس‌گیرنده، پیش از صرف‌نظرکردن از یک نوبت.                                       |
| `ringTimeoutMs`                 | `30000`      | مهلت زنگ‌خوردن تماس‌های خروجی.                                                                   |
| `maxConcurrentCalls`            | `1`          | تماس‌های خروجی فراتر از این محدودیت رد می‌شوند.                                                     |
| `outbound.notifyHangupDelaySec` | `3`          | مدت انتظار برحسب ثانیه پس از TTS، پیش از قطع خودکار تماس در حالت اعلان.                                       |
| `skipSignatureVerification`     | `false`      | فقط برای آزمایش محلی؛ هرگز در محیط عملیاتی فعال نکنید.                                                    |
| `store`                         | تنظیم‌نشده        | مسیر پیش‌فرض `$OPENCLAW_STATE_DIR/voice-calls` را بازنویسی می‌کند (معمولاً `~/.openclaw/voice-calls`). |
| `agentId`                       | `"main"`     | عامل استفاده‌شده برای تولید پاسخ و ذخیره‌سازی نشست.                                            |
| `responseModel`                 | تنظیم‌نشده        | مدل پیش‌فرض پاسخ‌های کلاسیک (غیربلادرنگ) را بازنویسی می‌کند.                                  |
| `responseSystemPrompt`          | تولیدشده    | پرامپت سیستمی سفارشی برای پاسخ‌های کلاسیک.                                                        |
| `responseTimeoutMs`             | `30000`      | مهلت تولید پاسخ کلاسیک (میلی‌ثانیه).                                                      |

Twilio به‌طور پیش‌فرض از نقطه پایانی REST مربوط به US1 استفاده می‌کند. برای پردازش تماس‌ها
در یک Region غیرآمریکایی پشتیبانی‌شده، `twilio.region` را روی `ie1` یا `au1` تنظیم کنید و از اعتبارنامه‌های
همان Region استفاده کنید. به
[راهنمای REST API غیرآمریکایی Twilio](https://www.twilio.com/docs/global-infrastructure/using-the-twilio-rest-api-in-a-non-us-region) مراجعه کنید.

<AccordionGroup>
  <Accordion title="نکات مربوط به دسترسی و امنیت ارائه‌دهنده">
    - Twilio، Telnyx و Plivo همگی به یک نشانی URL برای Webhook نیاز دارند که **به‌صورت عمومی قابل‌دسترسی** باشد.
    - `mock` یک ارائه‌دهنده توسعه محلی است (بدون فراخوانی شبکه).
    - Telnyx به `telnyx.publicKey` (یا `TELNYX_PUBLIC_KEY`) نیاز دارد، مگر اینکه `skipSignatureVerification` برابر با true باشد.
    - `skipSignatureVerification` فقط برای آزمایش محلی است.
    - در سطح رایگان ngrok، `publicUrl` را دقیقاً روی نشانی URL مربوط به ngrok تنظیم کنید؛ اعتبارسنجی امضا همیشه اعمال می‌شود.
    - `tunnel.allowNgrokFreeTierLoopbackBypass: true` تنها زمانی Webhookهای Twilio با امضاهای نامعتبر را مجاز می‌کند که `tunnel.provider="ngrok"` باشد و `serve.bind` حلقه بازگشتی باشد (عامل محلی ngrok). فقط برای توسعه محلی.
    - نشانی‌های URL سطح رایگان ngrok ممکن است تغییر کنند یا رفتار میان‌صفحه‌ای اضافه کنند؛ اگر `publicUrl` تغییر کند، امضاهای Twilio نامعتبر می‌شوند. محیط عملیاتی: یک دامنه پایدار یا funnel مربوط به Tailscale را ترجیح دهید.

  </Accordion>
  <Accordion title="سقف‌های اتصال جریانی">
    - `streaming.preStartTimeoutMs` (پیش‌فرض `5000`) سوکت‌هایی را می‌بندد که هرگز یک فریم معتبر `start` ارسال نمی‌کنند.
    - `streaming.maxPendingConnections` (پیش‌فرض `32`) تعداد کل سوکت‌های احرازنشده پیش از شروع را محدود می‌کند.
    - `streaming.maxPendingConnectionsPerIp` (پیش‌فرض `4`) تعداد سوکت‌های احرازنشده پیش از شروع را به‌ازای هر IP مبدأ محدود می‌کند.
    - `streaming.maxConnections` (پیش‌فرض `128`) تعداد همه سوکت‌های باز جریان رسانه (در انتظار + فعال) را محدود می‌کند.

  </Accordion>
  <Accordion title="مهاجرت‌های پیکربندی قدیمی">
    تجزیه پیکربندی این کلیدهای قدیمی را به‌طور خودکار نرمال‌سازی می‌کند و هشداری
    شامل مسیر جایگزین ثبت می‌کند؛ این لایه سازگاری در یک انتشار آینده
    (`2026.6.0`) حذف می‌شود، بنابراین برای بازنویسی پیکربندی ثبت‌شده
    به شکل متعارف، `openclaw doctor --fix` را اجرا کنید:

    - `provider: "log"` → `provider: "mock"`
    - `twilio.from` → `fromNumber`
    - `streaming.sttProvider` → `streaming.provider`
    - `streaming.openaiApiKey` → `streaming.providers.openai.apiKey`
    - `streaming.sttModel` → `streaming.providers.openai.model`
    - `streaming.silenceDurationMs` → `streaming.providers.openai.silenceDurationMs`
    - `streaming.vadThreshold` → `streaming.providers.openai.vadThreshold`
    - `realtime.agentContext.includeSystemPrompt` حذف شده است (زمینه بلادرنگ اکنون از پرامپت تولیدشده عامل استفاده می‌کند)

  </Accordion>
</AccordionGroup>

## دامنه نشست

به‌طور پیش‌فرض، تماس صوتی از `sessionScope: "per-phone"` استفاده می‌کند تا تماس‌های تکراری از
همان تماس‌گیرنده، حافظه مکالمه را حفظ کنند. هنگامی که
هر تماس اپراتور باید با زمینه‌ای تازه آغاز شود، `sessionScope: "per-call"` را تنظیم کنید؛ برای مثال در جریان‌های
پذیرش، رزرو، IVR یا پل Google Meet که ممکن است یک شماره تلفن یکسان
نماینده جلسه‌های متفاوت باشد.

تماس صوتی کلیدهای نشست تولیدشده را در فضای نام عامل پیکربندی‌شده
(`agent:<agentId>:voice:*`) ذخیره می‌کند. کلیدهای صریح خام یکپارچه‌سازی در
همان فضای نام رفع می‌شوند: یک کلید متعارف `agent:<configuredAgentId>:*` مالک
خود را حفظ می‌کند و از نام مستعار `session.mainKey`/دامنه سراسری هسته پیروی می‌کند؛ ورودی خارجی یا
بدشکل `agent:*` به‌صورت یک کلید مات زیر عامل پیکربندی‌شده دامنه‌بندی
می‌شود؛ `global` و `unknown` همچنان نگهبان‌های سراسری باقی می‌مانند.

## مکالمه‌های صوتی بلادرنگ

`realtime` یک ارائه‌دهنده صدای بلادرنگ تمام‌دوطرفه را برای صدای زنده تماس انتخاب می‌کند.
این گزینه از `streaming` جدا است که فقط صدا را به ارائه‌دهندگان
رونویسی بلادرنگ هدایت می‌کند.

<Warning>
`realtime.enabled` را نمی‌توان با `streaming.enabled` ترکیب کرد. برای هر تماس یک
حالت صوتی انتخاب کنید.
</Warning>

رفتار فعلی زمان‌اجرا:

- `realtime.enabled` برای Twilio و Telnyx پشتیبانی می‌شود.
- `realtime.provider` اختیاری است. اگر تنظیم نشده باشد، Voice Call از نخستین ارائه‌دهنده ثبت‌شده صدای بلادرنگ استفاده می‌کند.
- ارائه‌دهندگان همراه صدای بلادرنگ: Google Gemini Live ‏(`google`) و OpenAI ‏(`openai`) که توسط Pluginهای ارائه‌دهنده خود ثبت می‌شوند.
- پیکربندی خام متعلق به ارائه‌دهنده در `realtime.providers.<providerId>` قرار دارد.
- Voice Call به‌طور پیش‌فرض ابزار بلادرنگ مشترک `openclaw_agent_consult` را ارائه می‌کند. مدل بلادرنگ می‌تواند هنگامی که تماس‌گیرنده استدلال عمیق‌تر، اطلاعات جاری یا ابزارهای عادی OpenClaw را درخواست می‌کند، آن را فراخوانی کند.
- `realtime.consultPolicy` به‌صورت اختیاری راهنمایی‌هایی درباره زمان فراخوانی `openclaw_agent_consult` توسط مدل بلادرنگ اضافه می‌کند.
- `realtime.agentContext.enabled` به‌طور پیش‌فرض غیرفعال است. وقتی فعال باشد، Voice Call هنگام راه‌اندازی نشست، یک هویت عامل با اندازه محدود و کپسولی از فایل‌های منتخب فضای کاری را به دستورالعمل‌های ارائه‌دهنده بلادرنگ تزریق می‌کند.
- `realtime.fastContext.enabled` به‌طور پیش‌فرض غیرفعال است. وقتی فعال باشد، Voice Call ابتدا زمینه حافظه/نشست نمایه‌سازی‌شده را برای پرسش مشاوره جست‌وجو می‌کند و آن قطعه‌ها را در محدوده `realtime.fastContext.timeoutMs` به مدل بلادرنگ برمی‌گرداند؛ تنها در صورتی به عامل کامل مشاوره بازمی‌گردد که `realtime.fastContext.fallbackToConsult` برابر با true باشد.
- اگر `realtime.provider` به ارائه‌دهنده‌ای ثبت‌نشده اشاره کند، یا هیچ ارائه‌دهنده صدای بلادرنگی ثبت نشده باشد، Voice Call به‌جای از کار انداختن کل Plugin، هشداری ثبت می‌کند و از رسانه بلادرنگ صرف‌نظر می‌کند.
- وقتی `realtime.enabled` برابر با true است، `inboundPolicy` نباید `"disabled"` باشد؛ `validateProviderConfig` این ترکیب را رد می‌کند.
- کلیدهای نشست مشاوره، در صورت موجود بودن، از نشست ذخیره‌شده تماس دوباره استفاده می‌کنند و سپس به `sessionScope` پیکربندی‌شده بازمی‌گردند (`per-phone` به‌طور پیش‌فرض، یا `per-call` برای تماس‌های ایزوله).

### خط‌مشی ابزار

`realtime.toolPolicy` اجرای مشاوره را کنترل می‌کند:

| خط‌مشی           | رفتار                                                                                                                                 |
| ---------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | ابزار مشاوره را ارائه می‌کند و عامل عادی را به `read`، `web_search`، `web_fetch`، `x_search`، `memory_search` و `memory_get` محدود می‌کند. |
| `owner`          | ابزار مشاوره را ارائه می‌کند و به عامل عادی اجازه می‌دهد از خط‌مشی معمول ابزار عامل استفاده کند.                                                      |
| `none`           | ابزار مشاوره را ارائه نمی‌کند. `realtime.tools` سفارشی همچنان بدون تغییر به ارائه‌دهنده بلادرنگ منتقل می‌شوند.                               |

`realtime.consultPolicy` فقط دستورالعمل‌های مدل بلادرنگ را کنترل می‌کند:

| خط‌مشی        | راهنمایی                                                                                        |
| ------------- | ----------------------------------------------------------------------------------------------- |
| `auto`        | پرامپت پیش‌فرض را حفظ می‌کند و تصمیم‌گیری درباره زمان فراخوانی ابزار مشاوره را به ارائه‌دهنده می‌سپارد.              |
| `substantive` | پیوندهای ساده مکالمه را مستقیماً پاسخ می‌دهد و پیش از ارائه واقعیت‌ها، استفاده از حافظه و ابزارها یا به‌کارگیری زمینه، مشاوره می‌کند. |
| `always`      | پیش از هر پاسخ محتوایی مشاوره می‌کند.                                                        |

### زمینه صوتی عامل

زمانی `realtime.agentContext` را فعال کنید که پل صوتی باید بدون تحمیل رفت‌وبرگشت کامل
مشاوره با عامل در نوبت‌های عادی، مانند عامل پیکربندی‌شده OpenClaw به نظر برسد.
کپسول زمینه هنگام ایجاد نشست بلادرنگ یک‌بار اضافه می‌شود، بنابراین برای هر نوبت
تأخیر جداگانه‌ای ایجاد نمی‌کند. فراخوانی‌های
`openclaw_agent_consult` همچنان عامل کامل OpenClaw را اجرا می‌کنند و باید
برای کار با ابزارها، اطلاعات جاری، جست‌وجوهای حافظه یا وضعیت فضای کاری استفاده شوند.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          agentId: "main",
          realtime: {
            enabled: true,
            provider: "google",
            toolPolicy: "safe-read-only",
            consultPolicy: "substantive",
            agentContext: {
              enabled: true,
              maxChars: 6000,
              includeIdentity: true,
              includeWorkspaceFiles: true,
              files: ["SOUL.md", "IDENTITY.md", "USER.md"],
            },
          },
        },
      },
    },
  },
}
```

### نمونه‌های ارائه‌دهنده بلادرنگ

<Tabs>
  <Tab title="Google Gemini Live">
    مقادیر پیش‌فرض: کلید API از `realtime.providers.google.apiKey`، `GEMINI_API_KEY`
    یا `GOOGLE_API_KEY`؛ مدل `gemini-3.1-flash-live-preview`؛
    صدا `Kore`. ‏`sessionResumption` و `contextWindowCompression`
    برای تماس‌های طولانی‌تر و قابل اتصال مجدد، به‌طور پیش‌فرض فعال‌اند. برای تنظیم نوبت‌گیری سریع‌تر
    در صدای تلفنی از `silenceDurationMs`،
    `startSensitivity` و `endSensitivity` استفاده کنید.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              provider: "twilio",
              inboundPolicy: "allowlist",
              allowFrom: ["+15550005678"],
              realtime: {
                enabled: true,
                provider: "google",
                instructions: "کوتاه صحبت کن. پیش از استفاده از ابزارهای عمیق‌تر، openclaw_agent_consult را فراخوانی کن.",
                toolPolicy: "safe-read-only",
                consultPolicy: "substantive",
                consultThinkingLevel: "low",
                consultFastMode: true,
                agentContext: { enabled: true },
                providers: {
                  google: {
                    apiKey: "${GEMINI_API_KEY}",
                    model: "gemini-3.1-flash-live-preview",
                    speakerVoice: "Kore",
                    silenceDurationMs: 500,
                    startSensitivity: "high",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="OpenAI">
    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              realtime: {
                enabled: true,
                provider: "openai",
                providers: {
                  openai: { apiKey: "${OPENAI_API_KEY}" },
                },
              },
            },
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

برای گزینه‌های صدای بلادرنگ مختص هر ارائه‌دهنده، به [ارائه‌دهنده Google](/fa/providers/google) و
[ارائه‌دهنده OpenAI](/fa/providers/openai) مراجعه کنید.

## رونویسی جریانی

`streaming`، Twilio Media Streams را به یک ارائه‌دهنده رونویسی بلادرنگ متصل می‌کند.
مسیر جریانی کلاسیک به `provider: "twilio"` نیاز دارد؛ پیکربندی با
Telnyx، Plivo یا mock رد می‌شود. صدای زنده Telnyx در عوض از مسیر
`realtime.enabled` با احراز هویت جداگانه استفاده می‌کند.

رفتار فعلی زمان اجرا:

- `streaming.provider` اختیاری است. اگر تنظیم نشده باشد، Voice Call از نخستین ارائه‌دهنده ثبت‌شده رونویسی بلادرنگ استفاده می‌کند.
- ارائه‌دهندگان همراه رونویسی بلادرنگ: Deepgram ‏(`deepgram`)، ElevenLabs ‏(`elevenlabs`)، Mistral ‏(`mistral`)، OpenAI ‏(`openai`) و xAI ‏(`xai`) که توسط Pluginهای ارائه‌دهنده خود ثبت می‌شوند.
- پیکربندی خام متعلق به ارائه‌دهنده در `streaming.providers.<providerId>` قرار دارد.
- پس از اینکه Twilio پیام پذیرفته‌شده `start` جریان را ارسال می‌کند، Voice Call بلافاصله جریان را ثبت می‌کند، در زمان اتصال ارائه‌دهنده رسانه ورودی را از طریق ارائه‌دهنده رونویسی در صف قرار می‌دهد و خوشامدگویی اولیه را تنها پس از آماده‌شدن رونویسی بلادرنگ آغاز می‌کند.
- اگر `streaming.provider` به ارائه‌دهنده‌ای ثبت‌نشده اشاره کند، یا هیچ ارائه‌دهنده‌ای ثبت نشده باشد، Voice Call به‌جای از کار انداختن کل Plugin، هشداری ثبت می‌کند و از پخش جریانی رسانه صرف‌نظر می‌کند.

### نمونه‌های ارائه‌دهنده جریانی

<Tabs>
  <Tab title="OpenAI">
    مقادیر پیش‌فرض: کلید API ‏`streaming.providers.openai.apiKey` یا
    `OPENAI_API_KEY`؛ مدل `gpt-4o-transcribe`؛ ‏`silenceDurationMs: 800`؛
    `vadThreshold: 0.5`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "openai",
                streamPath: "/voice/stream",
                providers: {
                  openai: {
                    apiKey: "sk-...", // اگر OPENAI_API_KEY تنظیم شده باشد اختیاری است
                    model: "gpt-4o-transcribe",
                    silenceDurationMs: 800,
                    vadThreshold: 0.5,
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="xAI">
    مقادیر پیش‌فرض: کلید API ‏`streaming.providers.xai.apiKey` یا `XAI_API_KEY` (اگر
    هیچ‌کدام تنظیم نشده باشند، به نمایه احراز هویت OAuth متعلق به xAI بازمی‌گردد)؛ نقطه پایانی
    `wss://api.x.ai/v1/stt`؛ کدگذاری `mulaw`؛ نرخ نمونه‌برداری `8000`؛
    `endpointingMs: 800`؛ ‏`interimResults: true`.

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                streamPath: "/voice/stream",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}", // اگر XAI_API_KEY تنظیم شده باشد اختیاری است
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## TTS برای تماس‌ها

Voice Call برای گفتار جریانی در تماس‌ها از پیکربندی اصلی `tts`
استفاده می‌کند. می‌توان آن را در پیکربندی Plugin با **همان ساختار** بازنویسی کرد —
این پیکربندی به‌صورت عمیق با `tts` ادغام می‌شود.

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

<Warning>
**گفتار Microsoft برای تماس‌های صوتی نادیده گرفته می‌شود.** هم‌گذاری گفتار تلفنی به
ارائه‌دهنده‌ای نیاز دارد که خروجی ویژه تلفن را پیاده‌سازی کند؛ ارائه‌دهنده گفتار Microsoft
چنین قابلیتی ندارد، بنابراین برای تماس‌ها از آن صرف‌نظر می‌شود و در عوض سایر ارائه‌دهندگان
در زنجیره بازگشت امتحان می‌شوند.
</Warning>

نکات رفتاری:

- کلیدهای قدیمی `tts.<provider>` درون پیکربندی Plugin ‏(`openai`، `elevenlabs`، `microsoft`، `edge`) توسط `openclaw doctor --fix` اصلاح می‌شوند؛ پیکربندی ثبت‌شده باید از `tts.providers.<provider>` استفاده کند.
- هنگامی که پخش جریانی رسانه Twilio فعال است، از TTS اصلی استفاده می‌شود؛ در غیر این صورت، تماس‌ها به صداهای بومی ارائه‌دهنده بازمی‌گردند.
- اگر جریان رسانه Twilio از قبل فعال باشد، Voice Call به `<Say>` در TwiML بازنمی‌گردد. اگر در این وضعیت TTS تلفنی در دسترس نباشد، درخواست پخش به‌جای ترکیب دو مسیر پخش ناموفق می‌شود.
- وقتی TTS تلفنی به ارائه‌دهنده ثانویه بازمی‌گردد، Voice Call برای اشکال‌زدایی هشداری همراه با زنجیره ارائه‌دهندگان (`from`، `to`، `attempts`) ثبت می‌کند.
- وقتی ورود هم‌زمان صدای تماس‌گیرنده در Twilio یا برچیدن جریان، صف TTS در انتظار را پاک می‌کند، درخواست‌های پخش صف‌شده تعیین تکلیف می‌شوند تا تماس‌گیرندگانی که منتظر تکمیل پخش هستند معطل نمانند.

### نمونه‌های TTS

<Tabs>
  <Tab title="فقط TTS هسته">
```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "alloy" },
    },
  },
}
```
  </Tab>
  <Tab title="بازنویسی با ElevenLabs (فقط تماس‌ها)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "elevenlabs_key",
                speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
                modelId: "eleven_multilingual_v2",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
  <Tab title="بازنویسی مدل OpenAI (ادغام عمیق)">
```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tts: {
            providers: {
              openai: {
                model: "gpt-4o-mini-tts",
                speakerVoice: "marin",
              },
            },
          },
        },
      },
    },
  },
}
```
  </Tab>
</Tabs>

## تماس‌های ورودی

سیاست ورودی به‌طور پیش‌فرض `disabled` است. برای فعال‌کردن تماس‌های ورودی، تنظیم کنید:

```json5
{
  inboundPolicy: "allowlist",
  allowFrom: ["+15550001234"],
  inboundGreeting: "سلام! چطور می‌توانم کمک کنم؟",
}
```

<Warning>
`inboundPolicy: "allowlist"` یک غربالگری کم‌اطمینان شناسه تماس‌گیرنده است. Plugin
مقدار `From` ارائه‌شده توسط ارائه‌دهنده را نرمال‌سازی می‌کند و آن را با `allowFrom` مقایسه می‌کند.
تأیید Webhook، تحویل توسط ارائه‌دهنده و یکپارچگی محموله را احراز می‌کند،
اما مالکیت شماره تماس‌گیرنده PSTN/VoIP را **اثبات نمی‌کند**. با
`allowFrom` به‌عنوان فیلتر شناسه تماس‌گیرنده رفتار کنید، نه هویت قوی تماس‌گیرنده.
</Warning>

پاسخ‌های خودکار از سامانه عامل استفاده می‌کنند. آن‌ها را با `responseModel`،
`responseSystemPrompt` و `responseTimeoutMs` تنظیم کنید.

### مسیریابی به‌ازای هر شماره

وقتی یک Plugin تماس صوتی، تماس‌های چند شماره تلفن را دریافت می‌کند و هر شماره باید مانند خطی متفاوت رفتار کند، از `numbers` استفاده کنید. برای مثال،
یک شماره می‌تواند از یک دستیار شخصی خودمانی استفاده کند، درحالی‌که شماره‌ای دیگر از یک شخصیت
تجاری، عامل پاسخ‌گوی متفاوت و صدای TTS متفاوتی استفاده می‌کند.

مسیرها بر اساس شماره شماره‌گیری‌شده `To` که ارائه‌دهنده ارائه می‌کند انتخاب می‌شوند. کلیدها باید
شماره‌های E.164 باشند. هنگام ورود تماس، تماس صوتی مسیر منطبق را
یک‌بار تعیین می‌کند، مسیر منطبق را در رکورد تماس ذخیره می‌کند و همان
پیکربندی مؤثر را برای خوشامدگویی، مسیر کلاسیک پاسخ خودکار، مسیر
مشاوره بلادرنگ و پخش TTS دوباره به‌کار می‌برد. اگر هیچ مسیری منطبق نباشد، پیکربندی سراسری تماس صوتی
استفاده می‌شود. تماس‌های خروجی از `numbers` استفاده نمی‌کنند؛ هنگام آغاز تماس، مقصد خروجی،
پیام و نشست را به‌صراحت ارسال کنید.

بازنویسی مسیر درحال‌حاضر از موارد زیر پشتیبانی می‌کند:

- `inboundGreeting`
- `tts`
- `agentId`
- `responseModel`
- `responseSystemPrompt`
- `responseTimeoutMs`

مقدار مسیر `tts` به‌صورت عمیق روی پیکربندی سراسری `tts` تماس صوتی ادغام می‌شود، بنابراین
معمولاً می‌توانید فقط صدای ارائه‌دهنده را بازنویسی کنید:

```json5
{
  inboundGreeting: "سلام از خط اصلی.",
  responseSystemPrompt: "شما دستیار صوتی پیش‌فرض هستید.",
  tts: {
    provider: "openai",
    providers: {
      openai: { speakerVoice: "coral" },
    },
  },
  numbers: {
    "+15550001111": {
      inboundGreeting: "Silver Fox Cards، چطور می‌توانم کمک کنم؟",
      responseSystemPrompt: "شما متخصصی مختصرگو در زمینه کارت‌های بیسبال هستید.",
      tts: {
        providers: {
          openai: { speakerVoice: "alloy" },
        },
      },
    },
  },
}
```

### قرارداد خروجی گفتاری

برای پاسخ‌های خودکار، تماس صوتی قراردادی سخت‌گیرانه برای خروجی گفتاری به
اعلان سامانه می‌افزاید که پاسخ JSON با `{"spoken":"..."}` را الزامی می‌کند. تماس صوتی
متن گفتار را به‌طور تدافعی استخراج می‌کند:

- محموله‌هایی را که به‌عنوان محتوای استدلال/خطا علامت‌گذاری شده‌اند نادیده می‌گیرد.
- JSON مستقیم، JSON حصارشده یا کلیدهای درون‌خطی `"spoken"` را تجزیه می‌کند.
- به متن ساده بازمی‌گردد و بندهای آغازین احتمالی برنامه‌ریزی/فرااطلاعات را حذف می‌کند.

این کار پخش گفتاری را بر متن خطاب به تماس‌گیرنده متمرکز نگه می‌دارد و از نشت
متن برنامه‌ریزی به صدا جلوگیری می‌کند.

### رفتار آغاز مکالمه

برای تماس‌های خروجی `conversation`، مدیریت نخستین پیام به وضعیت پخش زنده
وابسته است:

- پاک‌سازی صف هنگام قطع‌کردن گفتار و پاسخ خودکار فقط تا زمانی سرکوب می‌شوند که خوشامدگویی اولیه فعالانه در حال پخش باشد.
- اگر پخش اولیه ناموفق باشد، تماس به `listening` بازمی‌گردد و پیام اولیه برای تلاش مجدد در صف باقی می‌ماند.
- پخش اولیه برای استریم Twilio هنگام اتصال استریم، بدون تأخیر اضافی آغاز می‌شود.
- قطع‌کردن گفتار، پخش فعال را متوقف می‌کند و ورودی‌های TTS مربوط به Twilio را که در صف هستند اما هنوز پخش نشده‌اند پاک می‌کند. ورودی‌های پاک‌شده به‌عنوان ردشده پایان می‌یابند تا منطق پاسخ بعدی بتواند بدون انتظار برای صدایی که هرگز پخش نخواهد شد ادامه دهد.
- مکالمات صوتی بلادرنگ از نوبت آغازین خود استریم بلادرنگ استفاده می‌کنند. تماس صوتی برای آن پیام اولیه، به‌روزرسانی قدیمی TwiML با `<Say>` ارسال **نمی‌کند**؛ بنابراین نشست‌های خروجی `<Connect><Stream>` متصل باقی می‌مانند.

### مهلت قطع اتصال استریم Twilio

وقتی یک استریم رسانه‌ای Twilio قطع می‌شود، تماس صوتی پیش از
پایان‌دادن خودکار تماس **2000 ms** منتظر می‌ماند:

- اگر استریم در این بازه دوباره متصل شود، پایان خودکار لغو می‌شود.
- اگر پس از دوره مهلت هیچ استریمی دوباره ثبت نشود، تماس پایان می‌یابد تا از گیرکردن تماس‌های فعال جلوگیری شود.

## پاک‌ساز تماس‌های کهنه

از `staleCallReaperSeconds` (پیش‌فرض **120**) برای پایان‌دادن به تماس‌هایی استفاده کنید که هرگز
پاسخ داده نمی‌شوند و هرگز به وضعیت مکالمه زنده نمی‌رسند؛ برای مثال تماس‌های حالت اعلان
که ارائه‌دهنده هرگز Webhook پایانی آن‌ها را تحویل نمی‌دهد. برای غیرفعال‌کردن، آن را روی `0` تنظیم کنید.

پاک‌ساز هر 30 ثانیه اجرا می‌شود و فقط تماس‌هایی را پایان می‌دهد که فاقد
مُهر زمانی `answeredAt` هستند و از قبل در وضعیت پایانی یا زنده
(`speaking`/`listening`) قرار ندارند؛ بنابراین مکالمات پاسخ‌داده‌شده هرگز توسط
این زمان‌سنج پاک‌سازی نمی‌شوند. `maxDurationSeconds` (پیش‌فرض 300) محدودیت جداگانه‌ای است که
تماس‌های پاسخ‌داده‌شده‌ای را که بیش‌ازحد طول می‌کشند پایان می‌دهد.

برای جریان‌های اعلان‌محور که اپراتورها ممکن است Webhookهای زنگ‌خوردن/پاسخ را
با تأخیر تحویل دهند، `staleCallReaperSeconds` را از مقدار پیش‌فرض بالاتر ببرید تا
تماس‌های کند اما عادی زودهنگام پاک‌سازی نشوند؛ `120`-`300` ثانیه بازه‌ای معقول برای محیط عملیاتی
است.

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          maxDurationSeconds: 300,
          staleCallReaperSeconds: 120,
        },
      },
    },
  },
}
```

## امنیت Webhook

وقتی یک پراکسی یا تونل جلوی Gateway قرار دارد، Plugin نشانی عمومی را برای
تأیید امضا بازسازی می‌کند. این گزینه‌ها مشخص می‌کنند کدام سرآیندهای
ارسال‌شده قابل اعتمادند:

<ParamField path="webhookSecurity.allowedHosts" type="string[]">
  میزبان‌های مجاز از سرآیندهای ارسال را مشخص می‌کند.
</ParamField>
<ParamField path="webhookSecurity.trustForwardingHeaders" type="boolean">
  بدون فهرست مجاز به سرآیندهای ارسال‌شده اعتماد می‌کند.
</ParamField>
<ParamField path="webhookSecurity.trustedProxyIPs" type="string[]">
  فقط زمانی به سرآیندهای ارسال‌شده اعتماد می‌کند که IP راه‌دور درخواست با فهرست منطبق باشد.
</ParamField>

محافظت‌های بیشتر:

- **محافظت در برابر بازپخش** Webhook برای Twilio، Telnyx و Plivo فعال است. درخواست‌های معتبر Webhook که بازپخش شده‌اند تأیید می‌شوند، اما اثرات جانبی آن‌ها اجرا نمی‌شود.
- نوبت‌های مکالمه Twilio در فراخوان‌های بازگشتی `<Gather>` یک توکن مختص هر نوبت دارند؛ بنابراین فراخوان‌های بازگشتی گفتار کهنه/بازپخش‌شده نمی‌توانند یک نوبت رونوشت جدیدترِ در انتظار را برآورده کنند.
- وقتی سرآیندهای امضای الزامی ارائه‌دهنده وجود ندارند، درخواست‌های احرازنشده Webhook پیش از خواندن بدنه رد می‌شوند.
- Webhook تماس صوتی پیش از تأیید امضا از پروفایل مشترک خواندن بدنه پیش از احراز (حداکثر بدنه 64 KB، مهلت خواندن 5 ثانیه) به‌همراه سقف درحال‌اجرای مختص هر کلید (به‌طور پیش‌فرض 8 درخواست هم‌زمان برای هر کلید) استفاده می‌کند.

نمونه با یک میزبان عمومی پایدار:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          webhookSecurity: {
            allowedHosts: ["voice.example.com"],
          },
        },
      },
    },
  },
}
```

## CLI

```bash
openclaw voicecall call --to "+15555550123" --message "سلام از OpenClaw"
openclaw voicecall start --to "+15555550123"   # نام مستعار call
openclaw voicecall continue --call-id <id> --message "سؤالی دارید؟"
openclaw voicecall speak --call-id <id> --message "یک لحظه"
openclaw voicecall dtmf --call-id <id> --digits "ww123456#"
openclaw voicecall end --call-id <id>
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw voicecall latency                      # خلاصه‌سازی تأخیر نوبت‌ها از گزارش‌ها
openclaw voicecall expose --mode funnel
```

وقتی Gateway از قبل در حال اجرا است، فرمان‌های عملیاتی `voicecall`
به زمان اجرای تماس صوتی تحت مالکیت Gateway واگذار می‌شوند تا CLI یک
سرور Webhook دوم را مقید نکند. اگر هیچ Gateway در دسترس نباشد، فرمان‌ها به
زمان اجرای مستقل CLI بازمی‌گردند.

`latency`، `calls.jsonl` را از مسیر ذخیره‌سازی پیش‌فرض تماس صوتی می‌خواند. برای
اشاره به گزارشی متفاوت از `--file <path>` و برای محدودکردن
تحلیل به آخرین N رکورد (پیش‌فرض 200) از `--last <n>` استفاده کنید. خروجی شامل کمینه/بیشینه/میانگین،
p50 و p95 برای تأخیر نوبت و زمان‌های انتظار برای شنیدن است.

## ابزار عامل

نام ابزار: `voice_call`.

| عملیات          | آرگومان‌ها                                       |
| --------------- | ------------------------------------------ |
| `initiate_call` | `message`, `to?`, `mode?`, `dtmfSequence?` |
| `continue_call` | `callId`, `message`                        |
| `speak_to_user` | `callId`, `message`                        |
| `send_dtmf`     | `callId`, `digits`                         |
| `end_call`      | `callId`                                   |
| `get_status`    | `callId`                                   |

Plugin تماس صوتی همراه با یک Skill عامل متناظر عرضه می‌شود.

## RPC در Gateway

| روش                      | آرگومان‌ها                                                             | توضیحات                                                                     |
| --------------------------- | ---------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `voicecall.initiate`        | `to?`, `message`, `mode?`, `sessionKey?`, `requesterSessionKey?` | وقتی `to` حذف شده باشد، به پیکربندی `toNumber` برمی‌گردد.                     |
| `voicecall.start`           | `to`, `message?`, `mode?`, `dtmfSequence?`, `sessionKey?`        | همانند `initiate` است، اما `dtmfSequence` پیش از اتصال را نیز می‌پذیرد.           |
| `voicecall.continue`        | `callId`, `message`                                              | تا پایان نوبت مسدود می‌ماند؛ رونوشت را برمی‌گرداند.                   |
| `voicecall.continue.start`  | `callId`, `message`                                              | گونهٔ ناهمگام: بلافاصله یک `operationId` برمی‌گرداند.                      |
| `voicecall.continue.result` | `operationId`                                                    | برای دریافت نتیجه، عملیات در انتظار `voicecall.continue.start` را پایش می‌کند.      |
| `voicecall.speak`           | `callId`, `message`                                              | بدون انتظار صحبت می‌کند؛ وقتی `realtime.enabled` باشد، از پل بلادرنگ استفاده می‌کند. |
| `voicecall.dtmf`            | `callId`, `digits`                                               |                                                                           |
| `voicecall.end`             | `callId`                                                         |                                                                           |
| `voicecall.status`          | `callId?`                                                        | برای فهرست‌کردن همهٔ تماس‌های فعال، `callId` را حذف کنید.                                   |

`dtmfSequence` فقط با `mode: "conversation"` معتبر است؛ تماس‌های حالت اعلان
در صورت نیاز به ارقام پس از اتصال، باید بعد از ایجاد تماس از
`voicecall.dtmf` استفاده کنند.

## عیب‌یابی

### راه‌اندازی افشای Webhook ناموفق است

راه‌اندازی را از همان محیطی اجرا کنید که Gateway در آن اجرا می‌شود:

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

برای `twilio`، `telnyx` و `plivo`، وضعیت `webhook-exposure` باید سبز باشد. حتی یک
`publicUrl` پیکربندی‌شده نیز وقتی به فضای شبکهٔ محلی یا خصوصی
اشاره کند ناموفق می‌شود، زیرا اپراتور نمی‌تواند با آن نشانی‌ها تماس برگشتی برقرار کند.
از `localhost`، `127.0.0.1`، `0.0.0.0`، `10.x`، `172.16.x`-`172.31.x`،
`192.168.x`، `169.254.x`، `fc00::/7`، `fd00::/8` یا سایر محدوده‌های
NAT در مقیاس اپراتور به‌عنوان `publicUrl` استفاده نکنید.

تماس‌های خروجی حالت اعلان Twilio، TwiML اولیهٔ `<Say>` خود را مستقیماً
در درخواست ایجاد تماس می‌فرستند؛ بنابراین نخستین پیام گفتاری به دریافت
TwiML مربوط به Webhook توسط Twilio وابسته نیست. همچنان برای بازخوانی‌های وضعیت،
تماس‌های مکالمه‌ای، DTMF پیش از اتصال، جریان‌های بلادرنگ و
کنترل تماس پس از اتصال، یک Webhook عمومی لازم است.

از یکی از مسیرهای افشای عمومی استفاده کنید:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          publicUrl: "https://voice.example.com/voice/webhook",
          // یا
          tunnel: { provider: "ngrok" },
          // یا
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

پس از تغییر پیکربندی، Gateway را راه‌اندازی مجدد یا بازبارگذاری کنید، سپس اجرا کنید:

```bash
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` یک اجرای آزمایشی است، مگر اینکه `--yes` را ارسال کنید.

### اعتبارنامه‌های ارائه‌دهنده ناموفق‌اند

ارائه‌دهندهٔ انتخاب‌شده و فیلدهای الزامی اعتبارنامه را بررسی کنید:

- Twilio: `twilio.accountSid`، `twilio.authToken` و `fromNumber`، یا
  `TWILIO_ACCOUNT_SID`، `TWILIO_AUTH_TOKEN` و `TWILIO_FROM_NUMBER`.
- Telnyx: `telnyx.apiKey`، `telnyx.connectionId`، `telnyx.publicKey` و
  `fromNumber`، یا `TELNYX_API_KEY`، `TELNYX_CONNECTION_ID` و
  `TELNYX_PUBLIC_KEY`.
- Plivo: `plivo.authId`، `plivo.authToken` و `fromNumber`، یا
  `PLIVO_AUTH_ID` و `PLIVO_AUTH_TOKEN`.

اعتبارنامه‌ها باید روی میزبان Gateway وجود داشته باشند. ویرایش نمایهٔ پوستهٔ محلی
تا زمانی که Gateway درحال اجرا، محیط خود را راه‌اندازی مجدد یا بازبارگذاری نکند،
بر آن تأثیری ندارد.

### تماس‌ها آغاز می‌شوند اما Webhookهای ارائه‌دهنده نمی‌رسند

تأیید کنید که کنسول ارائه‌دهنده دقیقاً به نشانی URL عمومی Webhook اشاره می‌کند:

```text
https://voice.example.com/voice/webhook
```

سپس وضعیت زمان اجرا را بررسی کنید:

```bash
openclaw voicecall status --call-id <id>
openclaw voicecall tail
openclaw logs --follow
```

دلایل رایج:

- `publicUrl` به مسیری متفاوت از `serve.path` اشاره می‌کند.
- نشانی URL تونل پس از شروع Gateway تغییر کرده است.
- یک پراکسی درخواست را هدایت می‌کند، اما سرآیندهای میزبان/پروتکل را حذف یا بازنویسی می‌کند.
- فایروال یا DNS نام میزبان عمومی را به جایی غیر از Gateway هدایت می‌کند.
- Gateway بدون فعال‌بودن Plugin تماس صوتی راه‌اندازی مجدد شده است.

وقتی یک پراکسی معکوس یا تونل در جلوی Gateway قرار دارد،
`webhookSecurity.allowedHosts` را روی نام میزبان عمومی تنظیم کنید، یا برای یک نشانی پراکسی شناخته‌شده از
`webhookSecurity.trustedProxyIPs` استفاده کنید. فقط زمانی از
`webhookSecurity.trustForwardingHeaders` استفاده کنید که مرز پراکسی
تحت کنترل شما باشد.

### تأیید امضا ناموفق است

امضاهای ارائه‌دهنده در برابر نشانی URL عمومی‌ای بررسی می‌شوند که OpenClaw
از درخواست ورودی بازسازی می‌کند. اگر امضاها ناموفق‌اند:

- تأیید کنید نشانی URL مربوط به Webhook ارائه‌دهنده، شامل طرح، میزبان و مسیر، دقیقاً با `publicUrl` مطابقت دارد.
- برای نشانی‌های URL سطح رایگان ngrok، هنگام تغییر نام میزبان تونل، `publicUrl` را به‌روزرسانی کنید.
- اطمینان یابید پراکسی سرآیندهای اصلی میزبان و پروتکل را حفظ می‌کند، یا `webhookSecurity.allowedHosts` را پیکربندی کنید.
- `skipSignatureVerification` را خارج از آزمایش محلی فعال نکنید.

### پیوستن‌های Google Meet با Twilio ناموفق‌اند

Google Meet برای پیوستن از طریق شماره‌گیری Twilio از این Plugin استفاده می‌کند. ابتدا تماس صوتی را
بررسی کنید:

```bash
openclaw voicecall setup
openclaw voicecall smoke --to "+15555550123"
```

سپس انتقال Google Meet را صراحتاً بررسی کنید:

```bash
openclaw googlemeet setup --transport twilio
```

اگر تماس صوتی سبز است اما شرکت‌کننده هرگز به Meet نمی‌پیوندد، شمارهٔ
شماره‌گیری Meet، PIN و `--dtmf-sequence` را بررسی کنید. ممکن است تماس تلفنی سالم باشد،
درحالی‌که جلسه یک دنبالهٔ DTMF نادرست را رد یا نادیده می‌گیرد.

Google Meet بخش تلفنی Twilio را از طریق `voicecall.start` و با یک
دنبالهٔ DTMF پیش از اتصال آغاز می‌کند. دنباله‌های مشتق‌شده از PIN شامل
`voiceCall.dtmfDelayMs` مربوط به Plugin ‏Google Meet (پیش‌فرض **12000 ms**) به‌عنوان ارقام انتظار ابتدایی Twilio
هستند، زیرا اعلان‌های شماره‌گیری Meet ممکن است دیر برسند. سپس تماس صوتی،
پیش از درخواست خوشامدگویی ابتدایی، به مدیریت بلادرنگ بازهدایت می‌شود.

برای ردگیری زندهٔ مرحله‌ها از `openclaw logs --follow` استفاده کنید. پیوستن سالم Twilio به Meet،
این ترتیب را ثبت می‌کند:

- Google Meet پیوستن Twilio را به تماس صوتی واگذار می‌کند.
- تماس صوتی، TwiML مربوط به DTMF پیش از اتصال را ذخیره می‌کند.
- TwiML اولیهٔ Twilio پیش از مدیریت بلادرنگ مصرف و ارائه می‌شود.
- تماس صوتی، TwiML بلادرنگ را برای تماس Twilio ارائه می‌کند.
- Google Meet پس از تأخیر بعد از DTMF، گفتار مقدماتی را با `voicecall.speak` درخواست می‌کند.

`openclaw voicecall tail` همچنان رکوردهای پایدارشدهٔ تماس را نشان می‌دهد؛ برای
وضعیت تماس و رونوشت‌ها مفید است، اما همهٔ گذارهای Webhook/بلادرنگ
در آن نمایش داده نمی‌شوند.

### تماس بلادرنگ گفتاری ندارد

تأیید کنید فقط یک حالت صوتی فعال است: `realtime.enabled` و
`streaming.enabled` نمی‌توانند هر دو true باشند.

برای تماس‌های بلادرنگ Twilio/Telnyx، این موارد را نیز بررسی کنید:

- یک Plugin ارائه‌دهندهٔ بلادرنگ بارگذاری و ثبت شده است.
- `realtime.provider` تنظیم نشده یا نام یک ارائه‌دهندهٔ ثبت‌شده را مشخص می‌کند.
- کلید API ارائه‌دهنده در دسترس فرایند Gateway است.
- `openclaw logs --follow` ارائه‌شدن TwiML بلادرنگ، شروع پل بلادرنگ و قرارگرفتن خوشامدگویی اولیه در صف را نشان می‌دهد.

## مرتبط

- [حالت مکالمه](/fa/nodes/talk)
- [تبدیل متن به گفتار](/fa/tools/tts)
- [بیدارباش صوتی](/fa/nodes/voicewake)
