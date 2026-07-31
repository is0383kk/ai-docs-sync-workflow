---
read_when:
    - می‌خواهید از اشتراک Claude Max با ابزارهای سازگار با OpenAI استفاده کنید
    - یک سرور API محلی می‌خواهید که Claude Code CLI را دربر بگیرد
    - می‌خواهید دسترسی به Anthropic مبتنی بر اشتراک را در مقایسه با دسترسی مبتنی بر کلید API ارزیابی کنید
summary: پروکسی جامعه برای ارائه اعتبارنامه‌های اشتراک Claude به‌صورت یک نقطه پایانی سازگار با OpenAI
title: پروکسی API کلود مکس
x-i18n:
    generated_at: "2026-07-27T16:04:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5d0d9a70e14d7d444e57e9bcf169816fec4013a2680dfc9b1761e6ab32109e9f
    source_path: providers/claude-max-api-proxy.md
    workflow: 16
---

**claude-max-api-proxy** یک بسته npm متعلق به جامعه است (نه یک Plugin برای OpenClaw) که
اشتراک Claude Max/Pro را به‌صورت یک نقطه پایانی API سازگار با OpenAI ارائه می‌کند؛ بنابراین
می‌توان به‌جای کلید API مربوط به Anthropic، هر ابزار سازگار با OpenAI را به اشتراک خود
متصل کرد.

<Warning>
این فقط سازگاری فنی است و مسیری نیست که رسماً تأیید شده باشد. Anthropic در
گذشته برخی استفاده‌های اشتراکی خارج از Claude Code را مسدود کرده است؛ پیش از اتکا
به این روش، قوانین فعلی صورت‌حساب Anthropic را بررسی کنید.

مستندات Claude Code متعلق به Anthropic، `claude -p` را به‌عنوان استفاده از Agent SDK/برنامه‌نویسی
توصیف می‌کنند. طبق به‌روزرسانی پشتیبانی Anthropic در 15 ژوئن 2026، Claude Agent SDK،
`claude -p` و استفاده از برنامه‌های شخص ثالث از محدودیت‌های استفاده اشتراک
واردشده استفاده می‌کنند (طرح اعتباری جداگانه Agent SDK که پیش‌تر اعلام شده بود،
متوقف شده است). مقاله Anthropic درباره [طرح Agent SDK
](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)،
مقاله‌های طرح [Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
و [Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
و نیز [ارائه‌دهنده Anthropic](/fa/providers/anthropic) را برای یادداشت‌های خود OpenClaw
درباره صورت‌حساب Claude CLI ببینید.
</Warning>

## چرا از این استفاده کنیم

| روش                       | مسیر هزینه                                              | مناسب برای                                      |
| ------------------------- | ------------------------------------------------------- | ----------------------------------------------- |
| کلید API مربوط به Anthropic | پرداخت به‌ازای هر توکن از طریق Claude Console           | برنامه‌های عملیاتی، خودکارسازی اشتراکی، حجم بالا |
| پروکسی اشتراک Claude      | قوانین طرح و اعتبار Claude Code / `claude -p` | آزمایش‌های شخصی با ابزارهای سازگار               |

این پروکسی امکان استفاده از اشتراک Claude Max یا Pro را با ابزارهای سازگار با OpenAI
فراهم می‌کند. این یک مسیر نامحدود با نرخ ثابت نیست — محدودیت‌های استفاده Claude Code
را به ارث می‌برد. برای استفاده عملیاتی، کلیدهای API همچنان مسیر شفاف‌تری برای
صورت‌حساب هستند.

## نحوه کار

```text
برنامه شما -> claude-max-api-proxy -> Claude Code CLI / claude -p -> Anthropic
     (قالب OpenAI)                  (تبدیل قالب)                    (استفاده از ورود شما)
```

پروکسی برای هر درخواست، Claude Code CLI را به‌عنوان یک زیرفرایند اجرا می‌کند،
درخواست‌های گفت‌وگوی دارای قالب OpenAI را به اعلان‌های CLI تبدیل می‌کند و پاسخ را
به‌صورت جریانی (یا یک‌جا) در قالب OpenAI بازمی‌گرداند.

## شروع به کار

<Steps>
  <Step title="نصب پروکسی">
    به Node.js 20+ و یک Claude Code CLI احراز هویت‌شده نیاز دارد.

    ```bash
    npm install -g claude-max-api-proxy

    # بررسی کنید Claude CLI احراز هویت شده باشد
    claude --version
    claude auth login   # اگر از قبل احراز هویت نشده است
    ```

  </Step>
  <Step title="راه‌اندازی سرور">
    ```bash
    claude-max-api
    # سرور در http://localhost:3456 اجرا می‌شود
    ```
  </Step>
  <Step title="آزمایش پروکسی">
    ```bash
    curl http://localhost:3456/health
    curl http://localhost:3456/v1/models

    curl http://localhost:3456/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d '{
        "model": "claude-opus-4",
        "messages": [{"role": "user", "content": "سلام!"}]
      }'
    ```

  </Step>
  <Step title="پیکربندی OpenClaw">
    OpenClaw را طوری تنظیم کنید که از پروکسی به‌عنوان یک نقطه پایانی سفارشی سازگار با OpenAI استفاده کند:

    ```json5
    {
      env: {
        OPENAI_API_KEY: "not-needed",
        OPENAI_BASE_URL: "http://localhost:3456/v1",
      },
      agents: {
        defaults: {
          model: { primary: "openai/claude-opus-4" },
        },
      },
    }
    ```

  </Step>
</Steps>

<Note>
شناسه‌های مدل زیر متعلق به کاتالوگ خود پروکسی هستند، نه ارجاع‌های مدل Anthropic در
OpenClaw. هر شناسه به یک نام مستعار مدل Claude Code CLI نگاشت می‌شود (`opus`، `sonnet`،
`haiku`)؛ بنابراین هر زمان Anthropic آن نام مستعار را در CLI به‌روزرسانی کند،
مدل زیربنایی تغییر می‌کند. پیش از اتکا به یک نگاشت مشخص، README فعلی پروکسی را
بررسی کنید.
</Note>

| شناسه مدل            | نام مستعار CLI | نگاشت فعلی      |
| -------------------- | -------------- | --------------- |
| `claude-opus-4`   | `opus`    | Claude Opus 4.5 |
| `claude-sonnet-4` | `sonnet`  | Claude Sonnet 4 |
| `claude-haiku-4`  | `haiku`   | Claude Haiku 4  |

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="نکات مربوط به سازگاری با OpenAI به سبک پروکسی">
    این روش از مسیر سفارشی و عمومی `/v1` سازگار با OpenAI در OpenClaw استفاده می‌کند؛ همان
    مسیری که هر بک‌اند خودمیزبان سازگار با OpenAI استفاده می‌کند:

    - شکل‌دهی درخواست مخصوص OpenAI بومی اعمال نمی‌شود.
    - `/fast` و `service_tier` فقط برای ترافیک مستقیم `api.anthropic.com`
      اعمال می‌شوند؛ مسیرهای پروکسی، `service_tier` را بدون تغییر باقی می‌گذارند (به
      [حالت سریع ارائه‌دهنده Anthropic](/fa/providers/anthropic#advanced-configuration) مراجعه کنید).
    - هیچ `store` برای Responses، راهنمای حافظه نهان اعلان یا شکل‌دهی بار داده
      سازگاری استدلال OpenAI وجود ندارد.
    - سرآیندهای انتساب OpenAI/Codex در OpenClaw (`originator`، `version`،
      `User-Agent`) فقط در ترافیک OAuth بومی `api.openai.com` ارسال می‌شوند، نه
      به مقصدهای سفارشی `OPENAI_BASE_URL` مانند این پروکسی.

  </Accordion>

  <Accordion title="راه‌اندازی خودکار در macOS با LaunchAgent">
    ```bash
    cat > ~/Library/LaunchAgents/com.claude-max-api.plist << 'EOF'
    <?xml version="1.0" encoding="UTF-8"?>
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
    <dict>
      <key>Label</key>
      <string>com.claude-max-api</string>
      <key>RunAtLoad</key>
      <true/>
      <key>KeepAlive</key>
      <true/>
      <key>ProgramArguments</key>
      <array>
        <string>/usr/local/bin/node</string>
        <string>/usr/local/lib/node_modules/claude-max-api-proxy/dist/server/standalone.js</string>
      </array>
      <key>EnvironmentVariables</key>
      <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/opt/homebrew/bin:~/.local/bin:/usr/bin:/bin</string>
      </dict>
    </dict>
    </plist>
    EOF

    launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.claude-max-api.plist
    ```

  </Accordion>
</AccordionGroup>

## نکات

- رفتار صورت‌حساب، اعتبار استفاده و محدودیت نرخ `claude -p` در Claude Code را به ارث می‌برد.
- فقط به `127.0.0.1` متصل می‌شود؛ به‌جز فراخوانی خود CLI به Anthropic، داده‌ای به هیچ سرور شخص ثالثی ارسال نمی‌کند.
- پاسخ‌های جریانی پشتیبانی می‌شوند.
- خطاهای احراز هویت هنگام راه‌اندازی بررسی نمی‌شوند و فقط زمانی نمایان می‌شوند که یک درخواست گفت‌وگو واقعاً اجرا شود؛ اگر CLI احراز هویت نشده باشد، انتظار می‌رود نخستین درخواست شکست بخورد، نه اینکه سرور از راه‌اندازی خودداری کند.

<Note>
برای یکپارچه‌سازی بومی Anthropic با Claude CLI یا کلیدهای API، به [ارائه‌دهنده Anthropic](/fa/providers/anthropic) مراجعه کنید. برای اشتراک‌های OpenAI/Codex، [ارائه‌دهنده OpenAI](/fa/providers/openai) را ببینید.
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهنده Anthropic" href="/fa/providers/anthropic" icon="bolt">
    یکپارچه‌سازی بومی OpenClaw با Claude CLI یا کلیدهای API.
  </Card>
  <Card title="ارائه‌دهنده OpenAI" href="/fa/providers/openai" icon="robot">
    برای اشتراک‌های OpenAI/Codex.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    نمای کلی همه ارائه‌دهندگان، ارجاع‌های مدل و رفتار انتقال در صورت خرابی.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی.
  </Card>
</CardGroup>
