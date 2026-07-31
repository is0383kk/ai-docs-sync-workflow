---
read_when:
    - می‌خواهید برای یک عامل از چارچوب GitHub Copilot SDK استفاده کنید
    - برای runtime ‏`copilot` به نمونه‌های پیکربندی نیاز دارید
    - در حال اتصال یک عامل به اشتراک Copilot (github / openclaw / copilot) هستید و می‌خواهید آن را از طریق Copilot CLI اجرا کنید
summary: نوبت‌های عامل تعبیه‌شده OpenClaw را از طریق چارچوب خارجی GitHub Copilot SDK اجرا کنید
title: چارچوب آزمون SDK کوپایلت
x-i18n:
    generated_at: "2026-07-27T16:51:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4b67959c2c72bda97a81d0b45bc32ba363373064ec40c54f9709705dd15dd9fc
    source_path: plugins/copilot.md
    workflow: 16
---

Plugin خارجی `@openclaw/copilot` نوبت‌های عامل Copilot اشتراکیِ تعبیه‌شده را از طریق GitHub Copilot CLI (`@github/copilot-sdk`) اجرا می‌کند، نه از طریق چارچوب داخلی OpenClaw. نشست Copilot CLI مالک حلقه سطح‌پایین عامل است: اجرای بومی ابزار، Compaction بومی (`infiniteSessions`) و وضعیت رشته مدیریت‌شده توسط CLI در `copilotHome`. OpenClaw همچنان مالک کانال‌های گفت‌وگو، فایل‌های نشست، انتخاب مدل، ابزارهای پویا (پل‌زده‌شده)، تأییدها، تحویل رسانه، آینه قابل‌مشاهده رونوشت، پرسش‌های جانبی `/btw` (نگاه کنید به
[پرسش‌های جانبی (`/btw`)](#side-questions-btw)) و `openclaw doctor` است.

برای آشنایی با تفکیک گسترده‌تر مدل/ارائه‌دهنده/زمان‌اجرا، از
[زمان‌های اجرای عامل](/fa/concepts/agent-runtimes) شروع کنید.

## الزامات

- OpenClaw با Plugin نصب‌شده `@openclaw/copilot`.
- اگر پیکربندی شما از `plugins.allow` استفاده می‌کند، `copilot` (شناسه مانیفستی که
  Plugin اعلام می‌کند) را وارد کنید. ورودی فهرست مجاز برای نام بسته npm یعنی
  `@openclaw/copilot` مطابقت نخواهد داشت و Plugin را مسدود نگه می‌دارد، حتی با
  تنظیم `agentRuntime.id: "copilot"`.
- اشتراک GitHub Copilot که بتواند Copilot CLI را راه‌اندازی کند، یا یک
  متغیر محیطی `gitHubToken` / ورودی نمایه احراز هویت برای اجراهای بدون رابط یا Cron.
- دایرکتوری `copilotHome` با قابلیت نوشتن. وقتی OpenClaw یک دایرکتوری عامل
  ارائه می‌کند، مقدار پیش‌فرض `<agentDir>/copilot` است؛ در غیر این صورت
  `~/.openclaw/agents/<agentId>/copilot`.

`openclaw doctor` [قرارداد doctor](#doctor) مربوط به Plugin را برای مالکیت
وضعیت نشست و مهاجرت‌های پیکربندی آینده اجرا می‌کند. این فرمان محیط Copilot CLI
را بررسی نمی‌کند.

## نصب

زمان‌اجرای Copilot به‌صورت Plugin خارجی عرضه می‌شود تا بسته اصلی `openclaw`
شامل `@github/copilot-sdk` یا فایل اجرایی CLI مختص پلتفرم آن،
`@github/copilot-<platform>-<arch>`، نباشد (در مجموع حدود 260 MB).
آن را فقط برای عامل‌هایی نصب کنید که این زمان‌اجرا را برمی‌گزینند:

```bash
openclaw plugins install @openclaw/copilot
```

جادوگر راه‌اندازی، نخستین باری که یک مدل `github-copilot/*` را انتخاب می‌کنید
**و** پیکربندی شما آن مدل (یا ارائه‌دهنده‌اش) را از طریق `agentRuntime: { id: "copilot" }` به
زمان‌اجرای Copilot هدایت می‌کند، Plugin را خودکار نصب می‌کند؛
[شروع سریع](#quickstart) را ببینید. بدون این انتخاب صریح، OpenClaw از
ارائه‌دهنده داخلی GitHub Copilot استفاده می‌کند و هرگز این Plugin را نصب نمی‌کند.

زمان‌اجرا SDK را به این ترتیب پیدا می‌کند:

1. `import("@github/copilot-sdk")` از بسته نصب‌شده `@openclaw/copilot`.
2. دایرکتوری جایگزین `~/.openclaw/npm-runtime/copilot/` (هدف قدیمی نصب
   برحسب تقاضا).

نبود SDK یک خطا با کد `COPILOT_SDK_MISSING` و فرمان نصب مجدد بالا ایجاد می‌کند.

## شروع سریع

یک مدل (یا یک ارائه‌دهنده) را به چارچوب سنجاق کنید:

```json5
{
  agents: {
    defaults: {
      model: "github-copilot/auto",
      models: {
        "github-copilot/auto": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
}
```

`agentRuntime.id` را روی ورودی یک مدل تنظیم کنید تا فقط همان مدل از طریق
چارچوب هدایت شود، یا آن را روی یک ارائه‌دهنده تنظیم کنید تا همه مدل‌های زیرمجموعه
آن ارائه‌دهنده هدایت شوند.

`github-copilot/auto` نقطه شروع قابل‌حمل است. مدل‌های نام‌گذاری‌شده Copilot به
خط‌مشی حساب و سازمان وابسته‌اند؛ پیش از سنجاق‌کردن یک مدل، تأیید کنید که
Copilot CLI احراز هویت‌شده شما واقعاً آن را ارائه می‌کند.

## ارائه‌دهندگان پشتیبانی‌شده

چارچوب از ارائه‌دهنده متعارف `github-copilot` (متعلق به
`extensions/github-copilot`) و نیز ورودی‌های سفارشی `models.providers` پشتیبانی می‌کند،
به‌شرط آنکه مدل دارای `baseUrl` غیرخالی و یکی از شکل‌های
`api` زیر باشد:

- `anthropic-messages`
- `azure-openai-responses`
- `ollama` (تکمیل‌های سازگار با OpenAI)
- `openai-completions`
- `openai-responses`

شناسه‌های ارائه‌دهنده بومی (`openai`، `anthropic`، `google`، `ollama`) همچنان متعلق به
زمان‌های اجرای بومی خود هستند. برای هدایت یک نقطه پایانی از طریق Copilot BYOK،
به‌جای آن از شناسه ارائه‌دهنده سفارشی و متمایزی استفاده کنید.

نقاط پایانی Copilot BYOK باید URLهای عمومی HTTPS باشند. چارچوب در هر تلاش
یک پراکسی حلقه‌بازگشتی به Copilot SDK می‌دهد، سپس ترافیک ارائه‌دهنده را از
مسیر fetch محافظت‌شده OpenClaw عبور می‌دهد تا سنجاق‌کردن DNS و خط‌مشی SSRF
همچنان تحت مالکیت OpenClaw بمانند. برای Ollama محلی، LM Studio یا
سرورهای مدل LAN از زمان‌اجرای بومی OpenClaw استفاده کنید.

## BYOK

Copilot BYOK از قرارداد ارائه‌دهنده سفارشی در سطح نشست SDK استفاده می‌کند.
OpenClaw نقطه پایانی مدل حل‌شده، کلید API، حالت توکن حامل، سرآیندها، شناسه مدل
و محدودیت‌های زمینه/خروجی را ارسال می‌کند؛ منطق انتقال ارائه‌دهنده در SDK
باقی می‌ماند، نه در هسته.

```json5
{
  agents: {
    defaults: {
      model: "custom-proxy/llama-3.1-8b",
      models: {
        "custom-proxy/llama-3.1-8b": {
          agentRuntime: { id: "copilot" },
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      "custom-proxy": {
        baseUrl: "https://api.example.com/v1",
        apiKey: "${CUSTOM_PROXY_API_KEY}",
        api: "openai-responses",
        authHeader: true,
        models: [{ id: "llama-3.1-8b", name: "Llama 3.1 8B" }],
      },
    },
  },
}
```

نشست‌های BYOK جدا از نشست‌های اشتراکی و سایر نقاط پایانی یا اعتبارنامه‌های
BYOK کلیدگذاری می‌شوند. چرخش کلید، سرآیندها، مدل یا نقطه پایانی، به‌جای
ازسرگیری وضعیت ناسازگار، یک نشست تازه Copilot SDK آغاز می‌کند.

## احراز هویت

ترتیب تقدم که در جریان `runCopilotAttempt` برای هر عامل اعمال می‌شود:

1. **`useLoggedInUser: true` صریح** در ورودی تلاش — از کاربر واردشده
   Copilot CLI در `copilotHome` عامل استفاده می‌کند.
2. **`gitHubToken` صریح** در ورودی تلاش (نیازمند `profileId` +
   `profileVersion`). برای فراخوانی‌های مستقیم CLI و آزمون‌هایی که باید
   حل نمایه احراز هویت را دور بزنند.
3. **`resolvedApiKey` + `authProfileId` حل‌شده توسط قرارداد** — مسیر
   اصلی تولید. هسته پیش از فراخوانی چارچوب، نمایه احراز هویت پیکربندی‌شده
   `github-copilot` عامل (`src/infra/provider-usage.auth.ts:resolveProviderAuths`) را حل می‌کند؛ بنابراین یک
   نمایه احراز هویت `github-copilot:<profile>` برای راه‌اندازی‌های بدون رابط، Cron یا
   چندنمایه‌ای بدون متغیر محیطی، از ابتدا تا انتها کار می‌کند.
4. **جایگزین متغیر محیطی**، با بررسی به این ترتیب (نخستین مقدار غیرخالی
   برنده است؛ رشته‌های خالی غایب محسوب می‌شوند؛ ترتیب تقدم ارائه‌دهنده
   عرضه‌شده `github-copilot` در `extensions/github-copilot/auth.ts` را بازتاب می‌دهد):
   1. `OPENCLAW_GITHUB_TOKEN` — بازنویسی مختص چارچوب؛ اجازه می‌دهد یک
      توکن را برای چارچوب OpenClaw سنجاق کنید، بدون آنکه پیکربندی سراسری
      `gh` / Copilot CLI را مختل کنید.
   2. `COPILOT_GITHUB_TOKEN` — متغیر محیطی استاندارد Copilot SDK / CLI.
   3. `GH_TOKEN` — متغیر محیطی استاندارد CLI مربوط به `gh`.
   4. `GITHUB_TOKEN` — جایگزین عمومی توکن GitHub.

   شناسه نمایه مخزن ترکیبی `env:<NAME>` است؛ نسخه نمایه یک اثر انگشت
   برگشت‌ناپذیر sha256 از توکن است، بنابراین چرخش مقدار محیطی، مخزن کلاینت
   را به‌طور پاک بازنشانی می‌کند.

5. **`useLoggedInUser` پیش‌فرض** وقتی هیچ نشانه توکنی در دسترس نیست.

هر عامل `copilotHome` مخصوص خود را دریافت می‌کند تا توکن‌ها، نشست‌ها و
پیکربندی Copilot CLI هرگز میان عامل‌های یک دستگاه نشت نکنند. مقدار پیش‌فرض:
`<agentDir>/copilot` (وضعیت SDK را خارج از همان دایرکتوری
`models.json` / `auth-profiles.json` متعلق به OpenClaw نگه می‌دارد)، یا
`~/.openclaw/agents/<agentId>/copilot` وقتی هیچ دایرکتوری عاملی ارائه نشده باشد.
برای مکان سفارشی (برای مثال، یک سوارسازی مشترک برای مهاجرت)، آن را با
`copilotHome: <path>` در ورودی تلاش بازنویسی کنید.

آزمون‌های زنده چارچوب از `OPENCLAW_COPILOT_AGENT_LIVE_TOKEN` برای یک
توکن مستقیم استفاده می‌کنند. راه‌اندازی مشترک آزمون زنده پس از آماده‌سازی
نمایه‌های احراز هویت واقعی در خانه ایزوله آزمون، `COPILOT_GITHUB_TOKEN`، `GH_TOKEN`
و `GITHUB_TOKEN` را پاک می‌کند؛ بنابراین مقدار `gh auth token` که از
طریق متغیر اختصاصی ارسال می‌شود، بدون نشت به مجموعه‌آزمون‌های نامرتبط از
ردشدن‌های کاذب جلوگیری می‌کند.

## سطح پیکربندی

چارچوب پیکربندی را از ورودی هر تلاش (`runCopilotAttempt({...})`)
به‌همراه مجموعه کوچکی از پیش‌فرض‌های محیطی در `extensions/copilot/src/` می‌خواند:

| فیلد                    | هدف                                                                                                                                                                                                                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `copilotHome`            | دایرکتوری وضعیت CLI برای هر عامل (پیش‌فرض‌ها در بالا).                                                                                                                                                                                                                                                 |
| `model`                  | رشته یا `{ provider, id, api?, baseUrl?, headers?, authHeader? }`. برای استفاده از انتخاب عادی مدل عامل، آن را حذف کنید؛ چارچوب بررسی می‌کند که ارائه‌دهنده حل‌شده پشتیبانی می‌شود.                                                                                                                   |
| `reasoningEffort`        | `"low" \| "medium" \| "high" \| "xhigh"`. از حل `ThinkLevel` / `ReasoningLevel` متعلق به OpenClaw در `auto-reply/thinking.ts` نگاشت می‌شود.                                                                                                                                                          |
| `infiniteSessionConfig`  | بازنویسی اختیاری برای بلوک `infiniteSessions` در SDK که توسط `harness.compact` هدایت می‌شود. می‌توان بدون تغییر رهایش کرد.                                                                                                                                                                                        |
| `hooksConfig`            | پیکربندی بومی و اختیاری `SessionHooks` مربوط به Copilot SDK برای فراخوان‌های برگشتی ابزار/MCP، اعلان کاربر، نشست و خطا. جدا از قلاب‌های چرخه‌عمر قابل‌حمل OpenClaw.                                                                                                                                   |
| `permissionPolicy`       | بازنویسی اختیاری برای کنترل‌کننده `onPermissionRequest` در SDK برای انواع ابزار داخلی SDK (`shell`، `write`، `read`، `url`، `mcp`، `memory`، `hook`). مقدار پیش‌فرض برای ایمنی `rejectAllPolicy` است؛ برای دلیل اینکه هرگز واقعاً فعال نمی‌شود، [مجوزها و ask_user](#permissions-and-ask_user) را ببینید. |
| `enableSessionTelemetry` | پرچم اختیاری تله‌متری نشست SDK.                                                                                                                                                                                                                                                            |

قلاب‌های Plugin در OpenClaw به پیکربندی تلاش مختص Copilot نیازی ندارند.
چارچوب `before_prompt_build`، `llm_input`، `llm_output` و `agent_end` را از طریق
کمک‌تابع‌های استاندارد چارچوب اجرا می‌کند. Compactionهای موفق SDK همچنین
`before_compaction` و `after_compaction` را اجرا می‌کنند. ابزارهای پل‌زده‌شده
OpenClaw، `before_tool_call` را اجرا و `after_tool_call` را گزارش می‌کنند؛
`hooksConfig` برای فراخوان‌های برگشتی صرفاً بومی SDK که معادل قابل‌حمل
ندارند، باقی می‌ماند.

هیچ بخش دیگری از OpenClaw لازم نیست درباره این فیلدها بداند. سایر Pluginها،
کانال‌ها و کد هسته فقط شکل استاندارد `AgentHarnessAttemptParams` /
`AgentHarnessAttemptResult` را می‌بینند.

## Compaction

وقتی `harness.compact` اجرا می‌شود، چارچوب Copilot SDK:

1. نشست پیگیری‌شده SDK را بدون ادامه کار معلق از سر می‌گیرد.
2. RPC مربوط به Compaction تاریخچه در سطح نشست SDK را فراخوانی می‌کند.
3. نتیجه Compaction در SDK را بدون نوشتن فایل‌های نشانگر سازگاری
   در فضای کاری بازمی‌گرداند.

آینه رونوشت در سمت OpenClaw (در ادامه) همچنان پیام‌های پس از Compaction را
دریافت می‌کند؛ بنابراین تاریخچه گفت‌وگوی قابل‌مشاهده برای کاربر سازگار می‌ماند.

## آینه‌سازی رونوشت

`runCopilotAttempt` در هر نوبت، پیام‌های قابل بازتاب آن نوبت را از طریق
`extensions/copilot/src/dual-write-transcripts.ts` به‌صورت دوگانه در رونوشت ممیزی
OpenClaw می‌نویسد. دامنه بازتاب برای هر
نشست (`copilot:${sessionId}`) جدا و کلید آن برای هر پیام
(`${role}:${sha256_16(role,content)}`) مستقل است؛ بنابراین ورودی‌های نوبت‌های قبلی که دوباره منتشر می‌شوند،
به‌جای تکرار، با کلیدهای موجود روی دیسک برخورد می‌کنند.

دو لایه مهار خرابی، بازتاب را در بر می‌گیرند تا خرابی در نوشتن رونوشت
هرگز تلاش را ناموفق نکند: یک پوشش داخلی با رویکرد بهترین تلاش، به‌علاوه یک
`.catch(...)` دفاع در عمق در سطح تلاش. خرابی‌ها ثبت می‌شوند، اما
نمایان نمی‌شوند.

## پرسش‌های جانبی (`/btw`)

`/btw` در این هارنس بومی **نیست**. `createCopilotAgentHarness()`
عمداً `harness.runSideQuestion` را تعریف‌نشده باقی می‌گذارد
(طبق بررسی‌های `extensions/copilot/harness.test.ts` و `describe("runSideQuestion")`)؛
بنابراین توزیع‌کننده `/btw` متعلق به OpenClaw
(`src/agents/btw.ts`) به همان مسیری می‌افتد که برای هر زمان‌اجرای غیر Codex
استفاده می‌کند: ارائه‌دهنده مدل پیکربندی‌شده مستقیماً با یک پرامپت کوتاهِ
پرسش جانبی فراخوانی می‌شود و پاسخ از طریق `streamSimple` به‌صورت جریانی
بازگردانده می‌شود (بدون نشست CLI و بدون اشغال جایگاه اضافی در مخزن).

این کار نشست‌های Copilot CLI را برای حلقه اصلی نوبت عامل محفوظ نگه می‌دارد و
رفتار `/btw` را با سایر زمان‌های‌اجرای غیر Codex یکسان نگه می‌دارد.

## Doctor

`extensions/copilot/doctor-contract-api.ts` به‌طور خودکار توسط
`src/plugins/doctor-contract-registry.ts` بارگذاری می‌شود. موارد زیر را فراهم می‌کند:

- یک `legacyConfigRules` خالی (هنوز هیچ فیلد بازنشسته‌ای وجود ندارد).
- یک `normalizeCompatibilityConfig` بدون عملیات (حفظ شده است تا بازنشستگی فیلدهای آینده
  جایگاهی پایدار در درخت مخزن داشته باشد).
- یک ورودی `sessionRouteStateOwners`: ارائه‌دهنده `github-copilot`، زمان‌اجرا
  `copilot`، کلید نشست CLI برابر با `copilot` و پیشوند نمایه احراز هویت `github-copilot:`.

## محدودیت‌ها

- هارنس، مالکیت `github-copilot` به‌همراه شناسه‌های ارائه‌دهنده سفارشی و بدون مالک BYOK را بر عهده می‌گیرد.
  شناسه‌های بومی ارائه‌دهنده که مالک آن‌ها در مانیفست مشخص است، حتی وقتی
  `agentRuntime.id` به‌اجبار روی `copilot` تنظیم شود، در زمان‌اجرای مالک خود باقی می‌مانند.
- هیچ سطح TUI وجود ندارد؛ TUI متعلق به PI برای زمان‌های‌اجرای فاقد سطح همتا،
  گزینه جایگزین باقی می‌ماند.
- وقتی یک عامل به `copilot` تغییر می‌کند، وضعیت نشست PI منتقل نمی‌شود.
  انتخاب برای هر تلاش جداگانه است؛ نشست‌های موجود PI همچنان معتبر می‌مانند.
- `ask_user` از زمان‌اجرای پرسش Gateway مستقل از ارائه‌دهنده استفاده می‌کند. رابط کاربری Control
  همان کارت پرسش سایر پرسش‌های OpenClaw را نمایش می‌دهد، کانال‌های پشتیبانی‌شده
  دکمه‌های انتخاب را رندر می‌کنند و پیام متنی ساده بعدی در صف،
  پیش از بازگشت درخواست SDK، آن رکورد Gateway را حل‌وفصل می‌کند.

## مجوزها و ask_user

اجرای مجوزها برای ابزارهای پل‌شده OpenClaw **درون پوشش ابزار**
انجام می‌شود، نه از طریق فراخوان بازگشتی `onPermissionRequest` متعلق به SDK. همان
`wrapToolWithBeforeToolCallHook` که PI استفاده می‌کند
(`src/agents/agent-tools.before-tool-call.ts`) توسط
`createOpenClawCodingTools` برای همه ابزارهای کدنویسی اعمال می‌شود: تشخیص حلقه، سیاست‌های
Plugin مورداعتماد، قلاب‌های پیش از فراخوانی ابزار و تأییدهای دومرحله‌ای Plugin از طریق
Gateway (`plugin.approval.request`) همگی دقیقاً از همان مسیر کدی عبور می‌کنند
که تلاش‌های بومی PI استفاده می‌کنند.

هر ابزار SDK که پل ابزار Copilot بازمی‌گرداند، با موارد زیر علامت‌گذاری می‌شود:

- `overridesBuiltInTool: true` — ابزار داخلی هم‌نام Copilot CLI
  (edit، read، write، bash، ...) را جایگزین می‌کند تا هر فراخوانی ابزار دوباره
  به OpenClaw هدایت شود.
- `skipPermission: true` — به SDK می‌گوید پیش از فراخوانی ابزار،
  `onPermissionRequest({kind: "custom-tool"})` را اجرا نکند.
  `execute()` پوشش‌داده‌شده از قبل بررسی سیاست غنی‌تر OpenClaw را انجام می‌دهد؛
  یک پرامپت در سطح SDK یا اجرای سیاست OpenClaw را میان‌بُر می‌زد
  (اجازه به همه) یا هر فراخوانی ابزار را مسدود می‌کرد (رد همه) — هیچ‌یک با
  برابری PI مطابقت ندارد.

هارنس Codex داخل مخزن از همین تفکیک استفاده می‌کند: ابزارهای پل‌شده OpenClaw
پوشش داده می‌شوند (`extensions/codex/src/app-server/dynamic-tools.ts`) و
گونه‌های تأیید بومی خود codex-app-server
(`item/commandExecution/requestApproval`، `item/fileChange/requestApproval`،
`item/permissions/requestApproval`) از طریق `plugin.approval.request`
(`extensions/codex/src/app-server/approval-bridge.ts`) هدایت می‌شوند. معادل آن در SDK متعلق به Copilot
— یعنی `rejectAllPolicy` با رفتار بسته در حالت خرابی برای هر گونه غیر `custom-tool`
که در نهایت به `onPermissionRequest` برسد — همان شبکه ایمنی است و
در عمل هرگز فعال نمی‌شود، زیرا `overridesBuiltInTool: true` همه
ابزارهای داخلی را کنار می‌زند.

برای اینکه لایه ابزار پوشش‌داده‌شده تصمیم‌های سیاستی معادل PI بگیرد،
هارنس زمینه کامل ابزارِ تلاش PI را به
`createOpenClawCodingTools` ارسال می‌کند: هویت (`senderIsOwner`، `memberRoleIds`،
`ownerOnlyToolAllowlist`، ...)، کانال/مسیریابی (`groupId`،
`currentChannelId`، `replyToMode`، کلیدهای تغییر وضعیت ابزار پیام)، احراز هویت
(`authProfileStore`)، هویت اجرا (`sessionKey` / `runSessionKey` که از
`sandboxSessionKey` و `runId` مشتق شده‌اند)، زمینه مدل (`modelApi`،
`modelContextWindowTokens`، `modelCompat`، `modelHasVision`) و قلاب‌های اجرا
(`onToolOutcome`، `onYield`). بدون این فیلدها، فهرست‌های مجاز مختص مالک
به‌طور خاموش و پیش‌فرض درخواست را رد می‌کنند، سیاست‌های اعتماد Plugin نمی‌توانند دامنه درست را
تشخیص دهند و `session_status: "current"` به یک کلید منسوخ sandbox حل می‌شود. سازنده
پل `extensions/copilot/src/tool-bridge.ts` است که فراخوانی مرجع PI در
`src/agents/embedded-agent-runner/run/attempt.ts:1262` را بازتاب می‌دهد.
`runAttempt` زمینه sandbox را از طریق درگاه مشترک
`resolveSandboxContext` حل می‌کند، یک دایرکتوری کاری مؤثر به SDK می‌دهد
و `sandbox` را به‌همراه فضای کاری ایجاد زیرعامل به پل ابزار
ارسال می‌کند. پل همچنین کنترل‌های محدود ساخت ابزار را که
می‌تواند در مرز SDK اعمال کند، ارسال می‌کند: `includeCoreTools`، فهرست مجاز ابزارهای
زمان‌اجرا و `toolConstructionPlan`.

پل برای برابری با PI از راهنمای مشترک سطح ابزار هارنس در
`openclaw/plugin-sdk/agent-harness-tool-runtime` نیز استفاده می‌کند. وقتی
جست‌وجوی ابزار فعال باشد، SDK به‌جای طرح‌واره همه ابزارهای OpenClaw، ابزارهای کنترلی
فشرده به‌همراه یک اجراکننده پنهان کاتالوگ را می‌بیند. وقتی حالت کد
فعال باشد، راهنما همان سطح کنترل حالت کد و چرخه عمر کاتالوگ را می‌سازد
که سایر هارنس‌های عامل استفاده می‌کنند. پیش‌فرض‌های سبک مدل محلی،
پالایش طرح‌واره سازگار با زمان‌اجرا، آب‌رسانی دایرکتوری و پاک‌سازی کاتالوگ
همگی در راهنمای مشترک باقی می‌مانند تا هارنس‌های Copilot و مجاور Codex
از هم منحرف نشوند.

### توکن GitHub در سطح نشست

قرارداد SDK متعلق به Copilot میان توکن GitHub در **سطح کلاینت**
(`CopilotClientOptions.gitHubToken`، که خود فرایند CLI را احراز هویت می‌کند)
و توکن در **سطح نشست** (`SessionConfig.gitHubToken`، که
حذف محتوا، مسیریابی مدل و سهمیه آن نشست را تعیین می‌کند و در هر دو
`createSession` و `resumeSession` رعایت می‌شود) تمایز قائل است. هارنس احراز هویت را یک‌بار از طریق
`resolveCopilotAuth` حل می‌کند و وقتی حالت احراز هویت `gitHubToken` باشد،
هر دو فیلد را تنظیم می‌کند (یک `auth.gitHubToken` صریح یا یک `resolvedApiKey`
حل‌شده طبق قرارداد از نمایه احراز هویت `github-copilot` پیکربندی‌شده). وقتی حالت حل‌شده
`useLoggedInUser` باشد، فیلد سطح نشست حذف می‌شود تا SDK همچنان
هویت را از هویت واردشده استخراج کند.

`ask_user` از `SessionConfig.onUserInputRequest` استفاده می‌کند. پل، گزینه‌های SDK
یا پرامپت‌های متن آزاد بدون گزینه را به‌عنوان پرسش‌های Gateway ثبت می‌کند، برای درخواست‌های
دارای گزینه ثابت، شاخص‌ها یا برچسب‌های گزینه‌ها را می‌پذیرد و وقتی درخواست SDK اجازه دهد،
پاسخ‌های آزاد را قبول می‌کند. لغو تلاش OpenClaw، رکورد
Gateway را لغو می‌کند و یک پاسخ خالی SDK بازمی‌گرداند.

## مرتبط

- [زمان‌های‌اجرای عامل](/fa/concepts/agent-runtimes)
- [هارنس Codex](/fa/plugins/codex-harness)
- [Pluginهای هارنس عامل (مرجع SDK)](/fa/plugins/sdk-agent-harness)
