---
read_when:
    - فهرست کاملی از قابلیت‌هایی که OpenClaw پشتیبانی می‌کند می‌خواهید
summary: قابلیت‌های OpenClaw در کانال‌ها، مسیریابی، رسانه و تجربه کاربری.
title: ویژگی‌ها
x-i18n:
    generated_at: "2026-07-27T15:03:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bc3ebdd87a0f6ea0f3d75d029bf7cae469ecd9db84a165bd47c4896936fe303
    source_path: concepts/features.md
    workflow: 16
---

## نکات برجسته

<Columns>
  <Card title="کانال‌ها" icon="message-square" href="/fa/channels">
    Discord، iMessage، Signal، Slack، Telegram، WhatsApp، WebChat و موارد دیگر با یک Gateway.
  </Card>
  <Card title="Pluginها" icon="plug" href="/fa/tools/plugin">
    Pluginهای رسمی، Matrix، Nextcloud Talk، Nostr، Twitch و Zalo و ده‌ها مورد دیگر را با یک فرمان نصب اضافه می‌کنند.
  </Card>
  <Card title="مسیریابی" icon="route" href="/fa/concepts/multi-agent">
    مسیریابی چندعاملی با نشست‌های مجزا.
  </Card>
  <Card title="رسانه" icon="image" href="/fa/nodes/images">
    تصویر، صدا، ویدئو، سند و تولید تصویر/ویدئو.
  </Card>
  <Card title="برنامه‌ها و رابط کاربری" icon="monitor" href="/fa/platforms">
    Windows Hub، رابط کاربری Control مبتنی بر مرورگر، برنامه نوار منوی macOS و Nodeهای همراه.
  </Card>
  <Card title="Nodeهای همراه" icon="smartphone" href="/fa/nodes">
    Nodeهای iOS و Android با جفت‌سازی، صدا/گفت‌وگو و فرمان‌های پیشرفته دستگاه.
  </Card>
</Columns>

## فهرست کامل

**کانال‌ها:**

- iMessage، Telegram و WebChat همراه نصب هسته ارائه می‌شوند؛ هر کانال دیگری یک
  Plugin رسمی است که با `openclaw plugins install @openclaw/<id>` نصب می‌شود (یا در صورت نیاز
  هنگام `openclaw onboard` / `openclaw channels add`)
- کانال‌های Plugin رسمی: Discord، Feishu، Google Chat، IRC، LINE، Matrix، Mattermost،
  Microsoft Teams، Nextcloud Talk، Nostr، QQ Bot، Raft، Signal، Slack، SMS، Synology Chat،
  Tlon، Twitch، Voice Call، WhatsApp، Zalo و Zalo Personal
- کانال‌های Plugin خارجی که خارج از مخزن OpenClaw نگهداری می‌شوند: WeChat، Yuanbao و Zalo ClawBot
- پشتیبانی از گفت‌وگوی گروهی با فعال‌سازی مبتنی بر اشاره
- ایمنی پیام مستقیم با فهرست‌های مجاز و جفت‌سازی

**عامل:**

- زمان‌اجرای عاملِ تعبیه‌شده با پخش جریانی ابزار
- مسیریابی چندعاملی با نشست‌های مجزا برای هر فضای کاری یا فرستنده
- نشست‌ها: گفت‌وگوهای مستقیم در `main` مشترک ادغام می‌شوند؛ گروه‌ها مجزا هستند
- پخش جریانی و قطعه‌بندی پاسخ‌های طولانی

**احراز هویت و ارائه‌دهندگان:**

- بیش از 35 ارائه‌دهنده مدل (Anthropic، OpenAI، Google و موارد دیگر)
- احراز هویت اشتراک از طریق OAuth (برای نمونه OpenAI Codex)
- پشتیبانی از ارائه‌دهندگان سفارشی و خودمیزبان (vLLM، SGLang، Ollama، llama.cpp، LM Studio و
  هر نقطه پایانی سازگار با OpenAI یا Anthropic)

**رسانه:**

- ورودی و خروجی تصویر، صدا، ویدئو و سند
- سطوح قابلیت مشترک برای تولید تصویر و ویدئو
- رونویسی یادداشت صوتی
- تبدیل متن به گفتار با چندین ارائه‌دهنده

**برنامه‌ها و رابط‌ها:**

- WebChat و رابط کاربری Control مبتنی بر مرورگر
- برنامه همراه نوار منوی macOS
- Node مربوط به iOS با جفت‌سازی، Canvas، دوربین، ضبط صفحه، موقعیت مکانی و صدا
- Node مربوط به Android با جفت‌سازی، گفت‌وگو، صدا، Canvas، دوربین و فرمان‌های دستگاه

**ابزارها و خودکارسازی:**

- خودکارسازی مرورگر، اجرا و محیط ایزوله
- جست‌وجوی وب (Brave، DuckDuckGo، Exa، Firecrawl، Gemini، Grok، Kimi، MiniMax Search، Ollama Web Search، Perplexity، SearXNG، Tavily)
- کارهای Cron و زمان‌بندی Heartbeat
- Skills، Pluginها و پایپ‌لاین‌های گردش کار (Lobster)

## مرتبط

<CardGroup cols={2}>
  <Card title="قابلیت‌های آزمایشی" href="/fa/concepts/experimental-features" icon="flask">
    قابلیت‌های انتخابی که هنوز در سطح پیش‌فرض عرضه نشده‌اند.
  </Card>
  <Card title="زمان‌اجرای عامل" href="/fa/concepts/agent" icon="robot">
    مدل زمان‌اجرای عامل و نحوه اعزام اجراها.
  </Card>
  <Card title="کانال‌ها" href="/fa/channels" icon="message-square">
    Telegram، WhatsApp، Discord، Slack و موارد دیگر را از یک Gateway متصل کنید.
  </Card>
  <Card title="Pluginها" href="/fa/tools/plugin" icon="plug">
    Pluginهای رسمی و خارجی که OpenClaw را گسترش می‌دهند.
  </Card>
</CardGroup>
