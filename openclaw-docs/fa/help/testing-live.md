---
read_when:
    - اجرای آزمون‌های دود زندهٔ ماتریس مدل / بک‌اند CLI / ACP / ارائه‌دهندهٔ رسانه
    - اشکال‌زدایی تفکیک اعتبارنامه‌های آزمون زنده
    - افزودن یک آزمون زنده جدید مختص ارائه‌دهنده
sidebarTitle: Live tests
summary: 'آزمون‌های زنده (متصل‌شونده به شبکه): ماتریس مدل، بک‌اندهای CLI، ACP، ارائه‌دهندگان رسانه، اعتبارنامه‌ها'
title: 'آزمایش: مجموعه‌های زنده'
x-i18n:
    generated_at: "2026-07-27T15:22:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8279e734e3aa09dd1fa184806c925e0404edfa9acf0f682f73a4955ed90b8b
    source_path: help/testing-live.md
    workflow: 16
---

برای شروع سریع، اجراکننده‌های QA، مجموعه‌آزمون‌های واحد/یکپارچه‌سازی و جریان‌های Docker، به
[آزمایش](/fa/help/testing) مراجعه کنید. این صفحه آزمایش‌های **زنده** (دارای ارتباط شبکه‌ای) را پوشش می‌دهد:
ماتریس مدل، بک‌اندهای CLI، ACP، ارائه‌دهندگان رسانه و مدیریت اطلاعات اعتبارسنجی.

## آزمایش‌های زنده در برابر Gateway واقعی شما

مجموعه‌آزمون‌های زنده و آزمایش‌های سریع موردی هرگز نباید Gatewayای را که از قبل
ترافیک واقعی را سرویس می‌دهد (متعلق به شما یا اپراتوری دیگر) مختل کنند:

- Gateway خود را بیاورید: از Gateway درون‌پردازشی (لایه 2 در ادامه) استفاده کنید یا یک
  نمونه توسعه را با دایرکتوری وضعیت ایزوله (`OPENCLAW_STATE_DIR=<scratch>`) و یک
  پورت آزاد راه‌اندازی کنید. هنگامی که یک Gateway واقعی روی پورت پیش‌فرض Gateway
  (18789) در حال اجرا است، به آن پورت متصل نشوید.
- سرویسی را که در این نشست راه‌اندازی نکرده‌اید، با `openclaw gateway stop`/`restart` (یا معادل‌های
  `launchctl`/`systemctl`/tmux) مدیریت نکنید — آن نمونه زنده
  اپراتور است. ابتدا تأیید صریح بگیرید.
- به داده‌های واقع‌گرایانه نیاز دارید؟ وضعیت/DB زنده را در دایرکتوری وضعیت توسعه خود کپی کنید و
  آزمایش را روی نسخه کپی‌شده انجام دهید. مهاجرت درجا روی وضعیت یک Gateway زنده نیز به
  تأیید صریح نیاز دارد.

## زنده: فرمان‌های آزمایش سریع محلی

پیش از بررسی‌های زنده موردی، کلید ارائه‌دهنده موردنیاز را در محیط پردازش
صادر کنید.

آزمایش سریع و ایمن رسانه:

```bash
pnpm openclaw infer tts convert --local --json \
  --text "آزمایش سریع زنده OpenClaw." \
  --output /tmp/openclaw-live-smoke.mp3
```

آزمایش سریع و ایمن آمادگی تماس صوتی:

```bash
pnpm openclaw voicecall setup --json
pnpm openclaw voicecall smoke --to "+15555550123"
```

`voicecall smoke` یک اجرای آزمایشی است، مگر اینکه `--yes` نیز وجود داشته باشد؛ از `--yes` فقط
زمانی استفاده کنید که قصد برقراری یک تماس واقعی را دارید. برای Twilio، Telnyx و Plivo،
بررسی موفق آمادگی به یک URL عمومی Webhook نیاز دارد؛ URLهای حلقه‌بازگشت محلی/خصوصی
رد می‌شوند، زیرا این ارائه‌دهندگان نمی‌توانند به آن‌ها دسترسی پیدا کنند.

## زنده: پیمایش قابلیت‌های Node در Android

- آزمایش: `src/gateway/android-node.capabilities.live.test.ts`
- اسکریپت: `pnpm android:test:integration`
- هدف: فراخوانی **تمام فرمان‌هایی که در حال حاضر اعلام شده‌اند** توسط یک Node متصل Android و بررسی رفتار قرارداد فرمان.
- دامنه:
  - راه‌اندازی پیش‌شرط‌دار/دستی (مجموعه‌آزمون برنامه را نصب/اجرا/جفت نمی‌کند).
  - اعتبارسنجی فرمان‌به‌فرمان `node.invoke` در Gateway برای Node انتخاب‌شده Android.
- پیش‌راه‌اندازی الزامی:
  - برنامه Android از قبل به Gateway متصل و با آن جفت شده باشد.
  - برنامه در پیش‌زمینه نگه داشته شود.
  - مجوزها/رضایت ضبط برای قابلیت‌هایی که انتظار دارید موفق شوند، اعطا شده باشد.
- بازنویسی‌های اختیاری هدف:
  - `OPENCLAW_ANDROID_NODE_ID` یا `OPENCLAW_ANDROID_NODE_NAME`.
  - `OPENCLAW_ANDROID_GATEWAY_URL` / `OPENCLAW_ANDROID_GATEWAY_TOKEN` / `OPENCLAW_ANDROID_GATEWAY_PASSWORD`.
- جزئیات کامل راه‌اندازی Android: [برنامه Android](/fa/platforms/android)

## زنده: آزمایش سریع مدل (کلیدهای نمایه)

آزمایش‌های زنده مدل به دو لایه تقسیم می‌شوند تا خرابی‌ها از هم جدا باشند:

- «مدل مستقیم» مشخص می‌کند که آیا ارائه‌دهنده/مدل با کلید داده‌شده اساساً می‌تواند پاسخ دهد.
- «آزمایش سریع Gateway» مشخص می‌کند که آیا پایپ‌لاین کامل Gateway+عامل برای آن مدل کار می‌کند (نشست‌ها، تاریخچه، ابزارها، سیاست sandbox و غیره).

فهرست‌های گزینش‌شده مدل در ادامه، در `src/agents/live-model-filter.ts` قرار دارند و
با گذر زمان تغییر می‌کنند؛ آرایه‌های آنجا را منبع حقیقت در نظر بگیرید، نه این
صفحه را.

MiniMax M3 از `minimax/MiniMax-M3` به‌عنوان ارجاع پیش‌فرض ارائه‌دهنده/مدل خود استفاده می‌کند.

### لایه 1: تکمیل مستقیم مدل (بدون Gateway)

- آزمایش: `src/agents/models.profiles.live.test.ts`
- هدف:
  - فهرست‌کردن مدل‌های کشف‌شده
  - استفاده از `getApiKeyForModel` برای انتخاب مدل‌هایی که اطلاعات اعتبارسنجی آن‌ها را دارید
  - اجرای یک تکمیل کوچک برای هر مدل (و رگرسیون‌های هدفمند در صورت نیاز)
- روش فعال‌سازی:
  - `pnpm test:live` (یا `OPENCLAW_LIVE_TEST=1` در صورت فراخوانی مستقیم Vitest)
  - برای اجرای واقعی این مجموعه‌آزمون، `OPENCLAW_LIVE_MODELS=modern`، `small` یا `all` (نام مستعار `modern`) را تنظیم کنید؛ در غیر این صورت رد می‌شود تا `pnpm test:live` به‌تنهایی بر آزمایش سریع Gateway متمرکز بماند.
- روش انتخاب مدل‌ها:
  - `OPENCLAW_LIVE_MODELS=modern` فهرست اولویت گزینش‌شده و پرسیگنال را اجرا می‌کند (به [زنده: ماتریس مدل](#live-model-matrix-what-we-cover) مراجعه کنید)
  - `OPENCLAW_LIVE_MODELS=small` فهرست اولویت گزینش‌شده مدل‌های کوچک را اجرا می‌کند
  - `OPENCLAW_LIVE_MODELS=all` نام مستعار `modern` است
  - یا `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,..."` (فهرست مجاز جداشده با ویرگول)
  - اجراهای محلی مدل کوچک Ollama به‌طور پیش‌فرض از `http://127.0.0.1:11434` استفاده می‌کنند؛ `OPENCLAW_LIVE_OLLAMA_BASE_URL` را فقط برای نقاط پایانی LAN، سفارشی یا Ollama Cloud تنظیم کنید.
  - پیمایش‌های مدرن/همه و کوچک، به‌طور پیش‌فرض طول فهرست گزینش‌شده خود را به‌عنوان سقف در نظر می‌گیرند؛ برای پیمایش جامع نمایه انتخاب‌شده، `OPENCLAW_LIVE_MAX_MODELS=0` یا برای سقفی کوچک‌تر یک عدد مثبت تنظیم کنید.
  - پیمایش‌های جامع از `OPENCLAW_LIVE_TEST_TIMEOUT_MS` برای مهلت زمانی کل آزمایش مدل مستقیم استفاده می‌کنند. پیش‌فرض: 60 دقیقه.
  - کاوش‌های مدل مستقیم به‌طور پیش‌فرض با هم‌روندی 20تایی اجرا می‌شوند؛ برای بازنویسی، `OPENCLAW_LIVE_MODEL_CONCURRENCY` را تنظیم کنید.
- روش انتخاب ارائه‌دهندگان:
  - `OPENCLAW_LIVE_PROVIDERS="google,google-antigravity,google-gemini-cli"` (فهرست مجاز جداشده با ویرگول)
- منبع کلیدها:
  - به‌طور پیش‌فرض: مخزن نمایه و جایگزین‌های محیطی
  - برای اجبار استفاده صرفاً از **مخزن نمایه**، `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` را تنظیم کنید
- دلیل وجود این قابلیت:
  - «API ارائه‌دهنده خراب است / کلید نامعتبر است» را از «پایپ‌لاین عامل Gateway خراب است» جدا می‌کند
  - شامل رگرسیون‌های کوچک و ایزوله است (مثال: بازپخش استدلال OpenAI Responses/Codex Responses و جریان‌های فراخوانی ابزار)

### لایه 2: آزمایش سریع Gateway + عامل توسعه (کاری که "@openclaw" واقعاً انجام می‌دهد)

- آزمایش: `src/gateway/gateway-models.profiles.live.test.ts`
- هدف:
  - راه‌اندازی یک Gateway درون‌پردازشی
  - ایجاد/اصلاح یک نشست `agent:dev:*` (بازنویسی مدل در هر اجرا)
  - پیمایش مدل‌های دارای کلید و بررسی موارد زیر:
    - پاسخ «معنادار» (بدون ابزار)
    - کارکرد یک فراخوانی واقعی ابزار (کاوش خواندن)
    - کاوش‌های اختیاری ابزار اضافی (کاوش اجرا+خواندن)
    - مسیرهای رگرسیون OpenAI (فقط فراخوانی ابزار -> پیگیری) همچنان کار می‌کنند
- جزئیات کاوش (تا بتوانید خرابی‌ها را سریع توضیح دهید):
  - کاوش `read`: آزمایش یک فایل nonce در فضای کاری می‌نویسد و از عامل می‌خواهد آن را `read` کند و nonce را بازگرداند.
  - کاوش `exec+read`: آزمایش از عامل می‌خواهد یک nonce را با `exec` در یک فایل موقت بنویسد، سپس آن را با `read` بازخوانی کند.
  - کاوش تصویر: آزمایش یک PNG تولیدشده (گربه + کد تصادفی) را پیوست می‌کند و انتظار دارد مدل `cat <CODE>` را برگرداند.
  - مرجع پیاده‌سازی: `src/gateway/gateway-models.profiles.live.test.ts` و `test/helpers/live-image-probe.ts`.
- روش فعال‌سازی:
  - `pnpm test:live` (یا `OPENCLAW_LIVE_TEST=1` در صورت فراخوانی مستقیم Vitest)
- روش انتخاب مدل‌ها:
  - پیش‌فرض: فهرست اولویت گزینش‌شده و پرسیگنال (`modern`)
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small` فهرست گزینش‌شده مدل‌های کوچک را از پایپ‌لاین کامل Gateway+عامل عبور می‌دهد
  - `OPENCLAW_LIVE_GATEWAY_MODELS=all` نام مستعار `modern` است
  - یا برای محدودسازی، `OPENCLAW_LIVE_GATEWAY_MODELS="provider/model"` (یا فهرست جداشده با ویرگول) را تنظیم کنید
  - پیمایش‌های مدرن/همه و کوچک Gateway به‌طور پیش‌فرض طول فهرست گزینش‌شده خود را به‌عنوان سقف در نظر می‌گیرند؛ برای پیمایش جامع انتخاب‌شده، `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0` یا برای سقفی کوچک‌تر یک عدد مثبت تنظیم کنید.
- روش انتخاب ارائه‌دهندگان (برای جلوگیری از «همه‌چیز OpenRouter»):
  - `OPENCLAW_LIVE_GATEWAY_PROVIDERS="google,google-antigravity,google-gemini-cli,openai,anthropic,zai,minimax"` (فهرست مجاز جداشده با ویرگول)
- کاوش‌های ابزار + تصویر در این آزمایش زنده همیشه فعال‌اند:
  - کاوش `read` + کاوش `exec+read` (فشار ابزار)
  - کاوش تصویر زمانی اجرا می‌شود که مدل پشتیبانی از ورودی تصویر را اعلام کند
  - جریان (در سطح کلی):
    - آزمایش یک PNG کوچک با «CAT» + کد تصادفی تولید می‌کند (`test/helpers/live-image-probe.ts`)
    - آن را از طریق `agent` `attachments: [{ mimeType: "image/png", content: "<base64>" }]` ارسال می‌کند
    - Gateway پیوست‌ها را به `images[]` تجزیه می‌کند (`src/gateway/server-methods/agent.ts` + `src/gateway/chat-attachments.ts`)
    - عامل تعبیه‌شده یک پیام چندوجهی کاربر را به مدل ارسال می‌کند
    - بررسی: پاسخ شامل `cat` + کد باشد (تلورانس OCR: خطاهای جزئی مجازند)

<Tip>
برای مشاهده موارد قابل‌آزمایش روی دستگاه خود (و شناسه‌های دقیق `provider/model`)، اجرا کنید:

```bash
openclaw models list
openclaw models list --json
```

</Tip>

## زنده: آزمایش سریع بک‌اند CLI (Claude، Gemini یا دیگر CLIهای محلی)

- آزمایش: `src/gateway/gateway-cli-backend.live.test.ts`
- هدف: اعتبارسنجی پایپ‌لاین Gateway + عامل با استفاده از یک بک‌اند CLI محلی، بدون دست‌زدن به پیکربندی پیش‌فرض شما.
- پیش‌فرض‌های آزمایش سریع مختص هر بک‌اند در تعریف `cli-backend.ts` متعلق به Plugin مالک قرار دارند.
- فعال‌سازی:
  - `pnpm test:live` (یا `OPENCLAW_LIVE_TEST=1` در صورت فراخوانی مستقیم Vitest)
  - `OPENCLAW_LIVE_CLI_BACKEND=1`
- پیش‌فرض‌ها:
  - ارائه‌دهنده/مدل پیش‌فرض: `claude-cli/claude-sonnet-4-6`
  - رفتار فرمان/آرگومان‌ها/تصویر از فراداده Plugin مالک بک‌اند CLI می‌آید.
- بازنویسی‌ها (اختیاری):
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6"`
  - `OPENCLAW_LIVE_CLI_BACKEND_COMMAND="/full/path/to/claude"`
  - `OPENCLAW_LIVE_CLI_BACKEND_ARGS='["-p","--output-format","json"]'`
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_PROBE=1` برای ارسال یک پیوست تصویری واقعی (مسیرها در اعلان تزریق می‌شوند). در دستورالعمل‌های Docker به‌طور پیش‌فرض غیرفعال است.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_ARG="--image"` برای ارسال مسیر فایل‌های تصویر به‌عنوان آرگومان‌های CLI به‌جای تزریق در اعلان.
  - `OPENCLAW_LIVE_CLI_BACKEND_IMAGE_MODE="repeat"` (یا `"list"`) برای کنترل نحوه ارسال آرگومان‌های تصویر هنگامی که `IMAGE_ARG` تنظیم شده است.
  - `OPENCLAW_LIVE_CLI_BACKEND_RESUME_PROBE=1` برای ارسال نوبت دوم و اعتبارسنجی جریان ازسرگیری.
  - `OPENCLAW_LIVE_CLI_BACKEND_MODEL_SWITCH_PROBE=1` برای فعال‌سازی اختیاری کاوش تداوم همان نشست Claude Sonnet -> Opus، هنگامی که مدل انتخاب‌شده از هدف تغییر پشتیبانی می‌کند. به‌طور پیش‌فرض، از جمله در دستورالعمل‌های Docker، غیرفعال است.
  - `OPENCLAW_LIVE_CLI_BACKEND_MCP_PROBE=1` برای فعال‌سازی اختیاری کاوش حلقه‌بازگشت MCP/ابزار. در دستورالعمل‌های Docker به‌طور پیش‌فرض غیرفعال است.

مثال:

```bash
  OPENCLAW_LIVE_CLI_BACKEND=1 \
  OPENCLAW_LIVE_CLI_BACKEND_MODEL="claude-cli/claude-sonnet-4-6" \
  pnpm test:live src/gateway/gateway-cli-backend.live.test.ts
```

آزمایش سریع کم‌هزینه پیکربندی MCP در Gemini:

```bash
OPENCLAW_LIVE_TEST=1 \
  pnpm test:live src/agents/cli-runner/bundle-mcp.gemini.live.test.ts
```

این کار از Gemini نمی‌خواهد پاسخی تولید کند. همان تنظیمات سیستمی را که
OpenClaw به Gemini می‌دهد می‌نویسد، سپس `gemini --debug mcp list` را اجرا می‌کند تا ثابت کند یک
سرور ذخیره‌شده `transport: "streamable-http"` به قالب HTTP MCP در Gemini
نرمال‌سازی می‌شود و می‌تواند به یک سرور محلی MCP مبتنی بر HTTP قابل‌جریان متصل شود.

دستورالعمل Docker:

```bash
pnpm test:docker:live-cli-backend
```

دستورالعمل‌های Docker برای یک ارائه‌دهنده:

```bash
pnpm test:docker:live-cli-backend:claude
pnpm test:docker:live-cli-backend:claude-subscription
pnpm test:docker:live-cli-backend:gemini
```

نکات:

- اجراکننده Docker در `scripts/test-live-cli-backend-docker.sh` قرار دارد.
- این اجراکننده آزمون دود زندهٔ بک‌اند CLI را درون تصویر Docker مخزن و با کاربر غیرریشهٔ `node` اجرا می‌کند.
- این اجراکننده فرادادهٔ آزمون دود CLI را از Plugin مالک آن استخراج می‌کند، سپس بستهٔ CLI متناظر Linux (`@anthropic-ai/claude-code` یا `@google/gemini-cli`) را در پیشوند نوشتنی ذخیره‌شده در حافظهٔ نهان در `OPENCLAW_DOCKER_CLI_TOOLS_DIR` (پیش‌فرض: `~/.cache/openclaw/docker-cli-tools`) نصب می‌کند.
- `codex-cli` دیگر یک بک‌اند CLI همراه نیست؛ به‌جای آن از `openai/*` با زمان‌اجرای app-server مربوط به Codex استفاده کنید (نگاه کنید به [زنده: آزمون دود هارنس app-server مربوط به Codex](#live-codex-app-server-harness-smoke)).
- `pnpm test:docker:live-cli-backend:claude-subscription` به OAuth قابل‌حمل اشتراک Claude Code از طریق `~/.claude/.credentials.json` با `claudeAiOauth.subscriptionType` یا `CLAUDE_CODE_OAUTH_TOKEN` از `claude setup-token` نیاز دارد. ابتدا `claude -p` مستقیم را در Docker اثبات می‌کند، سپس دو نوبت بک‌اند CLI در Gateway را بدون حفظ متغیرهای محیطی کلید API مربوط به Anthropic اجرا می‌کند. این مسیر اشتراک، کاوش‌های Claude MCP/ابزار و تصویر را به‌طور پیش‌فرض غیرفعال می‌کند، زیرا از محدودیت‌های استفادهٔ اشتراک واردشده مصرف می‌کند و Anthropic می‌تواند رفتار صورت‌حساب و محدودیت نرخ Claude Agent SDK / `claude -p` را بدون انتشار نسخه‌ای از OpenClaw تغییر دهد.
- Claude و Gemini از طریق پرچم‌های بالا از مجموعه‌کاوش یکسانی (نوبت متنی، دسته‌بندی تصویر، فراخوانی ابزار MCP `cron` و تداوم تعویض مدل) پشتیبانی می‌کنند، اما هیچ‌یک از این کاوش‌ها به‌طور پیش‌فرض اجرا نمی‌شوند؛ در صورت نیاز، هرکدام را با پرچم مربوط به آن فعال کنید.

## زنده: دسترسی‌پذیری پراکسی HTTP/2 مربوط به APNs

- آزمون: `src/infra/push-apns-http2.live.test.ts`
- هدف: ایجاد تونل از طریق یک پراکسی محلی HTTP CONNECT به نقطهٔ پایانی sandbox مربوط به APNs اپل، ارسال درخواست اعتبارسنجی HTTP/2 مربوط به APNs و اطمینان از اینکه پاسخ واقعی `403 InvalidProviderToken` اپل از مسیر پراکسی بازمی‌گردد.
- فعال‌سازی:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_APNS_REACHABILITY=1 pnpm test:live src/infra/push-apns-http2.live.test.ts`
- مهلت زمانی اختیاری:
  - `OPENCLAW_LIVE_APNS_TIMEOUT_MS=30000`

## زنده: آزمون دود اتصال ACP (`/acp spawn ... --bind here`)

- آزمون: `src/gateway/gateway-acp-bind.live.test.ts`
- هدف: اعتبارسنجی جریان واقعی اتصال مکالمهٔ ACP با یک عامل زندهٔ ACP:
  - ارسال `/acp spawn <agent> --bind here`
  - اتصال درجا یک مکالمهٔ مصنوعی کانال پیام
  - ارسال یک پیگیری عادی در همان مکالمه
  - تأیید اینکه پیگیری در رونوشت نشست ACP متصل ثبت می‌شود
- فعال‌سازی:
  - `pnpm test:live src/gateway/gateway-acp-bind.live.test.ts`
  - `OPENCLAW_LIVE_ACP_BIND=1`
- پیش‌فرض‌ها:
  - عامل‌های ACP در Docker: `claude,codex,gemini`
  - عامل ACP برای `pnpm test:live ...` مستقیم: `claude`
  - کانال مصنوعی: زمینهٔ مکالمه به سبک پیام مستقیم Slack
  - بک‌اند ACP: `acpx`
- بازنویسی‌ها:
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=claude`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=codex`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=droid`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT=opencode`
  - `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude,codex,gemini`
  - `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND='npx -y @agentclientprotocol/claude-agent-acp@<version>'`
  - `OPENCLAW_LIVE_ACP_BIND_CODEX_MODEL=gpt-5.6-luna`
  - `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL=opencode/kimi-k2.6`
  - `OPENCLAW_LIVE_ACP_BIND_IMAGE_PROBE=1` (یا `on`/`true`/`yes`) برای واداشتن کاوش تصویر به فعال‌شدن؛ هر مقدار دیگری آن را به اجبار غیرفعال می‌کند. به‌طور پیش‌فرض برای همهٔ عامل‌ها به‌جز `opencode` اجرا می‌شود.
  - `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1`
  - `OPENCLAW_LIVE_ACP_BIND_PARENT_MODEL=openai/gpt-5.6-luna`
- نکته‌ها:
  - این مسیر از سطح `chat.send` مربوط به Gateway با فیلدهای مصنوعی مسیر مبدأ مخصوص مدیر استفاده می‌کند تا آزمون‌ها بتوانند زمینهٔ کانال پیام را بدون تظاهر به تحویل خارجی پیوست کنند.
  - وقتی `OPENCLAW_LIVE_ACP_BIND_AGENT_COMMAND` تنظیم نشده باشد، آزمون برای عامل هارنس ACP انتخاب‌شده از رجیستری عامل داخلی Plugin تعبیه‌شدهٔ `acpx` استفاده می‌کند.
  - ایجاد Cron مبتنی بر MCP برای نشست متصل، به‌طور پیش‌فرض در حد بهترین تلاش است، زیرا هارنس‌های خارجی ACP ممکن است پس از موفقیت اثبات اتصال/تصویر فراخوانی‌های MCP را لغو کنند؛ برای سخت‌گیرانه‌کردن کاوش Cron پس از اتصال، `OPENCLAW_LIVE_ACP_BIND_REQUIRE_CRON=1` را تنظیم کنید.

مثال:

```bash
OPENCLAW_LIVE_ACP_BIND=1 \
  OPENCLAW_LIVE_ACP_BIND_AGENT=claude \
  pnpm test:live src/gateway/gateway-acp-bind.live.test.ts
```

دستورالعمل Docker:

```bash
pnpm test:docker:live-acp-bind
```

دستورالعمل‌های Docker برای یک عامل:

```bash
pnpm test:docker:live-acp-bind:claude
pnpm test:docker:live-acp-bind:codex
pnpm test:docker:live-acp-bind:droid
pnpm test:docker:live-acp-bind:gemini
pnpm test:docker:live-acp-bind:opencode
```

نکته‌های Docker:

- اجراکننده Docker در `scripts/test-live-acp-bind-docker.sh` قرار دارد.
- به‌طور پیش‌فرض، آزمون دود اتصال ACP را به‌ترتیب در برابر عامل‌های زندهٔ تجمیعی CLI اجرا می‌کند: `claude`، سپس `codex` و پس از آن `gemini`.
- برای محدودکردن ماتریس از `OPENCLAW_LIVE_ACP_BIND_AGENTS=claude`، `OPENCLAW_LIVE_ACP_BIND_AGENTS=codex`، `OPENCLAW_LIVE_ACP_BIND_AGENTS=droid`، `OPENCLAW_LIVE_ACP_BIND_AGENTS=gemini` یا `OPENCLAW_LIVE_ACP_BIND_AGENTS=opencode` استفاده کنید.
- ابتدا داده‌های احراز هویت CLI متناظر را در کانتینر آماده می‌کند، سپس اگر CLI زندهٔ درخواستی (`@anthropic-ai/claude-code`، `@openai/codex`، Factory Droid از طریق `https://app.factory.ai/cli`، `@google/gemini-cli` یا `opencode-ai`) موجود نباشد، آن را نصب می‌کند. خود بک‌اند ACP، بستهٔ تعبیه‌شدهٔ `acpx/runtime` از Plugin رسمی `acpx` است.
- گونهٔ Docker مربوط به Droid، برای تنظیمات `~/.factory` را آماده می‌کند، `FACTORY_API_KEY` را به کانتینر ارسال می‌کند و به آن کلید API نیاز دارد، زیرا احراز هویت محلی OAuth/جاکلیدی Factory به کانتینر قابل‌انتقال نیست. این گونه از ورودی داخلی `droid exec --output-format acp` در رجیستری ACPX استفاده می‌کند.
- گونهٔ Docker مربوط به OpenCode یک مسیر سخت‌گیرانهٔ رگرسیون تک‌عاملی است. این گونه یک مدل پیش‌فرض موقت `OPENCODE_CONFIG_CONTENT` را از `OPENCLAW_LIVE_ACP_BIND_OPENCODE_MODEL` (پیش‌فرض `opencode/kimi-k2.6`) می‌نویسد.
- فراخوانی‌های مستقیم CLI مربوط به `acpx` فقط مسیری دستی/راه‌حل موقت برای مقایسهٔ رفتار خارج از Gateway هستند. آزمون دود اتصال ACP در Docker، بک‌اند زمان‌اجرای تعبیه‌شدهٔ `acpx` متعلق به OpenClaw را آزمایش می‌کند.

## زنده: آزمون دود هارنس app-server مربوط به Codex

- هدف: اعتبارسنجی هارنس Codex تحت مالکیت Plugin از طریق متد عادی
  `agent` در Gateway:
  - بارگذاری Plugin همراه `codex`
  - انتخاب یک مدل OpenAI از طریق `/model <ref> --runtime codex`
  - ارسال نخستین نوبت عامل Gateway با سطح تفکر درخواستی
  - ارسال نوبت دوم به همان نشست OpenClaw و تأیید اینکه رشتهٔ
    app-server می‌تواند از سر گرفته شود
  - اجرای `/codex status` و `/codex models` از طریق همان مسیر فرمان
    Gateway
  - اجرای اختیاری دو کاوش پوستهٔ ارتقایافته که Guardian آن‌ها را بازبینی می‌کند: یک
    فرمان بی‌خطر که باید تأیید شود و یک بارگذاری راز جعلی که باید
    رد شود تا عامل دوباره درخواست کند
- آزمون: `src/gateway/gateway-codex-harness.live.test.ts`
- فعال‌سازی: `OPENCLAW_LIVE_CODEX_HARNESS=1`
- مدل مبنای هارنس: `openai/gpt-5.6-luna`
- پیش‌فرض انتخاب کلید API تازهٔ OpenAI: `openai/gpt-5.6`
- تفکر پیش‌فرض: `low`
- بازنویسی مدل: `OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/<model>`
- بازنویسی تفکر: `OPENCLAW_LIVE_CODEX_HARNESS_THINKING=<level>`
- ادعای تلاش برای مدل غیرپیش‌فرض:
  `OPENCLAW_LIVE_CODEX_HARNESS_EXPECTED_EFFORT=<level>`
- بازنویسی ماتریس: `OPENCLAW_LIVE_CODEX_HARNESS_TARGETS=<model>=<thinking>,...`
- حالت احراز هویت: `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=codex-auth` (پیش‌فرض) از ورود کپی‌شدهٔ Codex استفاده می‌کند؛ `api-key` از `OPENAI_API_KEY` از طریق app-server مربوط به Codex استفاده می‌کند.
- کاوش اختیاری تصویر: `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1`
- کاوش اختیاری MCP/ابزار: `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1`
- کاوش اختیاری Guardian: `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1`
- تنش اختیاری ازسرگیری: `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1` چهار
  نوبت تاریخچه اضافه می‌کند، سپس Gateway و app-server مربوط به Codex را
  سه بار می‌بندد و دوباره راه‌اندازی می‌کند، درحالی‌که همان شناسهٔ رشتهٔ بومی و تاریخچهٔ
  مکالمه را الزامی می‌داند. شمارهای محدودشده را با
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_HISTORY_TURNS` (1-20) و
  `OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS_RESTARTS` (1-10) بازنویسی کنید.
- تنش اختیاری گسترش موازی: `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1`
  و `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT` (1-12) را تنظیم کنید. هارنس
  همهٔ فرزندها را هم‌زمان آغاز می‌کند، منتظر اجرای پایانی همه می‌ماند و هر
  پاسخ یکتای فرزند و هویت رشتهٔ بومی را تأیید می‌کند.
- تنش اختیاری Compaction: `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1`
  خروجی محدودشدهٔ ابزار بومی را تولید می‌کند، رخدادهای Compaction خودکار را الزامی می‌داند،
  شمار Compaction ذخیره‌شده و یادآوری نشانگر پنهان را تأیید می‌کند، Gateway
  و app-server فیزیکی Codex را دوباره راه‌اندازی می‌کند، سپس موج خروجی و
  Compaction را تکرار می‌کند. حجم کار محدودشده را با
  `OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS` (1-8) و
  `OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES` (100000-800000) تنظیم کنید.
- زمینهٔ کامل API مستقیم: `OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1`
  زمینهٔ `922000` و محدودیت‌های کل Compaction به میزان `700000` را اعمال می‌کند، نوبت‌های متراکم و محدودشدهٔ
  کاربر را می‌فرستد، در هر موج دو نقطهٔ بازرسی صریح Compaction بومی اجرا می‌کند و
  پس از هر نقطهٔ بازرسی با نوبت‌های بعدی ادامه می‌دهد. این حالت به
  `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key` به‌علاوهٔ یک مسیر مطلق
  `OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG` نیاز دارد. کاتالوگ باید مدل
  انتخاب‌شده را با `max_context_window: 922000` ارائه کند تا Codex بازنویسی را
  دوباره به پنجرهٔ عادی کاتالوگ خود محدود نکند. تنش عادی با آستانهٔ کاهش‌یافتهٔ
  بالا، ادعاهای سخت‌گیرانه‌تر Compaction خودکار و حفظ نشانگر پنهان
  را نگه می‌دارد.
- کاوش اختیاری انصراف از رلهٔ حلقه:
  `OPENCLAW_LIVE_CODEX_HARNESS_DISABLE_LOOP_RELAY=1`
- ترجیح تفکر درخواستی ممکن است به نزدیک‌ترین سطح تلاش اعلام‌شده
  توسط Codex برای آن مدل نگاشت شود. برای مثال، Luna مقدار `minimal` را به `low` نگاشت می‌کند.
- مدل‌های شناخته‌شدهٔ کاتالوگ Codex، همان تلاش بومی دقیق را به‌طور خودکار استخراج می‌کنند.
  بازنویسی مدل‌های ناشناخته باید تلاش نگاشت‌شدهٔ مورد انتظار را مشخص کند.
- آزمون دود، ارائه‌دهنده/مدل را به `agentRuntime.id: "codex"` وادار می‌کند تا هارنس خراب Codex
  نتواند با بازگشت بی‌سروصدا به OpenClaw موفق شود.
- احراز هویت: احراز هویت app-server مربوط به Codex از ورود محلی اشتراک Codex، یا
  `OPENAI_API_KEY` هنگامی که `OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key`. Docker می‌تواند
  `~/.codex/auth.json` و `~/.codex/config.toml` را برای اجراهای اشتراکی کپی کند.

دستورالعمل محلی:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-luna \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

دستورالعمل Docker:

```bash
pnpm test:docker:live-codex-harness
```

تنش راه‌اندازی مجدد و تاریخچه:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
pnpm test:docker:live-codex-harness
```

تنش گسترش موازی، خروجی بزرگ، Compaction و راه‌اندازی مجدد:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_COUNT=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_RESUME_STRESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS=1 \
  pnpm test:docker:live-codex-harness
```

تنش Compaction برای بودجهٔ ورودی بومی `922000` در Codex:

```bash
OPENCLAW_LIVE_CODEX_HARNESS=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_FULL_CONTEXT=1 \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL_CATALOG=/absolute/path/to/models-api-1m.json \
  OPENCLAW_LIVE_CODEX_HARNESS_MODEL=openai/gpt-5.6-terra \
  OPENCLAW_LIVE_CODEX_HARNESS_THINKING=medium \
  OPENCLAW_LIVE_CODEX_HARNESS_COMPACTION_STRESS_TURNS=8 \
  OPENCLAW_LIVE_CODEX_HARNESS_LARGE_OUTPUT_BYTES=800000 \
  pnpm test:live -- src/gateway/gateway-codex-harness.live.test.ts
```

ماتریس بومی Codex برای GPT-5.6:

```bash
OPENCLAW_LIVE_CODEX_HARNESS_AUTH=api-key \
  OPENCLAW_LIVE_CODEX_HARNESS_TARGETS='openai/gpt-5.6-sol=ultra,openai/gpt-5.6-terra=ultra,openai/gpt-5.6-luna=max' \
  pnpm test:docker:live-codex-harness
```

## زنده: Compaction تکرارشوندهٔ OpenAI

- هدف: اجرای حلقه عامل تعبیه‌شده OpenClaw `openai-responses` از طریق
  حداقل دو Compaction خودکار واقعی، سپس بررسی ماندگاری یک نشانگر پایدار.
- آزمایش: `src/agents/sessions/agent-session.openai-compaction.live.test.ts`
- فعال‌سازی: `OPENCLAW_LIVE_OPENAI_COMPACTION=1`
- مدل پیش‌فرض: `gpt-5.6-luna`
- بازنویسی مدل: `OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=<model>`
- حالت عادی فشار از بودجه زمینه کاهش‌یافته سمت کارخواه استفاده می‌کند تا با هزینه
  محدود API به همان مسیر Compaction واقعی برسد.
- حالت زمینه کامل، بودجه کارخواه را روی `922000` و ذخیره Compaction را روی
  `222000` تنظیم می‌کند، بنابراین Compaction خودکار از `700000` آغاز می‌شود. همچنین به
  تعداد ورودی مشاهده‌شده ارائه‌دهنده‌ای بالاتر از مرز قیمت‌گذاری زمینه بلند `272000` نیاز دارد.

دستورالعمل زنده محدود:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

دستورالعمل بودجه ورودی کامل `922000`:

```bash
OPENCLAW_LIVE_TEST=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_FULL=1 \
  OPENCLAW_LIVE_OPENAI_COMPACTION_MODEL=gpt-5.6-terra \
  pnpm test:live -- src/agents/sessions/agent-session.openai-compaction.live.test.ts
```

<Warning>
حالت کامل عمداً از مرز قیمت‌گذاری زمینه بلند OpenAI عبور می‌کند و
ممکن است چندین فراخوانی بزرگ API انجام دهد. فقط با تأیید صریح هزینه از آن استفاده کنید.
</Warning>

پیش‌فرض کلید API تازه OpenAI:

```bash
OPENCLAW_LIVE_GATEWAY_OPENAI_API_DEFAULT=1 \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_THINKING=off \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

این اثبات، `OPENCLAW_LIVE_GATEWAY_MODELS` را تنظیم‌نشده باقی می‌گذارد، مدل را از طریق
درز انتخاب استنتاج در راه‌اندازی اولیه تازه حل می‌کند، `openai/gpt-5.6` را بررسی می‌کند و سپس
یک نوبت واقعی Gateway را با مدل حل‌شده اجرا می‌کند.

ماتریس OpenClaw تعبیه‌شده GPT-5.6:

```bash
OPENCLAW_LIVE_GATEWAY_THINKING=ultra \
  OPENCLAW_LIVE_GATEWAY_PROVIDERS=openai \
  OPENCLAW_LIVE_GATEWAY_MODELS='openai/gpt-5.6-sol,openai/gpt-5.6-terra,openai/gpt-5.6-luna' \
  pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
```

نکات Docker:

- اجراکننده Docker در `scripts/test-live-codex-harness-docker.sh` قرار دارد.
- این اجراکننده `OPENAI_API_KEY` را عبور می‌دهد، فایل‌های احراز هویت Codex CLI را در صورت وجود کپی می‌کند،
  `@openai/codex` را در یک پیشوند npm سوارشده و قابل‌نوشتن
  نصب می‌کند، درخت منبع را آماده می‌کند و سپس فقط آزمایش زنده مهار Codex را اجرا می‌کند.
- Docker به‌طور پیش‌فرض کاوش‌های تصویر، MCP/ابزار و Guardian را فعال می‌کند. در صورت نیاز به اجرای
  اشکال‌زدایی محدودتر، `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0` یا
  `OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0` یا
  `OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0` را تنظیم کنید.
- Docker از همان پیکربندی صریح زمان‌اجرای Codex استفاده می‌کند، بنابراین نام‌های مستعار قدیمی یا
  حالت جایگزین OpenClaw نمی‌توانند پس‌رفت مهار Codex را پنهان کنند.
- اهداف ماتریس به‌صورت ترتیبی در یک کانتینر اجرا می‌شوند. اسکریپت Docker مهلت زمانی
  پیش‌فرض 35 دقیقه‌ای خود را متناسب با تعداد اهداف افزایش می‌دهد؛ هر مهلت زمانی پوسته بیرونی یا CI باید
  همان مدت کل را مجاز بداند. CI مرجع هر هدف GPT-5.6 را در یک شارد جداگانه نگه می‌دارد.

### دستورالعمل‌های زنده پیشنهادی

فهرست‌های مجاز محدود و صریح سریع‌تر و کم‌نوسان‌تر هستند:

- یک مدل، مستقیم (بدون Gateway):
  - `OPENCLAW_LIVE_MODELS="openai/gpt-5.6-luna" pnpm test:live src/agents/models.profiles.live.test.ts`

- نمایه مستقیم مدل کوچک:
  - `OPENCLAW_LIVE_MODELS=small pnpm test:live src/agents/models.profiles.live.test.ts`

- نمایه Gateway مدل کوچک:
  - `OPENCLAW_LIVE_GATEWAY_MODELS=small pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- آزمایش دود API ‏Ollama Cloud:
  - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 pnpm test:live -- extensions/ollama/ollama.live.test.ts`

- یک مدل، آزمایش دود Gateway:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- فراخوانی ابزار در چند ارائه‌دهنده:
  - `OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.5-flash,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- آزمایش دود مستقیم Z.AI Coding Plan GLM-5.2:
  - `ZAI_CODING_LIVE_TEST=1 pnpm test:live src/agents/zai.live.test.ts`

- تمرکز Google (کلید API ‏Gemini و Antigravity):
  - Gemini (کلید API): `OPENCLAW_LIVE_GATEWAY_MODELS="google/gemini-3.5-flash" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`
  - Antigravity (OAuth): `OPENCLAW_LIVE_GATEWAY_MODELS="google-antigravity/claude-opus-4-6-thinking,google-antigravity/gemini-3-pro-high" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts`

- آزمایش دود تفکر تطبیقی Google ‏(`qa manual` از CLI خصوصی QA — نیازمند `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` و یک وارسی منبع؛ [نمای کلی QA](/fa/concepts/qa-e2e-automation) را ببینید):
  - پیش‌فرض پویا Gemini 3: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-3.1-pro-preview --alt-model google/gemini-3.1-pro-preview --message '/think adaptive Reply exactly: GEMINI_ADAPTIVE_OK' --timeout-ms 180000`
  - بودجه پویای Gemini 2.5: `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1 pnpm openclaw qa manual --provider-mode live-frontier --model google/gemini-2.5-flash --alt-model google/gemini-2.5-flash --message '/think adaptive Reply exactly: GEMINI25_ADAPTIVE_OK' --timeout-ms 180000`

نکات:

- `google/...` از API ‏Gemini (کلید API) استفاده می‌کند.
- `google-antigravity/...` از پل OAuth ‏Antigravity (نقطه پایانی عامل به سبک Cloud Code Assist) استفاده می‌کند.
- `google-gemini-cli/...` از Gemini CLI محلی روی دستگاه استفاده می‌کند (احراز هویت جداگانه و ویژگی‌های خاص ابزار).
- API ‏Gemini در برابر Gemini CLI:
  - API: ‏OpenClaw، ‏API میزبانی‌شده Gemini متعلق به Google را از طریق HTTP فراخوانی می‌کند (کلید API / احراز هویت نمایه)؛ منظور بیشتر کاربران از «Gemini» همین است.
  - CLI: ‏OpenClaw یک باینری محلی `gemini` را از طریق پوسته اجرا می‌کند؛ این باینری احراز هویت مستقل دارد و ممکن است رفتار متفاوتی داشته باشد (پخش جریانی/پشتیبانی ابزار/ناهمخوانی نسخه).

## زنده: ماتریس مدل (موارد تحت پوشش)

اجرای زنده اختیاری است، بنابراین «فهرست مدل CI» ثابتی وجود ندارد. `OPENCLAW_LIVE_MODELS=modern` / `OPENCLAW_LIVE_GATEWAY_MODELS=modern` (و نام مستعار `all` آن‌ها) فهرست اولویت گلچین‌شده را از `HIGH_SIGNAL_LIVE_MODEL_PRIORITY` در `src/agents/live-model-filter.ts`، با ترتیب اولویت زیر اجرا می‌کنند:

| ارائه‌دهنده/مدل                                | نکات      |
| --------------------------------------------- | ---------- |
| `anthropic/claude-opus-5`                     |            |
| `anthropic/claude-opus-4-8`                   |            |
| `anthropic/claude-sonnet-5`                   |            |
| `anthropic/claude-sonnet-4-6`                 |            |
| `anthropic/claude-opus-4-7`                   |            |
| `google/gemini-3.1-pro-preview`               | API ‏Gemini |
| `google/gemini-3.5-flash`                     | API ‏Gemini |
| `cohere/command-a-plus-05-2026`               |            |
| `moonshot/kimi-k3`                            |            |
| `anthropic/claude-opus-4-6`                   |            |
| `deepseek/deepseek-v4-flash`                  |            |
| `deepseek/deepseek-v4-pro`                    |            |
| `minimax/MiniMax-M3`                          |            |
| `openai/gpt-5.5`                              |            |
| `openrouter/openai/gpt-5.2-chat`              |            |
| `openrouter/minimax/minimax-m2.7`             |            |
| `opencode-go/glm-5`                           |            |
| `openrouter/ai21/jamba-large-1.7`             |            |
| `xai/grok-4.5`                                |            |
| `xai/grok-4.20-0309-reasoning`                |            |
| `zai/glm-5.1`                                 |            |
| `fireworks/accounts/fireworks/models/glm-5p1` |            |
| `minimax-portal/minimax-m3`                   |            |

فهرست گلچین‌شده **مدل‌های کوچک** (`OPENCLAW_LIVE_MODELS=small` / `OPENCLAW_LIVE_GATEWAY_MODELS=small`)، از `SMALL_LIVE_MODEL_PRIORITY`:

| ارائه‌دهنده/مدل               |
| ---------------------------- |
| `lmstudio/qwen/qwen3.5-9b`   |
| `vllm/qwen/qwen3-8b`         |
| `sglang/qwen/qwen3-8b`       |
| `ollama/gemma3:4b`           |
| `openrouter/qwen/qwen3.5-9b` |
| `openrouter/z-ai/glm-5.1`    |
| `openrouter/z-ai/glm-5`      |
| `zai/glm-5.1`                |

نکات مربوط به فهرست مدرن:

- ارائه‌دهندگان `codex` و `codex-cli` از پیمایش مدرن پیش‌فرض مستثنا هستند (آن‌ها رفتار بک‌اند CLI/ACP را پوشش می‌دهند که جداگانه در بالا آزمایش شده است). خود `openai/gpt-5.5` به‌طور پیش‌فرض از طریق مهار کارساز برنامه Codex مسیریابی می‌شود؛ [زنده: آزمایش دود مهار کارساز برنامه Codex](#live-codex-app-server-harness-smoke) را ببینید.
- `fireworks`، `google`، `openrouter` و `xai` در پیمایش مدرن فقط شناسه‌های مدل صراحتاً گلچین‌شده خود را اجرا می‌کنند (بدون گسترش خودکار «همه مدل‌های این ارائه‌دهنده»).
- حداقل یک مدل دارای قابلیت تصویر (انواع بینایی خانواده Claude/Gemini/OpenAI و غیره) را در `OPENCLAW_LIVE_GATEWAY_MODELS` بگنجانید تا کاوش تصویر اجرا شود.

آزمایش دود Gateway را همراه با ابزارها و تصویر روی مجموعه‌ای دست‌چین‌شده از چند ارائه‌دهنده اجرا کنید:

```bash
OPENCLAW_LIVE_GATEWAY_MODELS="openai/gpt-5.6-luna,anthropic/claude-opus-4-6,google/gemini-3.1-pro-preview,google/gemini-3.5-flash,google-antigravity/claude-opus-4-6-thinking,deepseek/deepseek-v4-flash,zai/glm-5.1,minimax/MiniMax-M3" pnpm test:live src/gateway/gateway-models.profiles.live.test.ts
```

پوشش اضافی اختیاری خارج از فهرست‌های گلچین‌شده (مطلوب است؛ مدلی با قابلیت «ابزارها» که فعال کرده‌اید انتخاب کنید):

- Mistral: ‏`mistral/...`
- Cerebras: ‏`cerebras/...` (اگر دسترسی دارید)
- LM Studio: ‏`lmstudio/...` (محلی؛ فراخوانی ابزار به حالت API بستگی دارد)

### تجمیع‌کننده‌ها / Gatewayهای جایگزین

اگر کلیدها را فعال کرده‌اید، می‌توانید از طریق موارد زیر نیز آزمایش کنید:

- OpenRouter: ‏`openrouter/...` (صدها مدل؛ برای یافتن گزینه‌های دارای قابلیت ابزار و تصویر از `openclaw models scan` استفاده کنید)
- OpenCode: ‏`opencode/...` برای Zen و `opencode-go/...` برای Go (احراز هویت از طریق `OPENCODE_API_KEY` / `OPENCODE_ZEN_API_KEY`)

ارائه‌دهندگان بیشتری که می‌توانید در ماتریس زنده بگنجانید (اگر اطلاعات اعتبار/پیکربندی آن‌ها را دارید):

- داخلی: `anthropic`، `cerebras`، `github-copilot`، `google`، `google-antigravity`، `google-gemini-cli`، `google-vertex`، `groq`، `mistral`، `openai`، `openrouter`، `opencode`، `opencode-go`، `xai`، `zai`
- از طریق `models.providers` (نقاط پایانی سفارشی): `minimax` (ابر/API)، به‌علاوه هر پراکسی سازگار با OpenAI/Anthropic ‏(LM Studio، ‏vLLM، ‏LiteLLM و غیره)

<Tip>
«همه مدل‌ها» را در مستندات به‌صورت ثابت ننویسید. فهرست معتبر، همان چیزی است که `discoverModels(...)` روی دستگاه برمی‌گرداند، به‌علاوه هر کلیدی که موجود است.
</Tip>

## اطلاعات اعتبار (هرگز ثبت نکنید)

آزمایش‌های زنده اطلاعات اعتبار را همانند CLI کشف می‌کنند. پیامدهای عملی:

- اگر CLI کار می‌کند، آزمایش‌های زنده نیز باید همان کلیدها را پیدا کنند.
- اگر آزمایش زنده پیام «اطلاعات اعتبار موجود نیست» می‌دهد، آن را همان‌گونه اشکال‌زدایی کنید که `openclaw models list` / انتخاب مدل را اشکال‌زدایی می‌کنید.

- نمایه‌های احراز هویت هر عامل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` (منظور از «کلیدهای نمایه» در آزمایش‌های زنده همین است)
- پیکربندی: `~/.openclaw/openclaw.json` (یا `OPENCLAW_CONFIG_PATH`)
- پوشه قدیمی OAuth: ‏`~/.openclaw/credentials/` (در صورت وجود، در خانه زنده آماده‌شده کپی می‌شود، اما مخزن اصلی کلید نمایه نیست)
- اجراهای زنده محلی، پیکربندی فعال (با حذف بازنویسی‌های `agents.*.workspace` / `agentDir`) و `auth-profiles.json` هر عامل را کپی می‌کنند — نه بقیه پوشه آن عامل؛ بنابراین داده‌های `workspace/` و `sandboxes/` هرگز به خانه آماده‌شده نمی‌رسند — و همچنین پوشه قدیمی `credentials/` و فایل‌ها/پوشه‌های احراز هویت CLI خارجی پشتیبانی‌شده (`.claude.json`، `.claude/.credentials.json`، `.claude/settings*.json`، `.claude/backups`، `.codex/auth.json`، `.codex/config.toml`، `.gemini`، `.minimax`) را در یک خانه آزمایشی موقت کپی می‌کنند.

اگر می‌خواهید به کلیدهای محیطی متکی باشید، آن‌ها را پیش از آزمایش‌های محلی صادر کنید یا از
اجراکننده‌های Docker زیر با یک `OPENCLAW_PROFILE_FILE` صریح استفاده کنید.

## اجرای زنده Deepgram (رونویسی صوت)

- آزمایش: `extensions/deepgram/audio.live.test.ts`
- فعال‌سازی: `DEEPGRAM_API_KEY=... DEEPGRAM_LIVE_TEST=1 pnpm test:live extensions/deepgram/audio.live.test.ts`

## اجرای زنده طرح کدنویسی BytePlus

- آزمایش: `extensions/byteplus/live.test.ts`
- فعال‌سازی: `BYTEPLUS_API_KEY=... BYTEPLUS_LIVE_TEST=1 pnpm test:live extensions/byteplus/live.test.ts`
- بازنویسی اختیاری مدل: `BYTEPLUS_CODING_MODEL=ark-code-latest`

## اجرای زنده رسانه گردش‌کار ComfyUI

- آزمایش: `extensions/comfy/comfy.live.test.ts`
- فعال‌سازی: `OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts`
- دامنه:
  - مسیرهای داخلی تصویر و ویدئوی comfy و `music_generate` را اجرا می‌کند
  - هر قابلیت را مگر آنکه `plugins.entries.comfy.config.<capability>` پیکربندی شده باشد، رد می‌کند
  - پس از تغییر ارسال گردش‌کار comfy، نظرسنجی، بارگیری‌ها یا ثبت Plugin مفید است

## اجرای زنده تولید تصویر

- آزمون: `test/image-generation.runtime.live.test.ts`
- دستور: `pnpm test:live test/image-generation.runtime.live.test.ts`
- هارنس: `pnpm test:live:media image`
- دامنه:
  - همه Pluginهای ثبت‌شدهٔ ارائه‌دهندهٔ تولید تصویر را فهرست می‌کند
  - پیش از کاوش، از متغیرهای محیطی ازپیش صادرشدهٔ ارائه‌دهنده استفاده می‌کند
  - به‌طور پیش‌فرض کلیدهای API زنده/محیطی را بر پروفایل‌های احراز هویت ذخیره‌شده مقدم می‌داند تا کلیدهای آزمون قدیمی در `auth-profiles.json` اعتبارنامه‌های واقعی پوسته را پنهان نکنند
  - ارائه‌دهندگان فاقد احراز هویت/پروفایل/مدل قابل‌استفاده را رد می‌کند
  - هر ارائه‌دهندهٔ پیکربندی‌شده را از مسیر زمان‌اجرای مشترک تولید تصویر اجرا می‌کند:
    - `<provider>:generate`
    - `<provider>:edit` هنگامی که ارائه‌دهنده پشتیبانی از ویرایش را اعلام می‌کند
- ارائه‌دهندگان همراه فعلی تحت پوشش:
  - `deepinfra`
  - `fal`
  - `google`
  - `minimax`
  - `openai`
  - `openrouter`
  - `vydra`
  - `xai`
- محدودسازی اختیاری:
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="openai,google,openrouter,xai"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS="deepinfra"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_MODELS="openai/gpt-image-2,google/gemini-3.1-flash-image,openrouter/google/gemini-3.1-flash-image-preview,xai/grok-imagine-image"`
  - `OPENCLAW_LIVE_IMAGE_GENERATION_CASES="google:flash-generate,google:pro-edit,openrouter:generate,xai:default-generate,xai:default-edit"`
- رفتار اختیاری احراز هویت:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` برای اجبار احراز هویت از مخزن پروفایل و نادیده‌گرفتن بازنویسی‌های صرفاً محیطی

برای مسیر CLI عرضه‌شده، پس از موفقیت آزمون زندهٔ ارائه‌دهنده/زمان‌اجرا، یک آزمون دود `infer` اضافه کنید:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_INFER_CLI_TEST=1 pnpm test:live -- test/image-generation.infer-cli.live.test.ts
openclaw infer image providers --json
openclaw infer image generate \
  --model google/gemini-3.1-flash-image \
  --prompt "تصویر آزمون تخت و مینیمال: یک مربع آبی روی پس‌زمینهٔ سفید، بدون متن." \
  --output ./openclaw-infer-image-smoke.png \
  --json
```

این کار تجزیهٔ آرگومان‌های CLI، تفکیک پیکربندی/عامل پیش‌فرض، فعال‌سازی
Plugin همراه، زمان‌اجرای مشترک تولید تصویر و درخواست زندهٔ ارائه‌دهنده را پوشش
می‌دهد. انتظار می‌رود وابستگی‌های Plugin پیش از بارگذاری زمان‌اجرا موجود باشند.

## تولید زندهٔ موسیقی

- آزمون: `extensions/music-generation-providers.live.test.ts`
- فعال‌سازی: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts`
- هارنس: `pnpm test:live:media music`
- دامنه:
  - مسیر مشترک ارائه‌دهندهٔ همراه تولید موسیقی را می‌آزماید
  - در حال حاضر `fal`،‏ `google`،‏ `minimax` و `openrouter` را پوشش می‌دهد
  - پیش از کاوش، از متغیرهای محیطی ازپیش صادرشدهٔ ارائه‌دهنده استفاده می‌کند
  - به‌طور پیش‌فرض کلیدهای API زنده/محیطی را بر پروفایل‌های احراز هویت ذخیره‌شده مقدم می‌داند تا کلیدهای آزمون قدیمی در `auth-profiles.json` اعتبارنامه‌های واقعی پوسته را پنهان نکنند
  - ارائه‌دهندگان فاقد احراز هویت/پروفایل/مدل قابل‌استفاده را رد می‌کند
  - در صورت دسترس‌بودن، هر دو حالت زمان‌اجرای اعلام‌شده را اجرا می‌کند:
    - `generate` با ورودی صرفاً پرامپت
    - `edit` هنگامی که ارائه‌دهنده `capabilities.edit.enabled` را اعلام می‌کند
  - `comfy` فایل زندهٔ جداگانهٔ خودش را دارد و بخشی از این پیمایش مشترک نیست
- محدودسازی اختیاری:
  - `OPENCLAW_LIVE_MUSIC_GENERATION_PROVIDERS="google,minimax"`
  - `OPENCLAW_LIVE_MUSIC_GENERATION_MODELS="google/lyria-3-clip-preview,minimax/music-2.6"`
- رفتار اختیاری احراز هویت:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` برای اجبار احراز هویت از مخزن پروفایل و نادیده‌گرفتن بازنویسی‌های صرفاً محیطی

## تولید زندهٔ ویدئو

- آزمون: `extensions/video-generation-providers.live.test.ts`
- فعال‌سازی: `OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/video-generation-providers.live.test.ts`
- هارنس: `pnpm test:live:media video`
- دامنه:
  - مسیر مشترک ارائه‌دهندهٔ همراه تولید ویدئو را در `alibaba`،‏ `byteplus`،‏ `deepinfra`،‏ `fal`،‏ `google`،‏ `minimax`،‏ `openai`،‏ `openrouter`،‏ `pixverse`،‏ `qwen`،‏ `runway`،‏ `together`،‏ `vydra` و `xai` می‌آزماید
  - به‌طور پیش‌فرض از مسیر آزمون دود امن برای انتشار استفاده می‌کند: یک درخواست متن‌به‌ویدئو برای هر ارائه‌دهنده، پرامپت یک‌ثانیه‌ای خرچنگ و سقف عملیات برای هر ارائه‌دهنده از `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS` (به‌طور پیش‌فرض `180000`)
  - به‌طور پیش‌فرض FAL را رد می‌کند، زیرا تأخیر صف سمت ارائه‌دهنده می‌تواند بر زمان انتشار غالب شود؛ برای اجرای صریح آن، `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="fal"` را ارسال کنید (یا فهرست ردشدن را پاک کنید)
  - پیش از کاوش، از متغیرهای محیطی ازپیش صادرشدهٔ ارائه‌دهنده استفاده می‌کند
  - به‌طور پیش‌فرض کلیدهای API زنده/محیطی را بر پروفایل‌های احراز هویت ذخیره‌شده مقدم می‌داند تا کلیدهای آزمون قدیمی در `auth-profiles.json` اعتبارنامه‌های واقعی پوسته را پنهان نکنند
  - ارائه‌دهندگان فاقد احراز هویت/پروفایل/مدل قابل‌استفاده را رد می‌کند
  - به‌طور پیش‌فرض فقط `generate` را اجرا می‌کند
  - برای اجرای حالت‌های تبدیل اعلام‌شده در صورت دسترس‌بودن نیز، `OPENCLAW_LIVE_VIDEO_GENERATION_FULL_MODES=1` را تنظیم کنید:
    - `imageToVideo` هنگامی که ارائه‌دهنده `capabilities.imageToVideo.enabled` را اعلام می‌کند و ارائه‌دهنده/مدل انتخاب‌شده ورودی تصویر محلی مبتنی بر بافر را در پیمایش مشترک می‌پذیرد
    - `videoToVideo` هنگامی که ارائه‌دهنده `capabilities.videoToVideo.enabled` را اعلام می‌کند و ارائه‌دهنده/مدل انتخاب‌شده ورودی ویدئوی محلی مبتنی بر بافر را در پیمایش مشترک می‌پذیرد
  - ارائه‌دهندهٔ فعلی `imageToVideo` که اعلام شده اما در پیمایش مشترک رد می‌شود:
    - `vydra` (ورودی تصویر محلی مبتنی بر بافر در این مسیر پشتیبانی نمی‌شود)
  - پوشش ویژهٔ ارائه‌دهندهٔ Vydra:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_VYDRA_VIDEO=1 pnpm test:live -- extensions/vydra/vydra.live.test.ts`
    - آن فایل، متن‌به‌ویدئوی `veo3` را به‌همراه یک مسیر تصویر‌به‌ویدئوی `kling` اجرا می‌کند که به‌طور پیش‌فرض از یک فیکسچر URL تصویر راه‌دور استفاده می‌کند (`OPENCLAW_LIVE_VYDRA_KLING_IMAGE_URL` برای بازنویسی).
  - پوشش ویژهٔ ارائه‌دهندهٔ xAI:
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"`
    - حالت کلاسیک ابتدا یک فریم اول PNG محلی مربعی تولید می‌کند، هندسه را حذف می‌کند، یک کلیپ یک‌ثانیه‌ای تصویر‌به‌ویدئو درخواست می‌دهد، تا تکمیل نظرسنجی می‌کند و بافر بارگیری‌شده را راستی‌آزمایی می‌کند.
    - `OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"`
    - حالت 1.5 یک فریم اول PNG محلی تولید می‌کند، یک کلیپ یک‌ثانیه‌ای تصویر‌به‌ویدئوی 1080P درخواست می‌دهد، تا تکمیل نظرسنجی می‌کند و بافر بارگیری‌شده را راستی‌آزمایی می‌کند.
  - پوشش زندهٔ فعلی `videoToVideo`:
    - `runway` فقط هنگامی که مدل انتخاب‌شده به `gen4_aleph` تفکیک شود
  - ارائه‌دهندگان فعلی `videoToVideo` که اعلام شده‌اند اما در پیمایش مشترک رد می‌شوند:
    - `alibaba`،‏ `google`،‏ `openai`،‏ `qwen`،‏ `xai`، زیرا این مسیرها در حال حاضر به URLهای مرجع راه‌دور `http(s)` نیاز دارند، نه ورودی محلی مبتنی بر بافر
- محدودسازی اختیاری:
  - `OPENCLAW_LIVE_VIDEO_GENERATION_PROVIDERS="deepinfra,google,openai,runway"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_MODELS="google/veo-3.1-fast-generate-preview,openai/sora-2,runway/gen4_aleph"`
  - `OPENCLAW_LIVE_VIDEO_GENERATION_SKIP_PROVIDERS=""` برای گنجاندن همهٔ ارائه‌دهندگان در پیمایش پیش‌فرض، از جمله FAL
  - `OPENCLAW_LIVE_VIDEO_GENERATION_TIMEOUT_MS=60000` برای کاهش سقف عملیات هر ارائه‌دهنده در یک اجرای دود تهاجمی
- رفتار اختیاری احراز هویت:
  - `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` برای اجبار احراز هویت از مخزن پروفایل و نادیده‌گرفتن بازنویسی‌های صرفاً محیطی

## هارنس زندهٔ رسانه

- دستور: `pnpm test:live:media`
- نقطهٔ ورود: `test/e2e/qa-lab/media/hosted-media-provider-live.ts`، که برای هر مجموعهٔ انتخاب‌شده `pnpm test:live -- <suite-test-file>` را اجرا می‌کند تا رفتار Heartbeat و حالت بی‌صدا با دیگر اجراهای `pnpm test:live` سازگار بماند.
- هدف:
  - مجموعه‌های زندهٔ مشترک تصویر، موسیقی و ویدئو را از طریق یک نقطهٔ ورود بومی مخزن اجرا می‌کند
  - متغیرهای محیطی مفقود ارائه‌دهنده را به‌طور خودکار از `~/.profile` بارگذاری می‌کند
  - به‌طور پیش‌فرض هر مجموعه را به ارائه‌دهندگانی محدود می‌کند که در حال حاضر احراز هویت قابل‌استفاده دارند
- پرچم‌ها:
  - `--providers <csv>` پالایهٔ سراسری ارائه‌دهنده؛ `--image-providers` / `--music-providers` / `--video-providers` یک پالایه را به یک مجموعه محدود می‌کنند
  - `--all-providers` پالایش خودکار مبتنی بر احراز هویت را رد می‌کند
  - `--allow-empty` هنگامی که پس از پالایش هیچ ارائه‌دهندهٔ قابل‌اجرایی باقی نماند، با `0` خارج می‌شود
  - `--quiet` / `--no-quiet` به `test:live` منتقل می‌شوند
- نمونه‌ها:
  - `pnpm test:live:media`
  - `pnpm test:live:media image video --providers openai,google,minimax`
  - `pnpm test:live:media video --video-providers openai,runway --all-providers`
  - `pnpm test:live:media music --quiet`

## مرتبط

- [آزمایش](/fa/help/testing) - مجموعه‌های واحد، یکپارچه‌سازی، QA و Docker
