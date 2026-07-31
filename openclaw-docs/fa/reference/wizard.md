---
read_when:
    - جست‌وجوی یک مرحله یا پرچم مشخص در فرایند راه‌اندازی اولیه
    - خودکارسازی راه‌اندازی اولیه با حالت غیرتعاملی
    - اشکال‌زدایی رفتار راه‌اندازی اولیه
sidebarTitle: Onboarding Reference
summary: 'مرجع کامل راه‌اندازی اولیه با CLI: همه مراحل، پرچم‌ها و فیلدهای پیکربندی'
title: مرجع راه‌اندازی اولیه
x-i18n:
    generated_at: "2026-07-27T15:55:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e5e7e42fa3fc1a6d85ad422d0d28dfeda233c89a4d7e97eee4fb974831816372
    source_path: reference/wizard.md
    workflow: 16
---

این مرجع کامل `openclaw onboard` است.
برای نمایی کلی، به [راه‌اندازی اولیه (CLI)](/fa/start/wizard) مراجعه کنید. برای رفتار و خروجی‌های
گام‌به‌گام، [مرجع راه‌اندازی CLI](/fa/start/wizard-cli-reference) را ببینید.

## جزئیات جریان (حالت محلی)

<Steps>
  <Step title="بازنشانی (اختیاری)">
    - `--reset` پیش از اجرای راه‌اندازی، وضعیت را بازنشانی می‌کند؛ بدون آن، اجرای دوباره راه‌اندازی اولیه
      پیکربندی موجود را حفظ می‌کند و آن را به‌عنوان مقادیر پیش‌فرض دوباره به‌کار می‌برد.
    - `--reset-scope` تعیین می‌کند `--reset` چه چیزهایی را حذف کند: `config` (فقط فایل
      پیکربندی)، `config+creds+sessions` (پیش‌فرض)، یا `full` (فضای کاری را نیز
      حذف می‌کند).
    - اگر فایل پیکربندی نامعتبر باشد، راه‌اندازی اولیه متوقف می‌شود و اعلام می‌کند ابتدا
      `openclaw doctor` را اجرا کنید، سپس راه‌اندازی را دوباره اجرا کنید.
    - بازنشانی، وضعیت را به سطل زباله منتقل می‌کند (هرگز مستقیماً حذف نمی‌کند).

  </Step>
  <Step title="پذیرش خطر">
    - در نخستین اجرا (یا هر اجرایی پیش از تنظیم‌شدن `wizard.securityAcknowledgedAt`)
      از شما خواسته می‌شود تأیید کنید که می‌دانید عامل‌ها قدرتمندند و دسترسی کامل
      به سیستم خطرناک است.
    - `--non-interactive` صراحتاً به `--accept-risk` نیاز دارد؛ بدون آن،
      راه‌اندازی اولیه به‌جای نمایش درخواست، با خطا خارج می‌شود.
    - در اجراهای تعاملی، به‌جای پرچم یک درخواست تأیید نمایش داده می‌شود؛ ردکردن آن
      راه‌اندازی را لغو می‌کند.

  </Step>
  <Step title="مدل/احراز هویت">
    - **کلید API شرکت Anthropic**: در صورت وجود از `ANTHROPIC_API_KEY` استفاده می‌کند یا کلید را درخواست می‌کند، سپس آن را برای استفاده دیمن ذخیره می‌کند.
    - **CLI مدل Anthropic Claude**: وقتی ورود Claude CLI از قبل وجود داشته باشد، مسیر محلی ترجیحی است؛ OpenClaw همچنان احراز هویت Anthropic با توکن راه‌اندازی را به‌عنوان گزینه جایگزین پشتیبانی می‌کند.
    - **اشتراک OpenAI Code (Codex) ‏(OAuth)**: جریان مرورگر؛ `code#state` را جای‌گذاری کنید.
      - در راه‌اندازی تازه و بدون مدل اصلی، `agents.defaults.model` را از طریق زمان‌اجرای Codex روی `openai/gpt-5.6-sol` تنظیم می‌کند.
    - **اشتراک OpenAI Code (Codex) ‏(جفت‌سازی دستگاه)**: جریان جفت‌سازی مرورگر با کد دستگاه کوتاه‌عمر.
      - در راه‌اندازی تازه و بدون مدل اصلی، `agents.defaults.model` را از طریق زمان‌اجرای Codex روی `openai/gpt-5.6-sol` تنظیم می‌کند.
    - **کلید API شرکت OpenAI**: در صورت وجود از `OPENAI_API_KEY` استفاده می‌کند یا کلید را درخواست می‌کند، سپس آن را در نمایه‌های احراز هویت ذخیره می‌کند.
      - در راه‌اندازی تازه و بدون مدل اصلی، `agents.defaults.model` را روی `openai/gpt-5.6` تنظیم می‌کند؛ شناسه ساده مدل API مستقیم به سطح Sol تفکیک می‌شود.
    - افزودن یا احراز هویت مجدد OpenAI، مدل اصلی صریح موجود، از جمله `openai/gpt-5.5`، را حفظ می‌کند. اگر حساب GPT-5.6 را ارائه نمی‌کند، `openai/gpt-5.5` را صراحتاً انتخاب کنید؛ OpenClaw مدل را بی‌سروصدا به نسخه پایین‌تر تنزل نمی‌دهد.
    - **OAuth سرویس xAI**: ورود از طریق مرورگر با کد دستگاه که به فراخوانی برگشتی localhost نیاز ندارد؛ بنابراین روی SSH/Docker/VPS نیز کار می‌کند (`--auth-choice xai-oauth`).
    - **کلید API سرویس xAI**: ‏`XAI_API_KEY` را درخواست می‌کند (`--auth-choice xai-api-key`).
    - `--auth-choice xai-device-code` همچنان به‌عنوان نام مستعار سازگاریِ صرفاً دستی برای همان جریان OAuth کد دستگاه xAI کار می‌کند؛ برای اسکریپت‌های جدید از `xai-oauth` استفاده کنید.
    - **OpenCode**: ‏`OPENCODE_API_KEY` (یا `OPENCODE_ZEN_API_KEY` که می‌توانید آن را از https://opencode.ai/auth دریافت کنید) را درخواست می‌کند و امکان انتخاب کاتالوگ Zen یا Go را می‌دهد.
    - **Ollama**: ابتدا گزینه‌های **ابری + محلی**، **فقط ابری** یا **فقط محلی** را ارائه می‌کند. `Cloud only`، ‏`OLLAMA_API_KEY` را درخواست و از `https://ollama.com` استفاده می‌کند؛ حالت‌های متکی بر میزبان، نشانی URL پایه Ollama (پیش‌فرض `http://127.0.0.1:11434`) را درخواست می‌کنند، مدل‌های موجود را کشف می‌کنند و در صورت نیاز مدل محلی انتخاب‌شده را خودکار دریافت می‌کنند؛ `Cloud + Local` همچنین بررسی می‌کند که آیا آن میزبان Ollama برای دسترسی ابری وارد حساب شده است یا خیر.
    - جزئیات بیشتر: [Ollama](/fa/providers/ollama)
    - **کلید API**: کلید را برای شما ذخیره می‌کند.
    - **Vercel AI Gateway (پراکسی چندمدلی)**: ‏`AI_GATEWAY_API_KEY` را درخواست می‌کند.
    - جزئیات بیشتر: [Vercel AI Gateway](/fa/providers/vercel-ai-gateway)
    - **Cloudflare AI Gateway**: شناسه حساب، شناسه Gateway و `CLOUDFLARE_AI_GATEWAY_API_KEY` را درخواست می‌کند.
    - جزئیات بیشتر: [Cloudflare AI Gateway](/fa/providers/cloudflare-ai-gateway)
    - **MiniMax**: پیکربندی به‌طور خودکار نوشته می‌شود؛ پیش‌فرض میزبانی‌شده `MiniMax-M3` است.
      راه‌اندازی با کلید API از `minimax/...` و راه‌اندازی OAuth از
      `minimax-portal/...` استفاده می‌کند.
    - جزئیات بیشتر: [MiniMax](/fa/providers/minimax)
    - **StepFun**: پیکربندی برای حالت استاندارد StepFun یا Step Plan در نقاط پایانی چین یا جهانی به‌طور خودکار نوشته می‌شود.
    - حالت استاندارد در حال حاضر به‌طور پیش‌فرض از `step-3.5-flash` استفاده می‌کند؛ Step Plan همچنین شامل `step-3.5-flash-2603` است.
    - جزئیات بیشتر: [StepFun](/fa/providers/stepfun)
    - **Synthetic (سازگار با Anthropic)**: ‏`SYNTHETIC_API_KEY` را درخواست می‌کند.
    - جزئیات بیشتر: [Synthetic](/fa/providers/synthetic)
    - **Moonshot (Kimi K2)**: پیکربندی به‌طور خودکار نوشته می‌شود.
    - **Kimi Coding**: پیکربندی به‌طور خودکار نوشته می‌شود.
    - جزئیات بیشتر: [Moonshot AI (Kimi + Kimi Coding)](/fa/providers/moonshot)
    - **ارائه‌دهنده سفارشی**: با نقاط پایانی سازگار با OpenAI، سازگار با OpenAI Responses یا سازگار با Anthropic کار می‌کند. پرچم‌های غیرتعاملی: `--auth-choice custom-api-key`، ‏`--custom-base-url`، ‏`--custom-model-id`، ‏`--custom-api-key` (اختیاری؛ در صورت نبود به `CUSTOM_API_KEY` برمی‌گردد)، `--custom-provider-id` (اختیاری؛ به‌طور خودکار از نشانی URL پایه استخراج می‌شود)، `--custom-compatibility openai|openai-responses|anthropic` (پیش‌فرض `openai`)، ‏`--custom-image-input` / `--custom-text-input` (تشخیص استنباط‌شده مدل بینایی را بازنویسی می‌کند).
    - **ردکردن**: هنوز هیچ احراز هویتی پیکربندی نمی‌شود.
    - یک مدل پیش‌فرض از میان گزینه‌های شناسایی‌شده انتخاب کنید (یا ارائه‌دهنده/مدل را دستی وارد کنید). برای بهترین کیفیت و کاهش خطر تزریق پرامپت، قوی‌ترین مدل از جدیدترین نسل موجود در پشته ارائه‌دهنده خود را انتخاب کنید.
    - راه‌اندازی اولیه مدل را بررسی می‌کند و اگر مدل پیکربندی‌شده ناشناخته باشد یا احراز هویت نداشته باشد، هشدار می‌دهد.
    - حالت ذخیره‌سازی کلید API به‌طور پیش‌فرض مقادیر متن ساده نمایه احراز هویت است. برای ذخیره ارجاع‌های متکی بر متغیر محیطی، به‌جای آن از `--secret-input-mode ref` استفاده کنید (برای نمونه `keyRef: { source: "env", provider: "default", id: "OPENAI_API_KEY" }`)؛ متغیر محیطی مورد ارجاع باید از قبل تنظیم شده باشد، وگرنه راه‌اندازی اولیه فوراً شکست می‌خورد.
    - نمایه‌های احراز هویت در `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` قرار دارند (کلیدهای API + ‏OAuth). ‏`~/.openclaw/credentials/oauth.json` فقط برای واردکردن داده‌های قدیمی است.
    - جزئیات بیشتر: [OAuth](/fa/concepts/oauth)
    <Note>
    نکته برای محیط بدون رابط گرافیکی/سرور: OAuth را روی دستگاهی دارای مرورگر تکمیل کنید، سپس
    `auth-profiles.json` آن عامل را (برای نمونه
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` یا مسیر متناظر
    `$OPENCLAW_STATE_DIR/...`) به میزبان Gateway کپی کنید. `credentials/oauth.json`
    فقط یک منبع قدیمی برای واردکردن داده‌ها است.
    </Note>
  </Step>
  <Step title="فضای کاری">
    - پیش‌فرض `~/.openclaw/workspace` است (قابل پیکربندی).
    - فایل‌های فضای کاری موردنیاز برای آیین راه‌اندازی عامل را ایجاد می‌کند.
    - طرح کامل فضای کاری + راهنمای پشتیبان‌گیری: [فضای کاری عامل](/fa/concepts/agent-workspace)

  </Step>
  <Step title="Gateway">
    - درگاه (پیش‌فرض **18789**)، اتصال، حالت احراز هویت، دسترسی از طریق Tailscale.
    - توصیه احراز هویت: حتی برای loopback نیز **توکن** را حفظ کنید تا کلاینت‌های محلی WS ملزم به احراز هویت باشند.
    - در حالت توکن، راه‌اندازی تعاملی این گزینه‌ها را ارائه می‌کند:
      - **تولید/ذخیره توکن متن ساده** (پیش‌فرض)
      - **استفاده از SecretRef** (اختیاری)
      - راه‌اندازی سریع، SecretRefهای موجود `gateway.auth.token` را در میان ارائه‌دهندگان `env`، ‏`file` و `exec` برای وارسی راه‌اندازی اولیه/راه‌اندازی داشبورد دوباره به‌کار می‌برد.
      - اگر آن SecretRef پیکربندی شده باشد اما قابل تفکیک نباشد، راه‌اندازی اولیه به‌جای تضعیف بی‌سروصدای احراز هویت زمان‌اجرا، زودهنگام و با پیام اصلاحی روشن شکست می‌خورد.
    - در حالت گذرواژه، راه‌اندازی تعاملی از ذخیره‌سازی متن ساده یا SecretRef نیز پشتیبانی می‌کند.
    - مسیر SecretRef توکن غیرتعاملی: `--gateway-token-ref-env <ENV_VAR>`.
      - به یک متغیر محیطی غیرخالی در محیط فرایند راه‌اندازی اولیه نیاز دارد.
      - نمی‌توان آن را با `--gateway-token` ترکیب کرد.
    - فقط زمانی احراز هویت را غیرفعال کنید که به همه فرایندهای محلی کاملاً اعتماد دارید.
    - اتصال‌های غیر-loopback همچنان به احراز هویت نیاز دارند.

  </Step>
  <Step title="کانال‌ها">
    - [WhatsApp](/fa/channels/whatsapp): ورود اختیاری با کد QR.
    - [Telegram](/fa/channels/telegram): توکن ربات.
    - [Discord](/fa/channels/discord): توکن ربات.
    - [Google Chat](/fa/channels/googlechat): ‏JSON حساب سرویس + مخاطب Webhook.
    - [Mattermost](/fa/channels/mattermost) ‏(Plugin): توکن ربات + نشانی URL پایه.
    - [Signal](/fa/channels/signal) ‏(Plugin): نصب اختیاری `signal-cli` + پیکربندی حساب.
    - [iMessage](/fa/channels/imessage): مسیر CLI ‏`imsg` + دسترسی به پایگاه داده Messages؛ هنگامی که Gateway خارج از Mac اجرا می‌شود، از یک پوشش SSH استفاده کنید.
    - Discord، ‏Feishu، ‏Microsoft Teams، ‏QQ Bot، ‏Slack و کانال‌های دیگر به‌صورت
      Plugin عرضه می‌شوند که راه‌اندازی اولیه می‌تواند آن‌ها را برای شما نصب کند. کاتالوگ کامل: [کانال‌ها](/fa/channels).
    - امنیت پیام مستقیم: پیش‌فرض جفت‌سازی است. نخستین پیام مستقیم کدی ارسال می‌کند؛ آن را از طریق `openclaw pairing approve <channel> <code>` تأیید کنید یا از فهرست‌های مجاز استفاده کنید.

  </Step>
  <Step title="جست‌وجوی وب">
    - یک ارائه‌دهنده پشتیبانی‌شده مانند Brave، ‏Codex (Hosted Search)، ‏DuckDuckGo، ‏Exa، ‏Firecrawl، ‏Gemini، ‏Grok، ‏Kimi، ‏MiniMax Search، ‏Ollama Web Search، ‏Parallel، ‏Perplexity، ‏SearXNG یا Tavily را انتخاب کنید (یا رد کنید).
    - ارائه‌دهندگان متکی بر API می‌توانند برای راه‌اندازی سریع از متغیرهای محیطی یا پیکربندی موجود استفاده کنند؛ ارائه‌دهندگان بدون کلید در عوض از پیش‌نیازهای خاص ارائه‌دهنده خود استفاده می‌کنند.
    - با `--skip-search` رد کنید.
    - پیکربندی در آینده: `openclaw configure --section web`.

  </Step>
  <Step title="نصب دیمن">
    - macOS: ‏LaunchAgent
      - به نشست کاربر واردشده نیاز دارد؛ برای محیط بدون رابط گرافیکی، از LaunchDaemon سفارشی استفاده کنید (عرضه نمی‌شود).
    - Linux (و Windows از طریق WSL2): واحد کاربری systemd
      - راه‌اندازی اولیه تلاش می‌کند ماندگاری را از طریق `loginctl enable-linger <user>` فعال کند تا Gateway پس از خروج کاربر نیز فعال بماند.
      - ممکن است sudo را درخواست کند (`/var/lib/systemd/linger` را می‌نویسد)؛ ابتدا بدون sudo تلاش می‌کند.
    - Windows بومی: ابتدا Scheduled Task؛ اگر ایجاد وظیفه رد شود، OpenClaw به یک مورد ورود در پوشه Startup هر کاربر برمی‌گردد و Gateway را فوراً راه‌اندازی می‌کند.
    - **انتخاب زمان‌اجرا:** ‏Node الزامی است، زیرا مخزن متعارف وضعیت زمان‌اجرا از `node:sqlite` استفاده می‌کند. سرویس‌های قدیمی Bun هنگام تعمیر به Node مهاجرت داده می‌شوند.
    - اگر احراز هویت توکنی به توکن نیاز داشته باشد و `gateway.auth.token` با SecretRef مدیریت شود، نصب دیمن آن را اعتبارسنجی می‌کند اما مقادیر متن ساده تفکیک‌شده توکن را در فراداده محیط سرویس ناظر ماندگار نمی‌کند.
    - اگر احراز هویت توکنی به توکن نیاز داشته باشد و SecretRef توکن پیکربندی‌شده قابل تفکیک نباشد، نصب دیمن با راهنمایی عملی مسدود می‌شود.
    - اگر هر دو `gateway.auth.token` و `gateway.auth.password` پیکربندی شده باشند و `gateway.auth.mode` تنظیم نشده باشد، نصب دیمن تا زمان تنظیم صریح حالت مسدود می‌شود.

  </Step>
  <Step title="بررسی سلامت">
    - Gateway را راه‌اندازی می‌کند (در صورت نیاز) و `openclaw health` را اجرا می‌کند.
    - نکته: `openclaw status --deep` وارسی زنده سلامت Gateway را، شامل وارسی کانال‌ها در صورت پشتیبانی، به خروجی وضعیت اضافه می‌کند (به یک Gateway قابل دسترسی نیاز دارد).

  </Step>
  <Step title="Skills (توصیه‌شده)">
    - Skills موجود را می‌خواند و الزامات را بررسی می‌کند.
    - امکان انتخاب مدیر Node را می‌دهد: **npm / pnpm / bun**.
    - وابستگی‌های اختیاری Skills بسته‌بندی‌شده و مورداعتماد را به‌طور خودکار نصب می‌کند (برخی در macOS از Homebrew استفاده می‌کنند).
    - Skillsی را که پیش‌نیاز نصب‌کننده Homebrew، ‏uv یا Go آن‌ها موجود نیست رد می‌کند، آن‌ها را همراه با راهنمای راه‌اندازی دستی گروه‌بندی می‌کند و پس از نصب پیش‌نیاز شما را به `openclaw doctor` هدایت می‌کند.

  </Step>
  <Step title="پایان">
    - خلاصه + گام‌های بعدی، شامل درخواست **چگونه می‌خواهید عامل خود را از تخم بیرون بیاورید؟** برای Terminal، ‏Browser یا بعداً.

  </Step>
</Steps>

<Note>
اگر هیچ رابط کاربری گرافیکی شناسایی نشود، راه‌اندازی به‌جای باز کردن مرورگر، دستورالعمل‌های انتقال پورت SSH برای رابط کاربری کنترل را نمایش می‌دهد.
اگر دارایی‌های رابط کاربری کنترل موجود نباشند، راه‌اندازی تلاش می‌کند آن‌ها را بسازد؛ گزینهٔ جایگزین `pnpm ui:build` است (وابستگی‌های رابط کاربری را به‌طور خودکار نصب می‌کند).
</Note>

## حالت غیرتعاملی

برای خودکارسازی یا اسکریپت‌نویسی راه‌اندازی از `--non-interactive --accept-risk` استفاده کنید (این
پرچم تأیید الزامی پذیرش ریسک است؛ راه‌اندازی بدون آن
با خطا خاتمه می‌یابد):

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --gateway-port 18789 \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-skills
```

برای دریافت خلاصه‌ای قابل‌خواندن توسط ماشین، `--json` را اضافه کنید.

SecretRef توکن Gateway در حالت غیرتعاملی:

```bash
export OPENCLAW_GATEWAY_TOKEN="your-token"
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice skip \
  --gateway-auth token \
  --gateway-token-ref-env OPENCLAW_GATEWAY_TOKEN
```

`--gateway-token` و `--gateway-token-ref-env` ناسازگار با یکدیگرند.

<Note>
`--json` به‌معنای حالت غیرتعاملی **نیست**. برای اسکریپت‌ها از `--non-interactive --accept-risk` (و `--workspace`) استفاده کنید.
</Note>

نمونه‌فرمان‌های مختص ارائه‌دهندگان در [خودکارسازی CLI](/fa/start/wizard-cli-automation#provider-specific-examples) قرار دارند.
برای معنای پرچم‌ها و ترتیب مراحل از این صفحهٔ مرجع استفاده کنید.

### افزودن عامل (غیرتعاملی)

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

`main` شناسهٔ رزروشدهٔ عامل است و نمی‌توان از آن برای `openclaw agents add` استفاده کرد.

## RPC راه‌انداز Gateway

Gateway جریان راه‌اندازی را از طریق RPC ارائه می‌کند (`wizard.start`، `wizard.next`، `wizard.cancel`، `wizard.status`).
کلاینت‌ها (برنامهٔ macOS و رابط کاربری کنترل) می‌توانند مراحل را بدون پیاده‌سازی دوبارهٔ منطق راه‌اندازی نمایش دهند.

## راه‌اندازی Signal (signal-cli)

راه‌اندازی تشخیص می‌دهد که آیا `signal-cli` در `PATH` قرار دارد یا خیر و اگر موجود نباشد، نصب آن را پیشنهاد می‌کند:

- Linux x86-64: ساخت بومی رسمی GraalVM را از انتشارهای GitHub مربوط به `signal-cli` بارگیری می‌کند و آن را در `~/.openclaw/tools/signal-cli/<version>/` ذخیره می‌کند.
- macOS و معماری‌های دیگر: در عوض از طریق Homebrew نصب می‌کند.
- Windows بومی: هنوز پشتیبانی نمی‌شود؛ راه‌اندازی را درون WSL2 اجرا کنید تا از مسیر نصب Linux استفاده شود.
- در هر دو حالت، `channels.signal.transport.cliPath` را با `kind: "managed-native"` می‌نویسد.

## مواردی که راه‌انداز می‌نویسد

فیلدهای معمول در `~/.openclaw/openclaw.json`:

- `agents.defaults.workspace`
- `agents.defaults.skipBootstrap` هنگام ارسال `--skip-bootstrap`
- `agents.defaults.model` / `models.providers` (اگر Minimax انتخاب شود)
- `tools.profile` (اگر تنظیم نشده باشد، راه‌اندازی محلی به‌طور پیش‌فرض از `"coding"` استفاده می‌کند؛ مقادیر صریح موجود حفظ می‌شوند)
- `gateway.*` (حالت، اتصال، احراز هویت، tailscale)
- `session.dmScope` (راه‌اندازی مقادیر صریح را حفظ می‌کند و در غیر این صورت آن را تنظیم‌نشده باقی می‌گذارد؛ بنابراین مقدار پیش‌فرض `"main"` همهٔ پیام‌های مستقیم کانال‌ها را در نشست اصلی چرخشی عامل نگه می‌دارد—پیش‌فرض عامل شخصی. برای صندوق‌های ورودی مشترک یا چندکاربره، از `"per-channel-peer"` استفاده کنید؛ `openclaw security audit` هنگام شناسایی ترافیک پیام مستقیم چندکاربره، جداسازی را توصیه می‌کند. جزئیات: [مرجع راه‌اندازی CLI](/fa/start/wizard-cli-reference#outputs-and-internals))
- `channels.telegram.botToken`، `channels.discord.token`، `channels.matrix.*`، `channels.signal.*`، `channels.imessage.*`
- فهرست‌های مجاز پیام مستقیم کانال هنگامی که در اعلان‌های کانال آن‌ها را فعال می‌کنید. Discord، Matrix، Microsoft Teams و Slack در صورت امکان نام‌ها را به شناسه تبدیل می‌کنند؛ کانال‌های دیگر مستقیماً شناسه می‌گیرند (برای مثال، شناسه‌های عددی فرستنده در Telegram یا شماره‌تلفن‌های WhatsApp).
- `skills.install.nodeManager`
  - `setup --node-manager` مقادیر `npm`، `pnpm` یا `bun` را می‌پذیرد.
  - پیکربندی دستی همچنان می‌تواند با تنظیم مستقیم `skills.install.nodeManager` از `yarn` استفاده کند.
- `wizard.lastRunAt`
- `wizard.lastRunVersion`
- `wizard.lastRunCommit`
- `wizard.lastRunCommand`
- `wizard.lastRunMode`
- `wizard.securityAcknowledgedAt`

`openclaw agents add`، `agents.entries.*` و در صورت نیاز `bindings` را می‌نویسد.

اعتبارنامه‌های WhatsApp در `~/.openclaw/credentials/whatsapp/<accountId>/` قرار می‌گیرند.
نشست‌های فعال و رونوشت‌ها در
`~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` ذخیره می‌شوند. پوشهٔ
`~/.openclaw/agents/<agentId>/sessions/` برای ورودی‌های مهاجرت قدیمی
و مصنوعات بایگانی/پشتیبانی استفاده می‌شود.

برخی کانال‌ها به‌صورت Plugin ارائه می‌شوند. هنگامی که یکی از آن‌ها را در زمان راه‌اندازی انتخاب کنید، راه‌اندازی
پیش از امکان پیکربندی آن، نصبش را (از npm یا یک مسیر محلی) درخواست می‌کند.

## مستندات مرتبط

- نمای کلی راه‌اندازی: [راه‌اندازی (CLI)](/fa/start/wizard)
- مرجع راه‌اندازی CLI: [مرجع راه‌اندازی CLI](/fa/start/wizard-cli-reference)
- راه‌اندازی برنامهٔ macOS: [راه‌اندازی](/fa/start/onboarding)
- مرجع پیکربندی: [پیکربندی Gateway](/fa/gateway/configuration)
- ارائه‌دهندگان: [WhatsApp](/fa/channels/whatsapp)، [Telegram](/fa/channels/telegram)، [Discord](/fa/channels/discord)، [Google Chat](/fa/channels/googlechat)، [Signal](/fa/channels/signal)، [iMessage](/fa/channels/imessage)
- Skills: [Skills](/fa/tools/skills)، [پیکربندی Skills](/fa/tools/skills-config)
