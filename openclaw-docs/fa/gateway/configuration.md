---
read_when:
    - راه‌اندازی OpenClaw برای نخستین بار
    - در جست‌وجوی الگوهای رایج پیکربندی
    - پیمایش به بخش‌های مشخص پیکربندی
summary: 'نمای کلی پیکربندی: کارهای رایج، راه‌اندازی سریع و پیوندها به مرجع کامل'
title: پیکربندی
x-i18n:
    generated_at: "2026-07-27T15:14:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09cc04efa16f32e12d6ebcea7a1d36b336df32227fe66953c5d70107708ee6c3
    source_path: gateway/configuration.md
    workflow: 16
---

OpenClaw یک پیکربندی اختیاری <Tooltip tip="JSON5 از توضیحات و ویرگول‌های انتهایی پشتیبانی می‌کند">**JSON5**</Tooltip> را از `~/.openclaw/openclaw.json` می‌خواند. اگر فایل وجود نداشته باشد، OpenClaw از پیش‌فرض‌های امن استفاده می‌کند.

مسیر پیکربندی فعال باید یک فایل معمولی باشد. نوشتن‌های متعلق به OpenClaw آن را به‌صورت اتمی جایگزین می‌کنند (با تغییر نام روی مسیر)، بنابراین هدف یک `openclaw.json` پیوند نمادین، به‌جای نوشتن از طریق آن، جایگزین می‌شود — از چیدمان‌های پیکربندی دارای پیوند نمادین پرهیز کنید. اگر پیکربندی را خارج از دایرکتوری پیش‌فرض وضعیت نگه می‌دارید، `OPENCLAW_CONFIG_PATH` را مستقیماً به فایل واقعی اشاره دهید.

دلایل رایج برای افزودن پیکربندی:

- کانال‌ها را متصل کنید و کنترل کنید چه کسانی می‌توانند به بات پیام دهند
- مدل‌ها، ابزارها، سندباکس‌سازی یا خودکارسازی (cron، هوک‌ها) را تنظیم کنید
- نشست‌ها، رسانه، شبکه یا رابط کاربری را تنظیم دقیق کنید

برای مشاهده همه فیلدهای موجود، [مرجع کامل](/fa/gateway/configuration-reference) را ببینید.

پیکربندی از یک قاعده دو‌بخشی پیروی می‌کند: هم‌سطح‌های ریشه، زیرساخت و پیش‌فرض‌های میان‌عامل را نگه می‌دارند، درحالی‌که `agents.defaults` رفتار حلقه عامل را نگه می‌دارد. ورودی‌های زیر `agents.entries` می‌توانند هرکدام از این دو بخش را، در جاهایی که شِما از بازنویسی به‌ازای هر عامل پشتیبانی می‌کند، بازنویسی کنند.

عامل‌ها و خودکارسازی باید پیش از ویرایش پیکربندی، برای مستندات دقیق در سطح فیلد
از `config.schema.lookup` استفاده کنند. از این صفحه برای راهنمایی وظیفه‌محور و از
[مرجع پیکربندی](/fa/gateway/configuration-reference) برای نقشه گسترده‌تر
فیلدها و پیش‌فرض‌ها استفاده کنید.

<Tip>
**با پیکربندی آشنا نیستید؟** برای راه‌اندازی تعاملی با `openclaw onboard` شروع کنید، یا برای پیکربندی‌های کامل و آماده کپی‌کردن، راهنمای [نمونه‌های پیکربندی](/fa/gateway/configuration-examples) را بررسی کنید.
</Tip>

## پیکربندی حداقلی

```json5
// ~/.openclaw/openclaw.json
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

## ویرایش پیکربندی

<Tabs>
  <Tab title="ویزارد تعاملی">
    ```bash
    openclaw onboard       # فرایند کامل آغاز به کار
    openclaw configure     # ویزارد پیکربندی
    ```
  </Tab>
  <Tab title="CLI (دستورهای تک‌خطی)">
    ```bash
    openclaw config get agents.defaults.workspace
    openclaw config set agents.defaults.heartbeat.every "2h"
    openclaw config unset plugins.entries.brave.config.webSearch.apiKey
    ```
  </Tab>
  <Tab title="رابط کاربری کنترل">
    [http://127.0.0.1:18789](http://127.0.0.1:18789) را باز کنید و از زبانه **Config** استفاده کنید.
    رابط کاربری کنترل، فرمی را از شِمای زنده پیکربندی رندر می‌کند که شامل فراداده مستندات
    فیلد `title` / `description` و نیز شِماهای Plugin و کانال در صورت
    موجود بودن است و یک ویرایشگر **Raw JSON** را به‌عنوان راه گریز ارائه می‌دهد. برای رابط‌های کاربری
    جزئی‌نگر و ابزارهای دیگر، Gateway همچنین `config.schema.lookup` را ارائه می‌کند تا
    یک گره شِمای محدود به مسیر را همراه با خلاصه فرزندان مستقیم واکشی کند.
    تنظیمات ابتدا فیلدهای رایج را نمایش می‌دهند. هر بخش فیلدهای پیشرفته خود را
    در یک گروه جمع‌شده **Advanced (N)** نگه می‌دارد؛ برای بازکردن همه
    گروه‌ها از **Show advanced** استفاده کنید. جست‌وجوی تنظیمات همیشه هر دو سطح را در بر می‌گیرد و در صورت نیاز
    گروه پیشرفته منطبق را باز می‌کند.
  </Tab>
  <Tab title="ویرایش مستقیم">
    `~/.openclaw/openclaw.json` را مستقیماً ویرایش کنید. Gateway فایل را زیر نظر می‌گیرد و تغییرات را به‌طور خودکار اعمال می‌کند ([بارگذاری مجدد فوری](#config-hot-reload) را ببینید).
  </Tab>
</Tabs>

## اعتبارسنجی سخت‌گیرانه

<Warning>
OpenClaw فقط پیکربندی‌هایی را می‌پذیرد که کاملاً با شِما مطابقت داشته باشند. کلیدهای ناشناخته، نوع‌های نادرست یا مقادیر نامعتبر باعث می‌شوند Gateway **از راه‌اندازی خودداری کند**. تنها استثنا در سطح ریشه `$schema` (رشته) است تا ویرایشگرها بتوانند فراداده JSON Schema را پیوست کنند.
</Warning>

`openclaw config schema`، JSON Schema متعارفی را که رابط کاربری کنترل
و اعتبارسنجی استفاده می‌کنند چاپ می‌کند. `config.schema.lookup` یک گره محدود به مسیر را همراه با
خلاصه فرزندان برای ابزارهای جزئی‌نگر واکشی می‌کند. فراداده مستندات فیلد `title`/`description`
در اشیای تودرتو، نویسه عام (`*`)، عضو آرایه (`[]`) و شاخه‌های `anyOf`/
`oneOf`/`allOf` منتقل می‌شود. شِماهای زمان اجرای Plugin و کانال هنگام بارگذاری
رجیستری مانیفست ادغام می‌شوند.

هر برگ پیکربندی در `uiHints` دارای سطح نمایش رایج یا پیشرفته است.
`advanced: false` تنظیمات رایج و `advanced: true` تنظیمات پیشرفته را
مشخص می‌کند. برگی که راهنمای مستقیمی ندارد، سطح نزدیک‌ترین نیای خود را به ارث می‌برد؛
مسیرهایی که هیچ نیای تعریف‌شده‌ای ندارند به‌طور پیش‌فرض پیشرفته‌اند. این فقط بر نمایش
اثر می‌گذارد، نه بر اعتبارسنجی، پیش‌فرض‌ها، رفتار بارگذاری مجدد یا امکان تنظیم کلید.

هنگام شکست اعتبارسنجی:

- Gateway راه‌اندازی نمی‌شود
- فقط دستورهای تشخیصی کار می‌کنند (`openclaw doctor`، `openclaw logs`، `openclaw health`، `openclaw status`)
- برای مشاهده مشکلات دقیق، `openclaw doctor` را اجرا کنید
- برای اعمال تعمیرات، `openclaw doctor --fix` را اجرا کنید (`--repair` همان پرچم است؛ `--yes` اعلان‌ها را رد می‌کند)

Gateway پس از هر راه‌اندازی موفق یک نسخه قابل‌اعتماد از آخرین پیکربندی سالم نگه می‌دارد،
اما راه‌اندازی و بارگذاری مجدد فوری آن را به‌طور خودکار بازیابی نمی‌کنند — فقط `openclaw doctor --fix`
این کار را انجام می‌دهد. اگر اعتبارسنجی `openclaw.json` شکست بخورد (از جمله اعتبارسنجی محلی Plugin)، راه‌اندازی Gateway
شکست می‌خورد یا بارگذاری مجدد نادیده گرفته می‌شود و زمان اجرای فعلی آخرین پیکربندی پذیرفته‌شده را
نگه می‌دارد. یک نوشتن ردشده نیز برای بررسی با نام `<path>.rejected.<timestamp>` ذخیره می‌شود.
Gateway نوشتن‌هایی را که شبیه بازنویسی کامل تصادفی باشند مسدود می‌کند — حذف `gateway.mode`،
از دست رفتن بلوک `meta` یا کوچک‌شدن فایل به بیش از نصف — مگر اینکه نوشتن
صراحتاً تغییرات مخرب را مجاز کند. اگر گزینه پیشنهادی حاوی جای‌نگهدار راز سانسورشده‌ای مانند `***` یا `[redacted]` باشد،
ارتقای آن به آخرین پیکربندی سالم انجام نمی‌شود.

## کارهای رایج

<AccordionGroup>
  <Accordion title="راه‌اندازی یک کانال (WhatsApp، Telegram، Discord و غیره)">
    هر کانال بخش پیکربندی خود را زیر `channels.<provider>` دارد. برای مراحل راه‌اندازی، صفحه اختصاصی کانال را ببینید:

    - [Discord](/fa/channels/discord) — `channels.discord`
    - [Feishu](/fa/channels/feishu) — `channels.feishu`
    - [Google Chat](/fa/channels/googlechat) — `channels.googlechat`
    - [iMessage](/fa/channels/imessage) — `channels.imessage`
    - [Mattermost](/fa/channels/mattermost) — `channels.mattermost`
    - [Microsoft Teams](/fa/channels/msteams) — `channels.msteams`
    - [Signal](/fa/channels/signal) — `channels.signal`
    - [Slack](/fa/channels/slack) — `channels.slack`
    - [Telegram](/fa/channels/telegram) — `channels.telegram`
    - [WhatsApp](/fa/channels/whatsapp) — `channels.whatsapp`

    همه کانال‌ها از الگوی سیاست پیام مستقیم یکسانی استفاده می‌کنند:

    ```json5
    {
      channels: {
        telegram: {
          enabled: true,
          botToken: "123:abc",
          dmPolicy: "pairing",   // جفت‌سازی | فهرست مجاز | باز | غیرفعال
          allowFrom: ["tg:123"], // فقط برای فهرست مجاز/باز
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="انتخاب و پیکربندی مدل‌ها">
    مدل اصلی و جایگزین‌های اختیاری را تنظیم کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "anthropic/claude-sonnet-4-6",
            fallbacks: ["openai/gpt-5.4"],
          },
          models: {
            "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
            "openai/gpt-5.4": { alias: "GPT" },
          },
        },
      },
    }
    ```

    - `agents.defaults.models` نام‌های مستعار و تنظیمات هر مدل را ذخیره می‌کند؛ افزودن یک ورودی هرگز بازنویسی‌های `/model` یا `--model` را محدود نمی‌کند.
    - `agents.defaults.modelPolicy.allow` فهرست مجاز صریح برای بازنویسی‌ها و انتخابگرهای مدل است. این فهرست ارجاع‌های دقیق و نویسه‌های عام `provider/*` را می‌پذیرد؛ برای مجاز کردن هر مدلی، آن را حذف کنید یا از `[]` استفاده کنید.
    - ارجاع‌های مدل از قالب `provider/model` استفاده می‌کنند (برای مثال `anthropic/claude-opus-4-6`).
    - `agents.defaults.imageMaxDimensionPx` کوچک‌سازی تصاویر رونوشت/ابزار را کنترل می‌کند (پیش‌فرض `1200`)؛ مقادیر کمتر معمولاً مصرف توکن بینایی را در اجراهای دارای اسکرین‌شات زیاد کاهش می‌دهند.
    - برای تعویض مدل‌ها در گفت‌وگو، [CLI مدل‌ها](/fa/concepts/models) و برای چرخش احراز هویت و رفتار جایگزینی، [جایگزینی مدل](/fa/concepts/model-failover) را ببینید.
    - برای ارائه‌دهندگان سفارشی/خودمیزبان، بخش [ارائه‌دهندگان سفارشی](/fa/gateway/config-tools#custom-providers-and-base-urls) را در مرجع ببینید.

  </Accordion>

  <Accordion title="کنترل افرادی که می‌توانند به بات پیام دهند">
    دسترسی پیام مستقیم برای هر کانال از طریق `dmPolicy` کنترل می‌شود (پیش‌فرض `"pairing"`):

    - `"pairing"`: فرستندگان ناشناخته یک کد جفت‌سازی یک‌بارمصرف برای تأیید دریافت می‌کنند
    - `"allowlist"`: فقط فرستندگان موجود در `allowFrom` (یا مخزن مجازِ جفت‌شده)
    - `"open"`: همه پیام‌های مستقیم ورودی را مجاز می‌کند (به `allowFrom: ["*"]` نیاز دارد)
    - `"disabled"`: همه پیام‌های مستقیم را نادیده می‌گیرد

    برای گروه‌ها، از `groupPolicy` (`"allowlist" | "open" | "disabled"`) همراه با `groupAllowFrom` یا فهرست‌های مجاز مختص کانال استفاده کنید.

    برای جزئیات هر کانال، [مرجع کامل](/fa/gateway/config-channels#dm-and-group-access) را ببینید.

  </Accordion>

  <Accordion title="راه‌اندازی محدودسازی اشاره در گفت‌وگوی گروهی">
    پیام‌های گروهی به‌طور پیش‌فرض **به اشاره نیاز دارند**. الگوهای فعال‌سازی را برای هر عامل پیکربندی کنید. پاسخ‌های عادی گروه/کانال به‌طور خودکار ارسال می‌شوند؛ برای اتاق‌های مشترکی که عامل باید درباره زمان صحبت‌کردن تصمیم بگیرد، مسیر ابزار پیام را فعال کنید:

    ```json5
    {
      messages: {
        visibleReplies: "automatic", // برای الزام ارسال با ابزار پیام در همه‌جا، روی "message_tool" تنظیم کنید
        groupChat: {
          visibleReplies: "message_tool", // اختیاری؛ خروجی قابل‌مشاهده به message(action=send) نیاز دارد
          unmentionedInbound: "room_event", // گفت‌وگوی گروهی همیشگیِ بدون اشاره، زمینه‌ای بی‌صدا است
        },
      },
      agents: {
        list: [
          {
            id: "main",
            groupChat: {
              mentionPatterns: ["@openclaw", "openclaw"],
            },
          },
        ],
      },
      channels: {
        whatsapp: {
          groups: { "*": { requireMention: true } },
        },
      },
    }
    ```

    - **اشاره‌های فراداده‌ای**: اشاره‌های بومی @ (اشاره با لمس در WhatsApp، ‎@bot در Telegram و غیره)
    - **الگوهای متنی**: الگوهای عبارت منظم امن در `mentionPatterns`
    - **پاسخ‌های قابل‌مشاهده**: `messages.visibleReplies` می‌تواند ارسال با ابزار پیام را به‌صورت سراسری الزامی کند؛ `messages.groupChat.visibleReplies` آن را برای گروه‌ها/کانال‌ها بازنویسی می‌کند.
    - برای حالت‌های پاسخ قابل‌مشاهده، بازنویسی‌های هر کانال و حالت گفت‌وگو با خود، [مرجع کامل](/fa/gateway/config-channels#group-chat-mention-gating) را ببینید.

  </Accordion>

  <Accordion title="محدودکردن Skills برای هر عامل">
    برای خط پایه مشترک از `agents.defaults.skills` استفاده کنید، سپس عامل‌های
    مشخص را با `agents.entries.*.skills` بازنویسی کنید:

    ```json5
    {
      agents: {
        defaults: {
          skills: ["github", "weather"],
        },
        list: [
          { id: "writer" }, // github و weather را به ارث می‌برد
          { id: "docs", skills: ["docs-search"] }, // پیش‌فرض‌ها را جایگزین می‌کند
          { id: "locked-down", skills: [] }, // بدون Skills
        ],
      },
    }
    ```

    - برای نامحدود بودن Skills به‌طور پیش‌فرض، `agents.defaults.skills` را حذف کنید.
    - برای به‌ارث‌بردن پیش‌فرض‌ها، `agents.entries.*.skills` را حذف کنید.
    - برای نداشتن Skills، `agents.entries.*.skills: []` را تنظیم کنید.
    - [Skills](/fa/tools/skills)، [پیکربندی Skills](/fa/tools/skills-config) و
      [مرجع پیکربندی](/fa/gateway/config-agents#agents-defaults-skills) را ببینید.

  </Accordion>

  <Accordion title="پیکربندی پایش سلامت برای هر کانال">
    راه‌اندازی مجدد خودکار سلامت را برای یک کانال یا حساب غیرفعال یا فعال کنید:

    ```json5
    {
      channels: {
        telegram: {
          healthMonitor: { enabled: false },
          accounts: {
            alerts: {
              healthMonitor: { enabled: true },
            },
          },
        },
      },
    }
    ```

    - برای کنترل راه‌اندازی مجدد خودکار یک کانال یا حساب، از `channels.<provider>.healthMonitor.enabled` یا `channels.<provider>.accounts.<id>.healthMonitor.enabled` استفاده کنید.
    - برای اشکال‌زدایی عملیاتی، [بررسی‌های سلامت](/fa/gateway/health) و برای همه فیلدها، [مرجع کامل](/fa/gateway/configuration-reference#gateway) را ببینید.

  </Accordion>

  <Accordion title="پیکربندی نشست‌ها و بازنشانی‌ها">
    نشست‌ها تداوم و جداسازی مکالمه را کنترل می‌کنند:

    ```json5
    {
      session: {
        dmScope: "per-channel-peer",  // برای چندکاربر توصیه می‌شود
        threadBindings: {
          enabled: true,
          idleHours: 24,
          maxAgeHours: 0,
        },
        reset: {
          mode: "daily",
          atHour: 4,
          idleMinutes: 120,
        },
      },
    }
    ```

    - `dmScope`: `main` (مشترک) | `per-peer` | `per-channel-peer` | `per-account-channel-peer`
    - `threadBindings`: پیش‌فرض‌های سراسری برای مسیریابی نشست متصل به رشته. `/focus`، `/unfocus`، `/agents`، `/session idle` و `/session max-age` این اتصال را برای هر نشست برقرار، قطع، فهرست و تنظیم می‌کنند (Discord رشته‌ها را متصل می‌کند و Telegram موضوع‌ها/مکالمه‌ها را).
    - برای محدوده‌بندی، پیوندهای هویتی و سیاست ارسال، به [مدیریت نشست](/fa/concepts/session) مراجعه کنید.
    - برای همه فیلدها، [مرجع کامل](/fa/gateway/config-agents#session) را ببینید.

  </Accordion>

  <Accordion title="فعال‌سازی سندباکس">
    نشست‌های عامل را در محیط‌های اجرای سندباکس ایزوله اجرا کنید:

    ```json5
    {
      agents: {
        defaults: {
          sandbox: {
            mode: "non-main",  // off | non-main | all
            scope: "agent",    // session | agent | shared
          },
        },
      },
    }
    ```

    ابتدا ایمیج را بسازید — در یک نسخه دریافت‌شده از کد منبع، `scripts/sandbox-setup.sh` را اجرا کنید؛ یا برای نصب از npm، فرمان درون‌خطی `docker build` را در [سندباکس § ایمیج‌ها و راه‌اندازی](/fa/gateway/sandboxing#images-and-setup) ببینید.

    برای راهنمای کامل به [سندباکس](/fa/gateway/sandboxing) و برای همه گزینه‌ها به [مرجع کامل](/fa/gateway/config-agents#agentsdefaultssandbox) مراجعه کنید.

  </Accordion>

  <Accordion title="فعال‌سازی پوش مبتنی بر رله برای بیلدهای رسمی iOS">
    پوش مبتنی بر رله برای بیلدهای عمومی App Store از رله میزبانی‌شده OpenClaw استفاده می‌کند: `https://ios-push-relay.openclaw.ai`.

    استقرارهای سفارشی رله به مسیر بیلد/استقرار iOS عمداً مجزایی نیاز دارند که URL رله آن با URL رله Gateway مطابقت داشته باشد. اگر از بیلد رله سفارشی استفاده می‌کنید، این مورد را در پیکربندی Gateway تنظیم کنید:

    ```json5
    {
      gateway: {
        push: {
          apns: {
            relay: {
              baseUrl: "https://relay.example.com",
              // اختیاری. پیش‌فرض: 10000
              timeoutMs: 10000,
            },
          },
        },
      },
    }
    ```

    معادل CLI:

    ```bash
    openclaw config set gateway.push.apns.relay.baseUrl https://relay.example.com
    ```

    کارکرد این تنظیم:

    - به Gateway امکان می‌دهد `push.test`، تلنگرهای بیدارسازی و بیدارسازی‌های اتصال مجدد را از طریق رله خارجی ارسال کند.
    - از مجوز ارسال مختص ثبت‌نام استفاده می‌کند که برنامه جفت‌شده iOS آن را منتقل می‌کند. Gateway به توکن رله سراسریِ استقرار نیاز ندارد.
    - هر ثبت‌نام مبتنی بر رله را به هویت Gateway جفت‌شده با برنامه iOS متصل می‌کند تا Gateway دیگری نتواند از ثبت‌نام ذخیره‌شده دوباره استفاده کند.
    - بیلدهای محلی/دستی iOS را روی APNs مستقیم نگه می‌دارد. ارسال‌های مبتنی بر رله فقط برای بیلدهای رسمی توزیع‌شده‌ای اعمال می‌شوند که از طریق رله ثبت‌نام کرده‌اند.
    - باید با URL پایه رله تعبیه‌شده در بیلد iOS مطابقت داشته باشد تا ترافیک ثبت‌نام و ارسال به همان استقرار رله برسد.

    جریان سرتاسری:

    1. برنامه رسمی iOS را نصب کنید.
    2. اختیاری: فقط هنگام استفاده از یک بیلد رله سفارشی و عمداً مجزا، `gateway.push.apns.relay.baseUrl` را روی Gateway پیکربندی کنید.
    3. برنامه iOS را با Gateway جفت کنید و اجازه دهید نشست‌های Node و اپراتور هر دو متصل شوند.
    4. برنامه iOS هویت Gateway را دریافت می‌کند، با استفاده از App Attest به‌همراه رسید برنامه در رله ثبت‌نام می‌کند و سپس محموله `push.apns.register` مبتنی بر رله را در Gateway جفت‌شده منتشر می‌کند.
    5. Gateway هندل رله و مجوز ارسال را ذخیره می‌کند، سپس از آن‌ها برای `push.test`، تلنگرهای بیدارسازی و بیدارسازی‌های اتصال مجدد استفاده می‌کند.

    نکات عملیاتی:

    - اگر برنامه iOS را به Gateway دیگری منتقل کردید، برنامه را دوباره متصل کنید تا بتواند ثبت‌نام رله جدیدی را که به آن Gateway متصل است منتشر کند.
    - اگر بیلد جدیدی از iOS منتشر کنید که به استقرار رله دیگری اشاره می‌کند، برنامه به‌جای استفاده مجدد از مبدأ رله قدیمی، ثبت‌نام رله ذخیره‌شده خود را تازه‌سازی می‌کند.

    نکته سازگاری:

    - `OPENCLAW_APNS_RELAY_BASE_URL` و `OPENCLAW_APNS_RELAY_TIMEOUT_MS` همچنان به‌عنوان بازنویسی‌های موقت محیطی کار می‌کنند.
    - URLهای رله سفارشی Gateway باید با URL پایه رله تعبیه‌شده در بیلد iOS مطابقت داشته باشند؛ مسیر انتشار عمومی App Store بازنویسی‌های URL رله سفارشی iOS را رد می‌کند.
    - `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true` همچنان یک راه گریز توسعه‌ای محدود به loopback است؛ URLهای رله HTTP را در پیکربندی ماندگار نکنید.

    برای جریان سرتاسری، [برنامه iOS](/fa/platforms/ios#relay-backed-push-for-official-builds) و برای مدل امنیتی رله، [جریان احراز هویت و اعتماد](/fa/platforms/ios#authentication-and-trust-flow) را ببینید.

  </Accordion>

  <Accordion title="راه‌اندازی Heartbeat (بررسی‌های دوره‌ای)">
    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "30m",
            target: "last",
          },
        },
      },
    }
    ```

    - `every`: رشته مدت‌زمان (`30m`، `2h`). برای غیرفعال‌سازی، `0m` را تنظیم کنید. پیش‌فرض: `30m`.
    - `target`: `last` | `none` | `<channel-id>` (برای نمونه `discord`، `matrix`، `telegram` یا `whatsapp`)
    - `directPolicy`: `allow` (پیش‌فرض) یا `block` برای مقصدهای Heartbeat به سبک پیام مستقیم
    - برای راهنمای کامل به [Heartbeat](/fa/gateway/heartbeat) مراجعه کنید.

  </Accordion>

  <Accordion title="پیکربندی کارهای Cron">
    ```json5
    {
      cron: {
        enabled: true,
        sessionRetention: "24h",
      },
    }
    ```

    - `sessionRetention`: نشست‌های تکمیل‌شده اجرای ایزوله را از ردیف‌های نشست SQLite پاک‌سازی می‌کند (پیش‌فرض `24h`؛ برای غیرفعال‌سازی، `false` را تنظیم کنید).
    - تاریخچه اجرا به‌طور خودکار جدیدترین 2000 ردیف ترمینال را برای هر کار نگه می‌دارد؛ ردیف‌های گم‌شده بازه پاک‌سازی 24 ساعته خود را حفظ می‌کنند.
    - برای نمای کلی قابلیت و نمونه‌های CLI به [کارهای Cron](/fa/automation/cron-jobs) مراجعه کنید.

  </Accordion>

  <Accordion title="راه‌اندازی Webhookها (هوک‌ها)">
    نقاط پایانی Webhook مبتنی بر HTTP را روی Gateway فعال کنید:

    ```json5
    {
      hooks: {
        enabled: true,
        token: "shared-secret",
        path: "/hooks",
        defaultSessionKey: "hook:ingress",
        allowRequestSessionKey: false,
        allowedSessionKeyPrefixes: ["hook:"],
        mappings: [
          {
            match: { path: "gmail" },
            action: "agent",
            agentId: "main",
            deliver: true,
          },
        ],
      },
    }
    ```

    نکته امنیتی:
    - تمام محتوای محموله هوک/Webhook را ورودی غیرقابل‌اعتماد در نظر بگیرید.
    - از یک `hooks.token` اختصاصی استفاده کنید؛ اسرار فعال احراز هویت Gateway ‏(`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` یا `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) را دوباره استفاده نکنید.
    - احراز هویت هوک فقط از طریق هدر انجام می‌شود (`Authorization: Bearer ...` یا `x-openclaw-token`)؛ توکن‌های رشته پرس‌وجو رد می‌شوند.
    - `hooks.path` نمی‌تواند `/` باشد؛ ورودی Webhook را روی یک زیرمسیر اختصاصی مانند `/hooks` نگه دارید.
    - پرچم‌های دور زدن محتوای ناامن (`hooks.gmail.allowUnsafeExternalContent`، `hooks.mappings[].allowUnsafeExternalContent`) را غیرفعال نگه دارید، مگر هنگام اشکال‌زدایی با محدوده کاملاً محدود.
    - اگر `hooks.allowRequestSessionKey` را فعال می‌کنید، `hooks.allowedSessionKeyPrefixes` را نیز تنظیم کنید تا کلیدهای نشست انتخاب‌شده توسط فراخواننده محدود شوند.
    - برای عامل‌های هدایت‌شده با هوک، رده‌های قدرتمند مدل‌های مدرن و سیاست سخت‌گیرانه ابزار را ترجیح دهید (برای نمونه، فقط پیام‌رسانی به‌همراه سندباکس در صورت امکان).

    برای همه گزینه‌های نگاشت و یکپارچه‌سازی Gmail به [مرجع کامل](/fa/gateway/configuration-reference#hooks) مراجعه کنید.

  </Accordion>

  <Accordion title="پیکربندی مسیریابی چندعاملی">
    چند عامل ایزوله را با فضاهای کاری و نشست‌های جداگانه اجرا کنید:

    ```json5
    {
      agents: {
        list: [
          { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
          { id: "work", workspace: "~/.openclaw/workspace-work" },
        ],
      },
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
      ],
    }
    ```

    برای قواعد اتصال و پروفایل‌های دسترسی هر عامل، [چندعاملی](/fa/concepts/multi-agent) و [مرجع کامل](/fa/gateway/config-agents#multi-agent-routing) را ببینید.

  </Accordion>

  <Accordion title="تقسیم پیکربندی میان چند فایل ($include)">
    برای سازمان‌دهی پیکربندی‌های بزرگ از `$include` استفاده کنید:

    ```json5
    // ~/.openclaw/openclaw.json
    {
      gateway: { port: 18789 },
      agents: { $include: "./agents.json5" },
      broadcast: {
        $include: ["./clients/a.json5", "./clients/b.json5"],
      },
    }
    ```

    - **یک فایل**: شیء دربرگیرنده را جایگزین می‌کند
    - **آرایه‌ای از فایل‌ها**: به‌ترتیب به‌صورت عمیق ادغام می‌شوند (مورد بعدی اولویت دارد)، تا عمق 10 سطح تودرتو
    - **کلیدهای هم‌سطح**: پس از includeها ادغام می‌شوند (مقادیر includeشده را بازنویسی می‌کنند)
    - **مسیرهای نسبی**: نسبت به فایل includeکننده تفکیک می‌شوند
    - **قالب مسیر**: مسیرهای include نباید شامل بایت null باشند و باید پیش و پس از تفکیک، اکیداً کوتاه‌تر از 4096 نویسه باشند
    - **نوشتن‌های متعلق به OpenClaw**: وقتی یک نوشتن فقط یک بخش سطح‌بالا را تغییر می‌دهد
      که پشتوانه آن یک include تک‌فایلی مانند `plugins: { $include: "./plugins.json5" }` است،
      OpenClaw همان فایل includeشده را به‌روزرسانی می‌کند و `openclaw.json` را دست‌نخورده باقی می‌گذارد
    - **نوشتن عبوری پشتیبانی‌نشده**: includeهای ریشه، آرایه‌های include و includeهایی
      که بازنویسی هم‌سطح دارند، برای نوشتن‌های متعلق به OpenClaw به‌صورت بسته و ایمن شکست می‌خورند،
      به‌جای اینکه پیکربندی را تخت کنند
    - **محصورسازی**: مسیرهای `$include` باید در دایرکتوری نگهدارنده
      `openclaw.json` تفکیک شوند. برای اشتراک‌گذاری یک درخت میان چند ماشین یا کاربر،
      `OPENCLAW_INCLUDE_ROOTS` را روی فهرستی از مسیرها (`:` در POSIX، ‏`;` در Windows) شامل
      دایرکتوری‌های اضافی قابل ارجاع توسط includeها تنظیم کنید. پیوندهای نمادین تفکیک
      و دوباره بررسی می‌شوند؛ بنابراین مسیری که از نظر واژگانی در دایرکتوری پیکربندی قرار دارد، اما
      مقصد واقعی‌اش از همه ریشه‌های مجاز خارج می‌شود، همچنان رد خواهد شد.
    - **مدیریت خطا**: خطاهای روشن برای فایل‌های مفقود، خطاهای تجزیه، includeهای حلقوی، قالب نامعتبر مسیر و طول بیش‌ازحد

  </Accordion>
</AccordionGroup>

## بارگذاری مجدد فوری پیکربندی

Gateway بر `~/.openclaw/openclaw.json` نظارت می‌کند و تغییرات را به‌طور خودکار اعمال می‌کند — برای بیشتر تنظیمات نیازی به راه‌اندازی مجدد دستی نیست.

ویرایش مستقیم فایل تا زمان اعتبارسنجی، غیرقابل‌اعتماد در نظر گرفته می‌شود. ناظر منتظر می‌ماند
تا آشفتگی ناشی از نوشتن فایل موقت/تغییرنام توسط ویرایشگر فروکش کند، فایل نهایی را می‌خواند و
ویرایش‌های خارجی نامعتبر را بدون بازنویسی `openclaw.json` رد می‌کند. نوشتن‌های پیکربندی
متعلق به OpenClaw پیش از نوشتن از همان دروازه طرح‌واره عبور می‌کنند (برای قواعد بازنویسی/بازگردانی
اعمال‌شده بر همه نوشتن‌ها، [اعتبارسنجی سخت‌گیرانه](#strict-validation) را ببینید).

اگر `config reload skipped (invalid config)` را مشاهده کردید یا هنگام راه‌اندازی `Invalid
config` گزارش شد، پیکربندی را بررسی کنید، `openclaw config validate` و سپس برای تعمیر `openclaw
doctor --fix` را اجرا کنید. برای چک‌لیست به [عیب‌یابی Gateway](/fa/gateway/troubleshooting#gateway-rejected-invalid-config)
مراجعه کنید.

### حالت‌های بارگذاری مجدد

| حالت                   | رفتار                                                                                |
| ---------------------- | --------------------------------------------------------------------------------------- |
| **`hybrid`** (پیش‌فرض) | تغییرات امن را فوراً و بدون راه‌اندازی مجدد اعمال می‌کند. برای تغییرات حیاتی به‌طور خودکار راه‌اندازی مجدد می‌شود.           |
| **`hot`**              | فقط تغییرات امن را بدون راه‌اندازی مجدد اعمال می‌کند. وقتی راه‌اندازی مجدد لازم باشد، هشداری ثبت می‌کند — مدیریت آن بر عهده شماست. |
| **`restart`**          | با هر تغییر پیکربندی، چه امن باشد چه نباشد، Gateway را راه‌اندازی مجدد می‌کند.                                 |
| **`off`**              | پایش فایل را غیرفعال می‌کند. تغییرات در راه‌اندازی مجدد دستی بعدی اعمال می‌شوند.                 |

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
  },
}
```

### چه چیزهایی بدون راه‌اندازی مجدد اعمال می‌شوند و چه چیزهایی به راه‌اندازی مجدد نیاز دارند

بیشتر فیلدها بدون وقفه و بدون راه‌اندازی مجدد اعمال می‌شوند؛ برخی بخش‌هایی که بدون راه‌اندازی مجدد اعمال می‌شوند، به‌جای کل Gateway فقط همان
زیرسامانه (کانال، cron، heartbeat، پایشگر سلامت) را راه‌اندازی مجدد می‌کنند. در حالت
`hybrid`، تغییراتی که به راه‌اندازی مجدد Gateway نیاز دارند، به‌طور خودکار مدیریت می‌شوند.

| دسته            | فیلدها                                                                  | نیازمند راه‌اندازی مجدد Gateway؟      |
| ------------------- | ----------------------------------------------------------------------- | ---------------------------- |
| کانال‌ها            | `channels.*`، `web` (WhatsApp) — همه کانال‌های داخلی و کانال‌های Plugin       | خیر (همان کانال را راه‌اندازی مجدد می‌کند)   |
| عامل و مدل‌ها      | `agent`، `agents`، `models`، `routing`                                  | خیر                           |
| خودکارسازی          | `hooks`، `cron`، `agent.heartbeat`                                      | خیر (همان زیرسامانه را راه‌اندازی مجدد می‌کند) |
| نشست‌ها و پیام‌ها | `session`، `messages`                                                   | خیر                           |
| ابزارها و رسانه       | `tools`، `skills`، `mcp`، `audio`، `talk`                               | خیر                           |
| پیکربندی Plugin       | `plugins.entries.*`، `plugins.allow`، `plugins.deny`، `plugins.enabled` | خیر (زمان‌اجرای Plugin را بارگذاری مجدد می‌کند)  |
| رابط کاربری و موارد متفرقه           | `ui`، `logging`، `identity`، `bindings`                                 | خیر                           |
| سرور Gateway      | `gateway.*` (درگاه، اتصال، احراز هویت، tailscale، TLS، HTTP، ارسال)              | **بله**                      |
| زیرساخت      | `discovery`، `browser`، `plugins.load`، `plugins.installs`              | **بله**                      |

<Note>
`gateway.reload` و `gateway.remote` در `gateway.*` استثنا هستند — تغییر آن‌ها باعث راه‌اندازی مجدد **نمی‌شود**. هر Plugin نیز می‌تواند این جدول را بازنویسی کند: یک Plugin بارگذاری‌شده ممکن است پیشوندهای پیکربندی مختص خود را که موجب راه‌اندازی مجدد می‌شوند اعلام کند (برای نمونه، Plugin همراه Canvas برای `plugins.enabled`، `plugins.allow` و `plugins.deny`، و نه فقط `plugins.entries.canvas` متعلق به خودش، Gateway را راه‌اندازی مجدد می‌کند)؛ بنابراین رفتار واقعی به Pluginهای فعال بستگی دارد.
</Note>

### برنامه‌ریزی بارگذاری مجدد

وقتی فایل منبعی را که از طریق `$include` ارجاع شده است ویرایش می‌کنید، OpenClaw
بارگذاری مجدد را براساس چیدمان نوشته‌شده در منبع برنامه‌ریزی می‌کند، نه نمای مسطح‌شده در حافظه.
این کار تصمیم‌های بارگذاری مجدد بدون توقف (اعمال مستقیم در برابر راه‌اندازی مجدد) را حتی زمانی قابل‌پیش‌بینی نگه می‌دارد که
یک بخش سطح‌بالا در فایل شامل‌شده جداگانه‌ای مانند
`plugins: { $include: "./plugins.json5" }` قرار داشته باشد. اگر
چیدمان منبع مبهم باشد، برنامه‌ریزی بارگذاری مجدد به‌صورت ایمن متوقف می‌شود.

## RPC پیکربندی (به‌روزرسانی‌های برنامه‌نویسی‌شده)

برای ابزارهایی که پیکربندی را از طریق API مربوط به Gateway می‌نویسند، این روند ترجیح داده می‌شود:

- `config.schema.lookup` برای بررسی یک زیردرخت (گره کم‌عمق طرح‌واره + خلاصه‌های
  فرزندان)
- `config.get` برای دریافت تصویر لحظه‌ای فعلی به‌همراه `hash`
- `config.patch` برای به‌روزرسانی‌های جزئی (وصله ادغام JSON: اشیا ادغام می‌شوند، `null`
  حذف می‌کند و آرایه‌ها، اگر حذف ورودی‌ها با `replacePaths` صریحاً تأیید شده باشد،
  جایگزین می‌شوند)
- `config.apply` فقط زمانی که قصد دارید کل پیکربندی را جایگزین کنید
- `update.run` برای خودبه‌روزرسانی صریح به‌همراه راه‌اندازی مجدد؛ اگر نشست پس از راه‌اندازی مجدد باید یک نوبت پیگیری را اجرا کند، `continuationMessage` را وارد کنید
- `update.status` برای بررسی جدیدترین نشانگر راه‌اندازی مجددِ به‌روزرسانی و تأیید نسخه در حال اجرا پس از راه‌اندازی مجدد

عامل‌ها باید `config.schema.lookup` را نخستین مرجع برای مستندات و محدودیت‌های دقیق
در سطح فیلد در نظر بگیرند. وقتی به نقشه گسترده‌تر پیکربندی، مقادیر پیش‌فرض یا پیوندهای ارجاعات اختصاصی
زیرسامانه‌ها نیاز دارند، از [مرجع پیکربندی](/fa/gateway/configuration-reference)
استفاده کنند.

<Note>
نوشتن‌های صفحه کنترل (`config.apply`، `config.patch`، `update.run`)
برای هر متد و هر `deviceId+clientIp` به 30 درخواست در هر 60 ثانیه
محدود شده‌اند؛ [محدودسازی نرخ](/gateway/security/rate-limiting) را ببینید. درخواست‌های راه‌اندازی مجدد
با هم ادغام می‌شوند و سپس بین چرخه‌های راه‌اندازی مجدد، یک دوره انتظار 30 ثانیه‌ای اعمال می‌شود.
`update.status` فقط‌خواندنی است، اما دامنه دسترسی مدیر را لازم دارد، زیرا نشانگر راه‌اندازی مجدد می‌تواند
شامل خلاصه مراحل به‌روزرسانی و بخش‌های انتهایی خروجی فرمان باشد.
</Note>

نمونه وصله جزئی:

```bash
openclaw gateway call config.get --params '{}'  # ثبت payload.hash
openclaw gateway call config.patch --params '{
  "raw": "{ channels: { telegram: { groups: { \"*\": { requireMention: false } } } } }",
  "baseHash": "<hash>"
}'
```

هر دو `config.apply` و `config.patch`، مقادیر `raw`، `baseHash`، `sessionKey`،
`note` و `restartDelayMs` را می‌پذیرند. پس از آنکه فایل پیکربندی از قبل وجود داشته باشد،
`baseHash` برای هر دو متد الزامی است (در نخستین نوشتن، اگر پیکربندی موجود نباشد، این بررسی انجام نمی‌شود).

`config.patch` همچنین `replacePaths` را می‌پذیرد؛ آرایه‌ای از مسیرهای پیکربندی که
جایگزینی آرایه در آن‌ها عمدی است. اگر وصله‌ای یک آرایه موجود را با ورودی‌های کمتر جایگزین
یا حذف کند، Gateway نوشتن را رد می‌کند، مگر آنکه همان مسیر دقیق در
`replacePaths` وجود داشته باشد؛ آرایه‌های تودرتوی داخل ورودی‌های آرایه از `[]` استفاده می‌کنند، مانند
`agents.entries.*.skills`. این کار مانع می‌شود تصاویر لحظه‌ای ناقص `config.get`
آرایه‌های مسیریابی یا فهرست مجاز را بی‌سروصدا بازنویسی کنند. وقتی قصد دارید
کل پیکربندی را جایگزین کنید، از `config.apply` استفاده کنید.

## متغیرهای محیطی

OpenClaw متغیرهای محیطی را از فرایند والد و همچنین موارد زیر می‌خواند:

- `.env` از دایرکتوری کاری فعلی (در صورت وجود)
- `~/.openclaw/.env` (جایگزین سراسری)

هیچ‌یک از این فایل‌ها متغیرهای محیطی موجود را بازنویسی نمی‌کنند. همچنین می‌توانید متغیرهای محیطی درون‌خطی را در پیکربندی تنظیم کنید:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
  },
}
```

<Accordion title="واردکردن محیط پوسته (اختیاری)">
  اگر فعال باشد و کلیدهای مورد انتظار تنظیم نشده باشند، OpenClaw پوسته ورود شما را اجرا و فقط کلیدهای موجودنبود‌ه را وارد می‌کند:

```json5
{
  env: {
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
}
```

متغیر محیطی معادل: `OPENCLAW_LOAD_SHELL_ENV=1`. مقدار پیش‌فرض `timeoutMs`: `15000`.
</Accordion>

<Accordion title="جایگزینی متغیر محیطی در مقادیر پیکربندی">
  با `${VAR_NAME}` در هر مقدار رشته‌ای پیکربندی به متغیرهای محیطی ارجاع دهید:

```json5
{
  gateway: { auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" } },
  models: { providers: { custom: { apiKey: "${CUSTOM_API_KEY}" } } },
}
```

قواعد:

- فقط نام‌های بزرگ‌نویسی‌شده تطبیق داده می‌شوند: `[A-Z_][A-Z0-9_]*`
- متغیرهای موجودنبوده یا خالی هنگام بارگذاری خطا ایجاد می‌کنند
- برای خروجی تحت‌اللفظی با `$${VAR}` از نویسه گریز استفاده کنید
- درون فایل‌های `$include` کار می‌کند
- جایگزینی درون‌خطی: `"${BASE}/v1"` → `"https://api.example.com/v1"`

</Accordion>

<Accordion title="ارجاعات محرمانه (محیط، فایل، اجرا)">
  برای فیلدهایی که از اشیای SecretRef پشتیبانی می‌کنند، می‌توانید از موارد زیر استفاده کنید:

```json5
{
  models: {
    providers: {
      openai: { apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" } },
    },
  },
  skills: {
    entries: {
      "image-lab": {
        apiKey: {
          source: "file",
          provider: "filemain",
          id: "/skills/entries/image-lab/apiKey",
        },
      },
    },
  },
  channels: {
    googlechat: {
      serviceAccount: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

جزئیات SecretRef (از جمله `secrets.providers` برای `env`/`file`/`exec`) در [مدیریت اسرار](/fa/gateway/secrets) آمده است.
مسیرهای اعتبارنامه پشتیبانی‌شده در [سطح اعتبارنامه SecretRef](/fa/reference/secretref-credential-surface) فهرست شده‌اند.
</Accordion>

برای تقدم کامل و منابع، [محیط](/fa/help/environment) را ببینید.

## مرجع کامل

برای مرجع کامل فیلدبه‌فیلد، **[مرجع پیکربندی](/fa/gateway/configuration-reference)** را ببینید.

---

_مرتبط: [نمونه‌های پیکربندی](/fa/gateway/configuration-examples) · [مرجع پیکربندی](/fa/gateway/configuration-reference) · [Doctor](/fa/gateway/doctor)_

## مرتبط

- [مرجع پیکربندی](/fa/gateway/configuration-reference)
- [نمونه‌های پیکربندی](/fa/gateway/configuration-examples)
- [راهنمای عملیاتی Gateway](/fa/gateway)
