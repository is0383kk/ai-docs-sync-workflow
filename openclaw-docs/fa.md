---
read_when:
    - معرفی OpenClaw به تازه‌واردان
summary: OpenClaw یک Gateway چندکاناله برای عامل‌های هوش مصنوعی است که روی هر سیستم‌عاملی اجرا می‌شود.
title: OpenClaw
x-i18n:
    generated_at: "2026-07-27T16:40:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ce948d12d4b4fcbde2597f9b33f50b99c4f677b69e0f5d72677b2f6683291f3
    source_path: index.md
    workflow: 16
---

# OpenClaw 🦞

<p align="center">
    <img
        src="/assets/openclaw-hero-light.png"
        alt="OpenClaw"
        width="500"
        class="dark:hidden"
    />
    <img
        src="/assets/openclaw-hero-dark.png"
        alt="OpenClaw"
        width="500"
        class="hidden dark:block"
    />
</p>

> _«پوست‌اندازی! پوست‌اندازی!»_ — احتمالاً یک خرچنگ فضایی

<p align="center">
  <strong>Gateway مستقل از سیستم‌عامل برای عامل‌های هوش مصنوعی در Discord، Google Chat، iMessage، Matrix، Microsoft Teams، Signal، Slack، Telegram، WhatsApp، Zalo و موارد دیگر.</strong><br />
  پیام بفرستید و پاسخ عامل را در جیب خود دریافت کنید. یک Gateway را در میان Pluginهای کانال، WebChat و Nodeهای همراه اجرا کنید.<br />
  توسعه‌یافته به‌صورت آزاد توسط <a href="https://openclaw.org">بنیاد OpenClaw</a>، یک سازمان غیرانتفاعی.
</p>

<Columns>
  <Card title="شروع به کار" href="/fa/start/getting-started" icon="rocket">
    OpenClaw را نصب کنید و Gateway را ظرف چند دقیقه راه‌اندازی کنید.
  </Card>
  <Card title="اجرای راه‌اندازی اولیه" href="/fa/start/wizard" icon="list-checks">
    راه‌اندازی هدایت‌شده با `openclaw onboard` و جریان‌های جفت‌سازی.
  </Card>
  <Card title="اتصال یک کانال" href="/fa/channels" icon="message-circle">
    Discord، Signal، Telegram، WhatsApp و موارد دیگر را متصل کنید تا از هرجا گفت‌وگو کنید.
  </Card>
  <Card title="باز کردن رابط کنترل" href="/fa/web/control-ui" icon="layout-dashboard">
    داشبورد مرورگر را برای گفت‌وگو، پیکربندی و نشست‌ها اجرا کنید.
  </Card>
</Columns>

## مرور مستندات

مرورگرهای همراه ممکن است منوی بخش را بدون نوار کامل زبانه‌های دسکتاپ نمایش دهند. از
این پیوندهای مرکزی برای دسترسی به همان بخش‌های سطح‌بالای مستندات از داخل صفحه استفاده کنید.

<Columns>
  <Card title="شروع به کار" href="/fa" icon="rocket">
    نمای کلی، ویترین، گام‌های نخست و راهنماهای راه‌اندازی.
  </Card>
  <Card title="نصب" href="/fa/install" icon="download">
    روش‌های نصب، به‌روزرسانی‌ها، کانتینرها، میزبانی و راه‌اندازی پیشرفته.
  </Card>
  <Card title="کانال‌ها" href="/fa/channels" icon="messages-square">
    کانال‌های پیام‌رسانی، جفت‌سازی، مسیریابی، گروه‌های دسترسی و تضمین کیفیت کانال.
  </Card>
  <Card title="عامل‌ها" href="/fa/concepts/architecture" icon="bot">
    معماری، نشست‌ها، زمینه، حافظه و مسیریابی چندعاملی.
  </Card>
  <Card title="قابلیت‌ها" href="/fa/tools" icon="wand-sparkles">
    ابزارها، Skills، cron، webhookها و قابلیت‌های خودکارسازی.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="store">
    بازار Plugin، انتشار، گزینش و راهنمای اعتماد.
  </Card>
  <Card title="مدل‌ها" href="/fa/providers" icon="brain">
    ارائه‌دهندگان، پیکربندی مدل، جایگزینی هنگام خرابی و سرویس‌های مدل محلی.
  </Card>
  <Card title="پلتفرم‌ها" href="/fa/platforms" icon="monitor-smartphone">
    macOS، Windows، iOS، Android، Nodeها و رابط‌های وب.
  </Card>
  <Card title="Gateway و عملیات" href="/fa/gateway" icon="server">
    پیکربندی، امنیت، عیب‌یابی و عملیات Gateway.
  </Card>
  <Card title="مرجع" href="/fa/cli" icon="terminal">
    مرجع CLI، طرح‌واره‌ها، RPC، یادداشت‌های انتشار و الگوها.
  </Card>
  <Card title="راهنما" href="/fa/help" icon="life-buoy">
    رفع اشکال، پرسش‌های متداول، آزمایش، عیب‌یابی و بررسی‌های محیطی.
  </Card>
</Columns>

## OpenClaw چیست؟

OpenClaw یک **Gateway خودمیزبان** است که برنامه‌های گفت‌وگوی محبوب شما — Discord، Google Chat، iMessage، Matrix، Microsoft Teams، Signal، Slack، Telegram، WhatsApp، Zalo و موارد دیگر از طریق Pluginهای کانال — را به عامل‌های کدنویسی هوش مصنوعی متصل می‌کند. یک فرایند Gateway را روی دستگاه خودتان (یا یک سرور) اجرا می‌کنید و آن به پلی میان برنامه‌های پیام‌رسانی شما و یک دستیار هوش مصنوعی همیشه‌دردسترس تبدیل می‌شود.

**برای چه کسانی است؟** توسعه‌دهندگان و کاربران حرفه‌ای که دستیار هوش مصنوعی شخصی می‌خواهند تا از هرجا به آن پیام بدهند، بدون آنکه کنترل داده‌هایشان را واگذار کنند یا به یک سرویس میزبانی‌شده متکی باشند.

**چه چیزی آن را متمایز می‌کند؟**

- **خودمیزبان**: روی سخت‌افزار شما و طبق قواعد شما اجرا می‌شود
- **چندکاناله**: یک Gateway به‌طور هم‌زمان به تمام Pluginهای کانال پیکربندی‌شده سرویس می‌دهد
- **عامل‌محور**: برای عامل‌های کدنویسی با امکان استفاده از ابزار، نشست‌ها، حافظه و مسیریابی چندعاملی ساخته شده است
- **متن‌باز**: دارای مجوز MIT و جامعه‌محور

**به چه چیزهایی نیاز دارید؟** Node 24.15+ (توصیه‌شده)، Node 22 LTS (`22.22.3+`) برای سازگاری، یا Node 25.9+، یک کلید API از ارائه‌دهنده منتخب و 5 دقیقه زمان. برای بهترین کیفیت و امنیت، از قدرتمندترین مدل نسل جدید موجود استفاده کنید.

## نحوه کار

```mermaid
flowchart LR
  A["برنامه‌های گفت‌وگو + Pluginها"] --> B["Gateway"]
  B --> C["عامل OpenClaw"]
  B --> D["CLI"]
  B --> E["رابط کنترل وب"]
  B --> F["برنامه macOS"]
  B --> G["Nodeهای iOS و Android"]
```

Gateway منبع واحد حقیقت برای نشست‌ها، مسیریابی و اتصال‌های کانال است.

## قابلیت‌های کلیدی

<Columns>
  <Card title="Gateway چندکاناله" icon="network" href="/fa/channels">
    Discord، iMessage، Signal، Slack، Telegram، WhatsApp، WebChat و موارد دیگر با یک فرایند Gateway.
  </Card>
  <Card title="کانال‌های Plugin" icon="plug" href="/fa/tools/plugin">
    Pluginهای کانال، Matrix، Nostr، Twitch، Zalo و موارد دیگر را اضافه می‌کنند؛ Pluginهای رسمی در صورت نیاز نصب می‌شوند.
  </Card>
  <Card title="مسیریابی چندعاملی" icon="route" href="/fa/concepts/multi-agent">
    نشست‌های جداگانه برای هر عامل، فضای کاری یا فرستنده.
  </Card>
  <Card title="پشتیبانی از رسانه" icon="image" href="/fa/nodes/images">
    تصاویر، صدا و اسناد را ارسال و دریافت کنید.
  </Card>
  <Card title="رابط کنترل وب" icon="monitor" href="/fa/web/control-ui">
    داشبورد مرورگر برای گفت‌وگو، پیکربندی، نشست‌ها و Nodeها.
  </Card>
  <Card title="Nodeهای همراه" icon="smartphone" href="/fa/nodes">
    Nodeهای iOS و Android را برای Canvas، دوربین و جریان‌های کاری صوتی جفت کنید.
  </Card>
</Columns>

## شروع سریع

<Steps>
  <Step title="نصب OpenClaw">
    ```bash
    npm install -g openclaw@latest
    ```
  </Step>
  <Step title="راه‌اندازی اولیه و نصب سرویس">
    ```bash
    openclaw onboard --install-daemon
    ```
  </Step>
  <Step title="گفت‌وگو">
    رابط کنترل را در مرورگر باز کنید و پیامی بفرستید:

    ```bash
    openclaw dashboard
    ```

    یا یک کانال متصل کنید ([Telegram](/fa/channels/telegram) سریع‌ترین است) و از تلفن خود گفت‌وگو کنید.

  </Step>
</Steps>

به نصب کامل و راه‌اندازی توسعه نیاز دارید؟ [شروع به کار](/fa/start/getting-started) را ببینید.

## داشبورد

پس از شروع Gateway، رابط کنترل مرورگر را باز کنید.

- پیش‌فرض محلی: [http://127.0.0.1:18789/](http://127.0.0.1:18789/)
- دسترسی از راه دور: [رابط‌های وب](/fa/web) و [Tailscale](/fa/gateway/tailscale)

<p align="center">
  <img src="/whatsapp-openclaw.jpg" alt="OpenClaw" width="420" />
</p>

## پیکربندی (اختیاری)

پیکربندی در `~/.openclaw/openclaw.json` قرار دارد.

- اگر **هیچ کاری نکنید**، OpenClaw از محیط اجرای عامل OpenClaw همراه استفاده می‌کند؛ پیام‌های مستقیم نشست اصلی عامل را به اشتراک می‌گذارند و هر گفت‌وگوی گروهی نشست مستقل خود را دریافت می‌کند.
- اگر می‌خواهید دسترسی را محدود کنید، از `channels.whatsapp.allowFrom` و (برای گروه‌ها) قواعد اشاره شروع کنید.

مثال:

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  messages: { groupChat: { mentionPatterns: ["@openclaw"] } },
}
```

## از اینجا شروع کنید

<Columns>
  <Card title="مراکز مستندات" href="/fa/start/hubs" icon="book-open">
    همه مستندات و راهنماها، سازمان‌دهی‌شده بر اساس مورد استفاده.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="settings">
    تنظیمات اصلی Gateway، توکن‌ها و پیکربندی ارائه‌دهنده.
  </Card>
  <Card title="دسترسی از راه دور" href="/fa/gateway/remote" icon="globe">
    الگوهای دسترسی SSH و tailnet.
  </Card>
  <Card title="کانال‌ها" href="/fa/channels/telegram" icon="message-square">
    راه‌اندازی مختص کانال برای Discord، Feishu، Microsoft Teams، Telegram، WhatsApp و موارد دیگر.
  </Card>
  <Card title="Nodeها" href="/fa/nodes" icon="smartphone">
    Nodeهای iOS و Android با جفت‌سازی، Canvas، دوربین و کنش‌های دستگاه.
  </Card>
  <Card title="راهنما" href="/fa/help" icon="life-buoy">
    نقطه شروع برای راه‌حل‌های رایج و رفع اشکال.
  </Card>
</Columns>

## بیشتر بیاموزید

<Columns>
  <Card title="فهرست کامل قابلیت‌ها" href="/fa/concepts/features" icon="list">
    قابلیت‌های کامل کانال، مسیریابی و رسانه.
  </Card>
  <Card title="مسیریابی چندعاملی" href="/fa/concepts/multi-agent" icon="route">
    جداسازی فضای کاری و نشست‌های مختص هر عامل.
  </Card>
  <Card title="امنیت" href="/fa/gateway/security" icon="shield">
    توکن‌ها، فهرست‌های مجاز و کنترل‌های ایمنی.
  </Card>
  <Card title="رفع اشکال" href="/fa/gateway/troubleshooting" icon="wrench">
    عیب‌یابی Gateway و خطاهای رایج.
  </Card>
  <Card title="درباره پروژه و قدردانی‌ها" href="/fa/reference/credits" icon="info">
    خاستگاه پروژه، مشارکت‌کنندگان و مجوز.
  </Card>
</Columns>
