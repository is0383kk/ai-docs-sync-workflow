---
read_when:
    - اجرای آزمون‌ها به‌صورت محلی یا در CI
    - افزودن آزمون‌های رگرسیون برای باگ‌های مدل/ارائه‌دهنده
    - اشکال‌زدایی رفتار Gateway و عامل
summary: 'مجموعه‌ابزار آزمون: مجموعه‌های آزمون واحد، سرتاسری و زنده، اجراکننده‌های Docker و آنچه هر آزمون پوشش می‌دهد'
title: آزمایش
x-i18n:
    generated_at: "2026-07-27T15:35:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 20e0aa22bf16561334f83342abffabb387ed0b41b901773939123ecfbc0ae330
    source_path: help/testing.md
    workflow: 16
---

OpenClaw دارای سه مجموعهٔ Vitest (واحد/یکپارچه‌سازی، e2e، زنده) به‌علاوهٔ اجراکننده‌های
Docker است. این صفحه توضیح می‌دهد هر مجموعه چه مواردی را پوشش می‌دهد، برای یک
گردش‌کار مشخص کدام فرمان باید اجرا شود، آزمون‌های زنده چگونه اطلاعات احراز هویت را پیدا می‌کنند، و چگونه
برای باگ‌های واقعی ارائه‌دهنده/مدل آزمون‌های رگرسیون اضافه کنید.

<Note>
**پشتهٔ QA (qa-lab، qa-channel، مسیرهای انتقال زنده)** به‌طور جداگانه مستند شده است:

- [نمای کلی QA](/fa/concepts/qa-e2e-automation) - معماری، سطح فرمان‌ها، نگارش سناریو، و پروفایل‌های Matrix.
- [کارت امتیاز بلوغ](/fa/maturity/scorecard) - اینکه شواهد QA انتشار چگونه از تصمیم‌های پایداری و LTS پشتیبانی می‌کنند.
- [کانال QA](/fa/channels/qa-channel) - Plugin انتقال مصنوعی که سناریوهای متکی بر مخزن از آن استفاده می‌کنند.

این صفحه مجموعه‌های آزمون معمول و اجراکننده‌های Docker/Parallels را پوشش می‌دهد. [اجراکننده‌های ویژهٔ QA](#qa-specific-runners) در ادامه، فراخوانی‌های مشخص `qa` را فهرست می‌کند و به منابع بالا ارجاع می‌دهد.
</Note>

## شروع سریع

در بیشتر روزها:

- گیت کامل (مورد انتظار پیش از push): `pnpm build && pnpm check && pnpm check:test-types && pnpm test`
- اجرای محلی سریع‌ترِ مجموعهٔ کامل روی دستگاهی با منابع کافی: `pnpm test:max`
- حلقهٔ مستقیم پایش Vitest: `pnpm test:watch`
- هدف‌گیری مستقیم فایل، مسیرهای Plugin/کانال را نیز هدایت می‌کند: `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts`
- هنگام کار تکرارشونده روی یک خرابی، ابتدا اجراهای هدفمند را ترجیح دهید.
- سایت QA مبتنی بر Docker: `pnpm qa:lab:up`
- مسیر QA مبتنی بر ماشین مجازی Linux: `pnpm openclaw qa suite --runner multipass --scenario channel-chat-baseline`

وقتی آزمون‌ها را تغییر می‌دهید یا اطمینان بیشتری می‌خواهید:

- گزارش پوشش اطلاع‌رسان V8: `pnpm test:coverage`
- مجموعهٔ E2E: `pnpm test:e2e`

## پوشه‌های موقت آزمون

برای پوشه‌های موقتی که مالک آن‌ها آزمون است، از ابزارهای کمکی مشترک در `test/helpers/temp-dir.ts`
استفاده کنید تا مالکیت صریح باشد و پاک‌سازی در چرخهٔ عمر آزمون باقی بماند:

```ts
import { afterEach } from "vitest";
import { useAutoCleanupTempDirTracker } from "../helpers/temp-dir.js";

const tempDirs = useAutoCleanupTempDirTracker(afterEach);

it("از یک فضای کاری موقت استفاده می‌کند", () => {
  const workspace = tempDirs.make("openclaw-example-");
  // استفاده از فضای کاری
});
```

`useAutoCleanupTempDirTracker(afterEach)` عمداً هیچ روش پاک‌سازی
دستی ارائه نمی‌کند - Vitest پس از هر آزمون مالک پاک‌سازی است. ابزارهای کمکی قدیمی‌ترِ سطح پایین
(`makeTempDir`، `cleanupTempDirs`، `createTempDirTracker`) همچنان برای آزمون‌هایی که
مهاجرت نکرده‌اند وجود دارند؛ از کاربرد جدید آن‌ها و فراخوانی‌های جدید و بدون پوشش
`fs.mkdtemp*` خودداری کنید، مگر اینکه آزمونی صراحتاً رفتار خام پوشهٔ موقت را
بررسی کند. هنگامی که واقعاً به یک پوشهٔ موقت بدون پوشش نیاز است، یک توضیح مجاز و قابل‌ممیزی
همراه با دلیل اضافه کنید:

```ts
// openclaw-temp-dir: allow رفتار پاک‌سازی خام fs را بررسی می‌کند
const workspace = fs.mkdtempSync(prefix);
```

`node scripts/report-test-temp-creations.mjs` ایجاد جدید پوشهٔ موقت بدون پوشش
و استفادهٔ جدید و دستی از ابزار کمکی مشترک را در خطوط افزوده‌شدهٔ diff گزارش می‌کند، بدون اینکه
سبک‌های پاک‌سازی موجود را مسدود کند. این گزارش از همان دسته‌بندی مسیر آزمون
در `scripts/changed-lanes.mjs` پیروی می‌کند و خودِ پیاده‌سازی ابزار کمکی مشترک را
نادیده می‌گیرد. `check:changed` این گزارش را برای مسیرهای آزمون تغییریافته به‌عنوان
سیگنال CI صرفاً هشداردهنده اجرا می‌کند (حاشیه‌نویسی‌های هشدار GitHub، نه خرابی).

## گردش‌کارهای زنده و Docker/Parallels

هنگام اشکال‌زدایی ارائه‌دهندگان/مدل‌های واقعی (نیازمند اطلاعات احراز هویت واقعی):

- مجموعهٔ زنده (مدل‌ها + کاوش‌های ابزار/تصویر Gateway): `pnpm test:live`
- هدف‌گیری بی‌سروصدای یک فایل زنده: `pnpm test:live -- src/agents/models.profiles.live.test.ts`
- گزارش‌های عملکرد زمان اجرا: `OpenClaw Performance` را با
  `live_openai_candidate=true` برای یک نوبت واقعی عامل `openai/gpt-5.6-luna` یا
  `deep_profile=true` برای مصنوعات CPU/heap/trace مربوط به Kova اجرا کنید. اجراهای زمان‌بندی‌شدهٔ روزانه
  گزارش‌های مسیر ارائه‌دهندهٔ ساختگی، پروفایل عمیق و GPT-5.6 Luna را از طریق یک کار انتشاردهندهٔ جداگانهٔ مصرف‌کنندهٔ مصنوعات
  در `openclaw/clawgrit-reports` منتشر می‌کنند؛
  نبود یا نامعتبر بودن احراز هویت انتشاردهنده باعث شکست اجراهای زمان‌بندی‌شده و
  `profile=release` می‌شود. اجراهای دستی غیرانتشاری، مصنوعات GitHub را
  حفظ می‌کنند و انتشار گزارش را جنبهٔ توصیه‌ای می‌دانند. گزارش ارائه‌دهندهٔ ساختگی همچنین
  شامل اعداد راه‌اندازی Gateway در سطح منبع، حافظه، فشار Plugin، حلقهٔ سلام تکرارشوندهٔ
  مدل ساختگی و راه‌اندازی CLI است.
- پویش زندهٔ مدل در Docker: `pnpm test:docker:live-models`
  - هر مدل انتخاب‌شده یک نوبت متنی به‌علاوهٔ یک کاوش کوچک به‌سبک خواندن فایل اجرا می‌کند.
    مدل‌هایی که فرادادهٔ آن‌ها ورودی `image` را اعلام می‌کند، یک نوبت تصویری کوچک نیز اجرا می‌کنند.
    هنگام جداسازی خرابی‌های ارائه‌دهنده، کاوش‌های اضافی را با `OPENCLAW_LIVE_MODEL_FILE_PROBE=0` یا
    `OPENCLAW_LIVE_MODEL_IMAGE_PROBE=0` غیرفعال کنید.
  - پوشش CI: هر دو `OpenClaw Scheduled Live And E2E Checks` روزانه و
    `OpenClaw Release Checks` دستی، گردش‌کار زنده/E2E قابل‌استفادهٔ مجدد را با
    `include_live_suites: true` فراخوانی می‌کنند که شامل کارهای ماتریس مدل زندهٔ Docker
    تقسیم‌شده بر اساس ارائه‌دهنده است.
  - برای اجرای مجدد متمرکز CI، `OpenClaw Live And E2E Checks (Reusable)` را
    با `include_live_suites: true` و `live_models_only: true` اجرا کنید.
  - رازهای جدید و پرسیگنال ارائه‌دهنده را به `scripts/ci-hydrate-live-auth.sh`
    به‌علاوهٔ `.github/workflows/openclaw-live-and-e2e-checks-reusable.yml` و فراخوان‌های
    زمان‌بندی‌شده/انتشاری آن اضافه کنید.
- آزمون دود native گفت‌وگوی مقید Codex: `pnpm test:docker:live-codex-bind`
  - یک مسیر زندهٔ Docker را در برابر مسیر app-server متعلق به Codex اجرا می‌کند، یک
    پیام خصوصی مصنوعی Slack را با `/codex bind` مقید می‌کند، `/codex fast` و
    `/codex permissions` را به‌کار می‌گیرد، سپس تأیید می‌کند یک پاسخ ساده و یک پیوست تصویر
    به‌جای ACP از طریق اتصال native Plugin مسیریابی می‌شوند.
- آزمون دود مهار app-server مربوط به Codex: `pnpm test:docker:live-codex-harness`
  - نوبت‌های عامل Gateway را از طریق مهار app-server مربوط به Codex که مالک آن Plugin است
    اجرا می‌کند، `/codex status` و `/codex models` را تأیید می‌کند و به‌طور پیش‌فرض
    کاوش‌های تصویر، MCP مربوط به cron، عامل فرعی و Guardian را به‌کار می‌گیرد. هنگام
    جداسازی خرابی‌های دیگر، کاوش عامل فرعی را با `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=0`
    غیرفعال کنید. برای بررسی متمرکز عامل فرعی، کاوش‌های
    دیگر را غیرفعال کنید:
    `OPENCLAW_LIVE_CODEX_HARNESS_IMAGE_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_MCP_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_GUARDIAN_PROBE=0 OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_PROBE=1 pnpm test:docker:live-codex-harness`.
    این اجرا پس از کاوش عامل فرعی خارج می‌شود، مگر اینکه
    `OPENCLAW_LIVE_CODEX_HARNESS_SUBAGENT_ONLY=0` تنظیم شده باشد.
- آزمون دود نصب درخواستی Codex: `pnpm test:docker:codex-on-demand`
  - بستهٔ tar مربوط به OpenClaw را در Docker نصب می‌کند، راه‌اندازی اولیه با کلید API
    مربوط به OpenAI را اجرا می‌کند و تأیید می‌کند Plugin مربوط به Codex به‌همراه وابستگی `@openai/codex`
    در صورت نیاز در ریشهٔ مدیریت‌شدهٔ پروژهٔ npm بارگیری شده‌اند.
- آزمون دود زندهٔ بستهٔ npm-Plugin مربوط به Codex: `pnpm test:docker:live-codex-npm-plugin`
  - بستهٔ نامزد OpenClaw و Plugin دقیق Codex را در Docker نصب می‌کند،
    سپس برای پیش‌بررسی CLI و نوبت‌های همان نشست از یک کلید واقعی OpenAI استفاده می‌کند.
  - نوبت پیگیری آن با تفکر متوسط و بدون تلاش مجدد باید پیشرفت را ارسال کند، کار
    روی خواندن‌های تصادفی فضای کاری و نوشتن یک مصنوع دقیق را ادامه دهد،
    سپس تکمیل را ارسال کند. نوبت پایانی که فقط پیشرفت را گزارش دهد باعث شکست مسیر می‌شود.
- آزمون دود زندهٔ وابستگی ابزار Plugin: `pnpm test:docker:live-plugin-tool`
  - یک Plugin آزمایشی را با یک وابستگی واقعی `slugify` بسته‌بندی می‌کند، آن را
    از طریق `npm-pack:` نصب می‌کند، وابستگی را زیر ریشهٔ مدیریت‌شدهٔ پروژهٔ npm
    تأیید می‌کند، سپس از یک مدل زندهٔ OpenAI می‌خواهد ابزار Plugin را فراخوانی کند و
    شناسهٔ متنی پنهان را برگرداند.
- آزمون دود فرمان نجات OpenClaw: `pnpm test:live:system-agent-rescue-channel`
  - بررسی دفاع چندلایهٔ اختیاری برای سطح فرمان نجات کانال پیام.
    `/openclaw status` را به‌کار می‌گیرد، یک تغییر پایدار مدل را در صف قرار می‌دهد،
    با `/openclaw yes` پاسخ می‌دهد و مسیر نوشتن ممیزی/پیکربندی را تأیید می‌کند.
- آزمون دود نخستین اجرای OpenClaw در Docker: `pnpm test:docker:system-agent-first-run`
  - از یک پوشهٔ وضعیت خالی OpenClaw آغاز می‌کند و ابتدا ثابت می‌کند CLI بسته‌بندی‌شدهٔ
    `openclaw setup` بدون استنتاج به‌صورت بسته شکست می‌خورد. سپس
    Claude ساختگی را از طریق ماژول فعال‌سازی بسته‌بندی‌شده آزمایش و فعال می‌کند.
    تنها پس از آن، یک درخواست مبهم CLI بسته‌بندی‌شده به برنامه‌ریز می‌رسد و
    به راه‌اندازی نوع‌دار تفکیک می‌شود، و پس از آن عملیات یک‌بارهٔ مدل، عامل، پیکربندی Discord
    و SecretRef انجام می‌شوند. این مسیر پیکربندی و ورودی‌های ممیزی را اعتبارسنجی می‌کند. این
    شاهد پشتیبان برای گیت/عملیات است، نه مدرکی برای راه‌اندازی اولیهٔ تعاملی یا
    عامل/ابزار/تأیید OpenClaw. همین مسیر در QA Lab با
    `pnpm openclaw qa suite --scenario system-agent-ring-zero-setup` ارائه می‌شود.
- آزمون دود هزینهٔ Moonshot/Kimi: با تنظیم `MOONSHOT_API_KEY`،
  `openclaw models list --provider moonshot --json` را اجرا کنید، سپس یک
  `openclaw agent --local --session-id live-kimi-cost --message 'Reply exactly: KIMI_LIVE_OK' --thinking off --json` ایزوله را
  در برابر `moonshot/kimi-k2.6` اجرا کنید. تأیید کنید JSON، Moonshot/K2.6 را گزارش می‌دهد و
  رونوشت دستیار، `usage.cost` نرمال‌شده را ذخیره می‌کند.

<Tip>
وقتی فقط به یک مورد ناموفق نیاز دارید، محدود کردن آزمون‌های زنده از طریق متغیرهای محیطی فهرست مجاز که در ادامه توضیح داده شده‌اند را ترجیح دهید.
</Tip>

## اجراکننده‌های ویژهٔ QA

وقتی به واقع‌گرایی qa-lab نیاز دارید، این فرمان‌ها در کنار مجموعه‌های اصلی آزمون قرار می‌گیرند.

CI، QA Lab را در گردش‌کارهای اختصاصی اجرا می‌کند. هم‌ارزی عاملی زیر
`QA-Lab - All Lanes` و اعتبارسنجی انتشار قرار دارد، نه در یک گردش‌کار مستقل PR.
اعتبارسنجی گسترده باید از `Full Release Validation` با
`rerun_group=qa-parity` یا گروه QA بررسی‌های انتشار استفاده کند. بررسی‌های انتشار
پایدار/پیش‌فرض، آزمون فرسایشی جامع زنده/Docker را پشت `run_release_soak=true` نگه می‌دارند؛
پروفایل `full` آزمون فرسایشی را اجباری می‌کند. `QA-Lab - All Lanes` هر شب روی `main` و
از طریق اجرای دستی، با مسیر هم‌ارزی ساختگی، مسیر زندهٔ Matrix،
مسیر زندهٔ Telegram مدیریت‌شده با Convex و مسیر زندهٔ Discord مدیریت‌شده با Convex به‌عنوان
کارهای موازی اجرا می‌شود. QA زمان‌بندی‌شده و بررسی‌های انتشار، پروفایل انتشار Matrix را
از طریق آداپتور زندهٔ مشترک اجرا می‌کنند. مقدار پیش‌فرض CLI مربوط به Matrix و ورودی گردش‌کار دستی
همچنان `all` است؛ اجراهای دستی `all` به پروفایل‌های انتقال، رسانه و
E2EE منشعب می‌شوند، درحالی‌که اجراهای متمرکز می‌توانند `fast`، `release` یا
`transport` را انتخاب کنند. `OpenClaw Release Checks` پیش از تأیید انتشار، هم‌ارزی را به‌همراه پروفایل قابل‌استفادهٔ مجدد
آداپتور زندهٔ Matrix و مسیر Telegram اجرا می‌کند. بررسی‌های انتقال انتشار از
`mock-openai/gpt-5.6-luna` استفاده می‌کنند تا قطعی باقی بمانند و از راه‌اندازی معمول
Plugin ارائه‌دهنده جلوگیری کنند. این Gatewayهای انتقال زنده
جست‌وجوی حافظه را غیرفعال می‌کنند؛ رفتار حافظه همچنان توسط مجموعه‌های هم‌ارزی QA پوشش داده می‌شود.

شاردهای کامل رسانهٔ زندهٔ انتشار از
`ghcr.io/openclaw/openclaw-live-media-runner:ubuntu-24.04` استفاده می‌کنند که از قبل
`ffmpeg` و `ffprobe` را دارد. شاردهای مدل/backend زندهٔ Docker از تصویر مشترک
`ghcr.io/openclaw/openclaw-live-test:<sha>` استفاده می‌کنند که برای هر commit انتخاب‌شده یک‌بار ساخته می‌شود،
سپس به‌جای ساخت مجدد در هر شارد، آن را با `OPENCLAW_SKIP_DOCKER_BUILD=1` دریافت می‌کنند.

- `pnpm openclaw qa suite`
  - سناریوهای QA مبتنی بر مخزن را مستقیماً روی میزبان اجرا می‌کند.
  - برای مجموعه سناریوی انتخاب‌شده، آرتیفکت‌های سطح‌بالای `qa-evidence.json`، `qa-suite-summary.json` و
    `qa-suite-report.md` را می‌نویسد که شامل انتخاب سناریوهای
    جریان ترکیبی، Vitest و Playwright است.
  - هنگامی که توسط `pnpm openclaw qa run --qa-profile <profile>` اجرا شود،
    کارت امتیاز پروفایل رده‌بندی انتخاب‌شده را در همان `qa-evidence.json` جاسازی می‌کند.
    `smoke-ci` شواهد کم‌حجم می‌نویسد (`evidenceMode: "slim"`، بدون
    `execution` برای هر مدخل). `release` بخش گزینش‌شده آمادگی انتشار را پوشش می‌دهد؛ `all`
    همه دسته‌های فعال بلوغ را انتخاب می‌کند و هنگامی که آرتیفکت کامل کارت امتیاز لازم باشد،
    اجرای صریح گردش‌کار شواهد پروفایل QA را هدف می‌گیرد.
  - به‌طور پیش‌فرض، چند سناریوی انتخاب‌شده را با workerهای
    gateway ایزوله به‌صورت موازی اجرا می‌کند. مقدار پیش‌فرض هم‌روندی `qa-channel` برابر 4 است (محدود به
    تعداد سناریوهای انتخاب‌شده). برای تنظیم تعداد workerها از `--concurrency <count>`
    یا برای مسیر سری قدیمی‌تر از `--concurrency 1` استفاده کنید.
  - اگر هر سناریویی ناموفق باشد، با کد غیرصفر خارج می‌شود. برای تولید
    آرتیفکت‌ها بدون کد خروج ناموفق، از `--allow-failures` استفاده کنید.
  - از حالت‌های ارائه‌دهنده `live-frontier`، `mock-openai` و `aimock` پشتیبانی می‌کند.
    `aimock` یک سرور ارائه‌دهنده محلی مبتنی بر AIMock را برای پوشش آزمایشی
    fixture و شبیه‌سازی پروتکل راه‌اندازی می‌کند، بدون آن‌که جایگزین مسیر
    آگاه از سناریوی `mock-openai` شود.
- `pnpm openclaw qa coverage --match <query>`
  - شناسه‌ها، عنوان‌ها، سطوح، شناسه‌های پوشش، ارجاعات مستندات، ارجاعات کد،
    Pluginها و الزامات ارائه‌دهنده سناریوها را جست‌وجو می‌کند و سپس اهداف
    مجموعه منطبق را چاپ می‌کند.
  - پیش از اجرای QA Lab، وقتی رفتار یا مسیر فایل تغییرکرده را می‌دانید
    اما کوچک‌ترین سناریو را نمی‌شناسید، از این استفاده کنید. صرفاً راهنما است؛ همچنان بر اساس
    رفتار در حال تغییر، بین اثبات mock، زنده، Multipass، Matrix یا انتقال
    انتخاب کنید.
- `pnpm test:plugins:kitchen-sink-live`
  - مجموعه آزمون دشوار Plugin زنده OpenAI Kitchen Sink را از طریق QA Lab اجرا می‌کند.
    بسته خارجی Kitchen Sink را نصب می‌کند، فهرست سطوح SDK مربوط به Plugin را
    تأیید می‌کند، `/healthz` و `/readyz` را می‌آزماید، شواهد CPU/RSS مربوط به gateway را
    ثبت می‌کند، یک نوبت زنده OpenAI را اجرا می‌کند و تشخیص‌های
    خصمانه را بررسی می‌کند. به احراز هویت زنده OpenAI مانند `OPENAI_API_KEY` نیاز دارد. در
    نشست‌های Testbox دارای داده‌های احراز هویت، هنگامی که کمک‌کننده `openclaw-testbox-env`
    موجود باشد، پروفایل احراز هویت زنده Testbox را به‌طور خودکار بارگذاری می‌کند.
- `pnpm test:gateway:cpu-scenarios`
  - بنچ راه‌اندازی gateway را همراه با یک بسته کوچک از سناریوهای mock در QA Lab
    (`channel-chat-baseline`، `memory-failure-fallback`،
    `gateway-restart-inflight-run`) اجرا می‌کند و خلاصه ترکیبی مشاهده CPU را
    در `.artifacts/gateway-cpu-scenarios/` می‌نویسد.
  - به‌طور پیش‌فرض فقط مشاهده‌های مداوم CPU داغ را علامت‌گذاری می‌کند (`--cpu-core-warn`،
    با مقدار پیش‌فرض `0.9`؛ `--hot-wall-warn-ms`، با مقدار پیش‌فرض `30000`)؛ بنابراین جهش‌های کوتاه
    هنگام راه‌اندازی به‌عنوان معیار ثبت می‌شوند، بدون آن‌که شبیه رگرسیون
    چنددقیقه‌ای اشغال gateway به نظر برسند.
  - در برابر آرتیفکت‌های ساخته‌شده `dist` اجرا می‌شود؛ اگر checkout
    از قبل خروجی تازه زمان اجرا ندارد، ابتدا build را اجرا کنید.
- `pnpm openclaw qa suite --runner multipass`
  - همان مجموعه QA را داخل یک ماشین مجازی Linux یک‌بارمصرف Multipass اجرا می‌کند و
    همان پرچم‌های انتخاب سناریو و ارائه‌دهنده/مدل `qa suite` را حفظ می‌کند.
  - اجراهای زنده، ورودی‌های احراز هویت QA قابل‌استفاده برای مهمان را ارسال می‌کنند:
    کلیدهای ارائه‌دهنده مبتنی بر env، مسیر پیکربندی ارائه‌دهنده زنده QA و
    `CODEX_HOME` در صورت وجود.
  - دایرکتوری‌های خروجی باید زیر ریشه مخزن باقی بمانند تا مهمان بتواند
    از طریق فضای کاری mountشده در آن‌ها بنویسد.
  - گزارش و خلاصه معمول QA را همراه با لاگ‌های Multipass در
    `.artifacts/qa-e2e/...` می‌نویسد.
- `pnpm qa:lab:up`
  - سایت QA مبتنی بر Docker را برای کار QA به‌سبک اپراتور راه‌اندازی می‌کند.
- `pnpm test:docker:npm-onboard-channel-agent`
  - از checkout فعلی یک tarball مربوط به npm می‌سازد، آن را به‌صورت سراسری در
    Docker نصب می‌کند، راه‌اندازی غیرتعاملی کلید API مربوط به OpenAI را اجرا می‌کند، به‌طور پیش‌فرض
    Telegram را پیکربندی می‌کند، تأیید می‌کند که زمان اجرای بسته‌بندی‌شده Plugin بدون
    ترمیم وابستگی هنگام راه‌اندازی بارگذاری می‌شود، doctor را اجرا می‌کند و یک نوبت عامل محلی را
    در برابر endpoint شبیه‌سازی‌شده OpenAI اجرا می‌کند.
  - برای اجرای همان مسیر نصب بسته‌بندی‌شده
    با Discord، از `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` استفاده کنید.
- `pnpm test:docker:session-runtime-context`
  - یک smoke قطعی Docker را برای transcriptهای context زمان اجرای برنامه ساخته‌شده اجرا می‌کند.
    تأیید می‌کند که context پنهان زمان اجرای OpenClaw به‌عنوان یک پیام سفارشی
    غیرقابل‌نمایش باقی می‌ماند و به نوبت قابل‌مشاهده کاربر نشت نمی‌کند، سپس یک JSONL
    نشست خرابِ تحت‌تأثیر را مقداردهی می‌کند و تأیید می‌کند که
    `openclaw doctor --fix` آن را همراه با یک نسخه پشتیبان روی شاخه فعال بازنویسی می‌کند.
- `pnpm test:docker:npm-telegram-live`
  - یک نامزد بسته OpenClaw را در Docker نصب می‌کند، راه‌اندازی بسته نصب‌شده را
    اجرا می‌کند، Telegram را از طریق CLI نصب‌شده پیکربندی می‌کند و سپس مسیر زنده QA مربوط به
    Telegram را با همان بسته نصب‌شده به‌عنوان Gateway سامانه تحت آزمون دوباره استفاده می‌کند.
  - wrapper فقط منبع harness در `qa-lab` را از checkout mount می‌کند؛
    بسته نصب‌شده مالک `dist`، `openclaw/plugin-sdk` و زمان اجرای
    Pluginهای همراه است؛ بنابراین این مسیر، Pluginهای checkout فعلی را با
    بسته تحت آزمون ترکیب نمی‌کند.
  - مقدار پیش‌فرض `OPENCLAW_NPM_TELEGRAM_PACKAGE_SPEC=openclaw@beta` است؛ برای آزمودن یک tarball محلی resolveشده
    به‌جای نصب از registry، `OPENCLAW_NPM_TELEGRAM_PACKAGE_TGZ=/path/to/openclaw-current.tgz` یا
    `OPENCLAW_CURRENT_PACKAGE_TGZ` را تنظیم کنید.
  - به‌طور پیش‌فرض با `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES=20` زمان‌بندی تکرارشونده RTT را در
    `qa-evidence.json` منتشر می‌کند. برای تنظیم اجرا،
    `OPENCLAW_NPM_TELEGRAM_RTT_SAMPLES`،
    `OPENCLAW_NPM_TELEGRAM_RTT_TIMEOUT_MS` یا
    `OPENCLAW_NPM_TELEGRAM_RTT_MAX_FAILURES` را بازنویسی کنید.
    `OPENCLAW_NPM_TELEGRAM_RTT_CHECKS` سناریوی QA مربوط به Telegram را برای
    نمونه‌برداری انتخاب می‌کند؛ هدف RTT پشتیبانی‌شده `channel-canary` است.
  - از همان اعتبارنامه‌های env مربوط به Telegram یا منبع اعتبارنامه Convex در
    `pnpm openclaw qa telegram` استفاده می‌کند. برای خودکارسازی CI/انتشار،
    `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_SOURCE=convex` را همراه با
    `OPENCLAW_QA_CONVEX_SITE_URL` و یک secret نقش تنظیم کنید. اگر
    `OPENCLAW_QA_CONVEX_SITE_URL` و یک secret نقش Convex در
    CI موجود باشند، wrapper مربوط به Docker به‌طور خودکار Convex را انتخاب می‌کند.
  - wrapper پیش از کار build/install در Docker، env اعتبارنامه Telegram یا Convex را
    روی میزبان اعتبارسنجی می‌کند. `OPENCLAW_NPM_TELEGRAM_SKIP_CREDENTIAL_PREFLIGHT=1` را فقط هنگامی تنظیم کنید که
    عمداً در حال اشکال‌زدایی تنظیمات پیش از اعتبارنامه هستید.
  - `OPENCLAW_NPM_TELEGRAM_CREDENTIAL_ROLE=ci|maintainer` مقدار مشترک
    `OPENCLAW_QA_CREDENTIAL_ROLE` را فقط برای این مسیر بازنویسی می‌کند. وقتی اعتبارنامه‌های
    Convex انتخاب شده‌اند و هیچ نقشی تنظیم نشده است، wrapper در CI از `ci`
    و خارج از CI از `maintainer` استفاده می‌کند.
  - GitHub Actions این مسیر را به‌عنوان گردش‌کار دستی نگه‌دارنده
    `NPM Telegram Beta E2E` ارائه می‌کند. هنگام merge اجرا نمی‌شود. این گردش‌کار از
    محیط `qa-live-shared` و اجاره‌های اعتبارنامه CI مربوط به Convex استفاده می‌کند.
- GitHub Actions همچنین `Package Acceptance` را برای اثبات جانبی محصول
  در برابر یک بسته نامزد ارائه می‌کند. این گردش‌کار یک Git ref، مشخصه منتشرشده npm،
  نشانی URL مربوط به tarball از طریق HTTPS به‌همراه SHA-256، سیاست URL مورداعتماد یا آرتیفکت tarball
  از اجرای دیگری (`source=ref|npm|url|trusted-url|artifact`) را می‌پذیرد،
  `openclaw-current.tgz` نرمال‌شده را با نام `package-under-test` بارگذاری می‌کند و سپس زمان‌بند
  E2E موجود Docker را با پروفایل‌های مسیر `smoke`، `package`، `product`، `full`
  یا `custom` اجرا می‌کند. برای اجرای گردش‌کار QA مربوط به Telegram در برابر همان
  آرتیفکت `package-under-test`، مقدار `telegram_mode=mock-openai` یا
  `live-frontier` را تنظیم کنید.
  - اثبات محصول آخرین نسخه بتا:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=npm \
  -f package_spec=openclaw@beta \
  -f suite_profile=product \
  -f telegram_mode=mock-openai
```

- اثبات دقیق نشانی URL مربوط به tarball به digest نیاز دارد و از سیاست ایمنی URL عمومی استفاده می‌کند:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=url \
  -f package_url=https://registry.npmjs.org/openclaw/-/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

- mirrorهای tarball سازمانی/خصوصی از یک سیاست صریح منبع مورداعتماد استفاده می‌کنند:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=trusted-url \
  -f trusted_source_id=enterprise-artifactory \
  -f package_url=https://packages.example.internal:8443/artifactory/openclaw/openclaw-VERSION.tgz \
  -f package_sha256=<sha256> \
  -f suite_profile=package
```

`source=trusted-url` مقدار `.github/package-trusted-sources.json` را از ref گردش‌کار مورداعتماد می‌خواند و اعتبارنامه‌های URL یا دورزدن شبکه خصوصی از طریق ورودی گردش‌کار را نمی‌پذیرد. اگر سیاست نام‌برده احراز هویت bearer را اعلام می‌کند، secret ثابت `OPENCLAW_TRUSTED_PACKAGE_TOKEN` را پیکربندی کنید.

- اثبات آرتیفکت، یک آرتیفکت tarball را از اجرای دیگری در Actions دانلود می‌کند:

```bash
gh workflow run package-acceptance.yml --ref main \
  -f source=artifact \
  -f artifact_run_id=<run-id> \
  -f artifact_name=<artifact-name> \
  -f suite_profile=smoke
```

- `pnpm test:docker:plugins`
  - build فعلی OpenClaw را در Docker بسته‌بندی و نصب می‌کند، Gateway را با
    پیکربندی OpenAI راه‌اندازی می‌کند و سپس channel/Pluginهای همراه را از طریق
    ویرایش‌های پیکربندی فعال می‌کند.
  - تأیید می‌کند که کشف راه‌اندازی، Pluginهای دانلودشدنی پیکربندی‌نشده را
    غایب نگه می‌دارد، نخستین ترمیم doctor پس از پیکربندی هر Plugin دانلودشدنی گمشده را
    صریحاً نصب می‌کند و راه‌اندازی مجدد دوم، ترمیم پنهان وابستگی را
    اجرا نمی‌کند.
  - همچنین یک baseline قدیمی و شناخته‌شده npm را نصب می‌کند، پیش از اجرای
    `openclaw update --tag <candidate>`، Telegram را فعال می‌کند و تأیید می‌کند که
    doctor پس از به‌روزرسانی نامزد، بقایای وابستگی قدیمی Plugin را
    بدون ترمیم postinstall در سمت harness پاک می‌کند.
- `pnpm test:parallels:npm-update`
  - smoke بومی به‌روزرسانی نصب بسته‌بندی‌شده را در مهمان‌های Parallels اجرا می‌کند.
    هر پلتفرم انتخاب‌شده ابتدا بسته baseline درخواستی را نصب می‌کند،
    سپس فرمان نصب‌شده `openclaw update` را در همان مهمان اجرا می‌کند و
    نسخه نصب‌شده، وضعیت به‌روزرسانی، آمادگی gateway و
    یک نوبت عامل محلی را تأیید می‌کند.
  - هنگام تکرار روی یک مهمان، از `--platform macos`، `--platform windows` یا `--platform linux`
    استفاده کنید. برای مسیر آرتیفکت خلاصه و وضعیت هر مسیر از `--json` استفاده کنید.
  - مسیر OpenAI به‌طور پیش‌فرض برای اثبات نوبت زنده عامل از `openai/gpt-5.6-luna` استفاده می‌کند.
    برای اعتبارسنجی مدل دیگری از OpenAI، `--model <provider/model>` را ارسال یا
    `OPENCLAW_PARALLELS_OPENAI_MODEL` را تنظیم کنید.
  - اجراهای طولانی محلی را در timeout میزبان قرار دهید تا توقف‌های انتقال
    Parallels نتوانند باقی‌مانده بازه آزمون را مصرف کنند:

    ```bash
    timeout --foreground 150m pnpm test:parallels:npm-update -- --json
    timeout --foreground 90m pnpm test:parallels:npm-update -- --platform windows --json
    ```

  - اسکریپت، لاگ‌های تو‌در‌توی مسیرها را در
    `/tmp/openclaw-parallels-npm-update.*` می‌نویسد. پیش از آن‌که فرض کنید wrapper بیرونی
    متوقف شده است، `windows-update.log`،
    `macos-update.log` یا `linux-update.log` را بررسی کنید.
  - به‌روزرسانی Windows روی یک مهمان سرد ممکن است 10 تا 15 دقیقه صرف doctor پس از
    به‌روزرسانی و کار به‌روزرسانی بسته کند؛ تا زمانی که لاگ اشکال‌زدایی تو‌در‌توی npm
    در حال پیشروی است، این وضعیت همچنان سالم است.
  - این wrapper تجمیعی را هم‌زمان با مسیرهای منفرد smoke مربوط به macOS،
    Windows یا Linux در Parallels اجرا نکنید. آن‌ها وضعیت ماشین مجازی را به‌اشتراک می‌گذارند و ممکن است
    هنگام بازیابی snapshot، ارائه بسته یا وضعیت gateway مهمان با یکدیگر
    تداخل کنند.
  - اثبات پس از به‌روزرسانی، سطح معمول Pluginهای همراه را اجرا می‌کند، زیرا
    facadeهای قابلیت مانند گفتار، تولید تصویر و درک رسانه
    از طریق APIهای زمان اجرای همراه بارگذاری می‌شوند، حتی هنگامی که خود نوبت عامل
    فقط یک پاسخ متنی ساده را بررسی می‌کند.

- `pnpm openclaw qa aimock`
  - فقط سرور محلی ارائه‌دهنده AIMock را برای آزمون دود مستقیم پروتکل
    راه‌اندازی می‌کند.
- `pnpm openclaw qa matrix`
  - مسیر QA زنده Matrix را در برابر یک homeserver موقت Tuwunel با پشتوانه Docker
    اجرا می‌کند. فقط برای checkout کد منبع است — نصب‌های بسته‌بندی‌شده
    `qa-lab` را ارائه نمی‌کنند.
  - CLI کامل، کاتالوگ پروفایل/سناریو، متغیرهای محیطی و چیدمان آرتیفکت:
    [مسیرهای آزمون دود Matrix](/fa/concepts/qa-e2e-automation#matrix-smoke-lanes).
- `pnpm openclaw qa telegram`
  - مسیر QA زنده Telegram را در برابر یک گروه خصوصی واقعی، با استفاده از
    توکن‌های ربات درایور و SUT از محیط اجرا می‌کند.
  - به `OPENCLAW_QA_TELEGRAM_GROUP_ID`،
    `OPENCLAW_QA_TELEGRAM_DRIVER_BOT_TOKEN` و
    `OPENCLAW_QA_TELEGRAM_SUT_BOT_TOKEN` نیاز دارد. شناسه گروه باید شناسه عددی
    گفت‌وگوی Telegram باشد.
  - برای اعتبارنامه‌های تجمیعی اشتراکی از `--credential-source convex` پشتیبانی می‌کند.
    به‌طور پیش‌فرض از حالت محیط استفاده کنید، یا برای انتخاب اجاره‌های تجمیعی
    `OPENCLAW_QA_CREDENTIAL_SOURCE=convex` را تنظیم کنید.
  - پیش‌فرض‌ها canary، دروازه‌گذاری اشاره، آدرس‌دهی فرمان، `/status`،
    پاسخ‌های اشاره‌شده ربات‌به‌ربات و پاسخ‌های فرمان بومی هسته را پوشش می‌دهند.
    پیش‌فرض‌های `mock-openai` رگرسیون‌های زنجیره پاسخ قطعی و
    پخش جریانی پیام نهایی Telegram را نیز پوشش می‌دهند. برای کاوش‌های اختیاری مانند
    `session_status` از `--list-scenarios` استفاده کنید.
  - اگر هر سناریویی شکست بخورد، با کد غیرصفر خارج می‌شود. برای تولید
    آرتیفکت‌ها بدون کد خروج شکست، از `--allow-failures` استفاده کنید.
  - به دو ربات متمایز در یک گروه خصوصی واحد نیاز دارد و ربات SUT باید
    یک نام کاربری Telegram ارائه کند.
  - برای مشاهده پایدار ربات‌به‌ربات، حالت Bot-to-Bot Communication Mode را
    در `@BotFather` برای هر دو ربات فعال کنید و مطمئن شوید ربات درایور می‌تواند
    ترافیک ربات‌های گروه را مشاهده کند.
  - یک گزارش QA مربوط به Telegram، خلاصه و `qa-evidence.json` را زیر
    `.artifacts/qa-e2e/...` می‌نویسد. سناریوهای پاسخ‌دهی شامل RTT از درخواست ارسال
    درایور تا پاسخ مشاهده‌شده SUT هستند.

`Mantis Telegram Live` پوشش شواهد PR پیرامون این مسیر است. این پوشش،
رف کاندید را با اعتبارنامه‌های Telegram اجاره‌شده از Convex اجرا می‌کند، بسته
گزارش/شواهد QA پاک‌سازی‌شده را در مرورگر دسکتاپ Crabbox رندر می‌کند، شواهد MP4
را ضبط می‌کند، یک GIF برش‌خورده بر اساس حرکت تولید می‌کند، بسته آرتیفکت را بارگذاری می‌کند و
هنگامی که `pr_number` تنظیم شده باشد، شواهد درون‌خطی PR را از طریق Mantis GitHub App
ارسال می‌کند. نگه‌دارندگان می‌توانند آن را از رابط Actions از طریق `Mantis Scenario`
(`scenario_id: telegram-live`) یا مستقیماً از یک نظر Pull request آغاز کنند:

```text
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
```

`Mantis Telegram Desktop Proof` پوشش عامل‌محور بومی Telegram Desktop
برای شواهد بصری قبل/بعد PR است. آن را از رابط Actions با
`instructions` آزاد، از طریق `Mantis Scenario` (`scenario_id:
telegram-desktop-proof`) یا از یک نظر PR آغاز کنید:

```text
@openclaw-mantis telegram desktop proof
```

عامل Mantis، PR را می‌خواند، تصمیم می‌گیرد چه رفتار قابل‌مشاهده‌ای در Telegram
تغییر را اثبات می‌کند، مسیر اثبات کاربر واقعی Telegram Desktop در Crabbox را روی
رف‌های مبنا و کاندید اجرا می‌کند، تا مفیدشدن GIFهای بومی تکرار می‌کند،
یک مانیفست جفت‌شده `motionPreview` می‌نویسد و هنگامی که
`pr_number` تنظیم شده باشد، همان جدول GIF دو ستونی را از طریق Mantis GitHub App ارسال می‌کند.

- `pnpm openclaw qa mantis telegram-desktop-builder`
  - یک دسکتاپ لینوکس Crabbox را اجاره می‌کند یا دوباره به‌کار می‌گیرد، Telegram
    Desktop بومی را نصب می‌کند، OpenClaw را با توکن اجاره‌شده ربات SUT مربوط به Telegram
    پیکربندی می‌کند، Gateway را راه‌اندازی می‌کند و از دسکتاپ قابل‌مشاهده VNC
    شواهد اسکرین‌شات/MP4 ضبط می‌کند.
  - به‌طور پیش‌فرض `--credential-source convex` است تا گردش‌های کاری فقط به
    رمز broker مربوط به Convex نیاز داشته باشند. از `--credential-source env` با همان
    متغیرهای `OPENCLAW_QA_TELEGRAM_*` مانند `pnpm openclaw qa telegram` استفاده کنید.
  - Telegram Desktop همچنان به ورود/پروفایل کاربر نیاز دارد. توکن ربات
    فقط OpenClaw را پیکربندی می‌کند. از `--telegram-profile-archive-env <name>`
    برای آرشیو پروفایل base64 مربوط به `.tgz` استفاده کنید، یا از `--keep-lease` استفاده کنید و
    یک‌بار به‌صورت دستی از طریق VNC وارد شوید.
  - `mantis-telegram-desktop-builder-report.md`،
    `mantis-telegram-desktop-builder-summary.json`،
    `telegram-desktop-builder.png` و `telegram-desktop-builder.mp4`
    را زیر دایرکتوری خروجی می‌نویسد.

مسیرهای انتقال زنده یک قرارداد استاندارد مشترک دارند تا انتقال‌های جدید
دچار واگرایی نشوند؛ ماتریس پوشش هر مسیر در
[نمای کلی QA — پوشش انتقال زنده](/fa/concepts/qa-e2e-automation#live-transport-coverage) قرار دارد.
`qa-channel` مجموعه مصنوعی گسترده است و بخشی از آن ماتریس نیست.

### اعتبارنامه‌های اشتراکی Telegram از طریق Convex (v1)

هنگامی که `--credential-source convex` (یا `OPENCLAW_QA_CREDENTIAL_SOURCE=convex`)
برای QA انتقال زنده فعال باشد، آزمایشگاه QA یک اجاره انحصاری را از یک
مخزن با پشتوانه Convex دریافت می‌کند، هنگام اجرای مسیر برای آن اجاره Heartbeat
می‌فرستد و هنگام خاموش‌شدن اجاره را آزاد می‌کند. نام این بخش پیش از پشتیبانی از Discord، Slack و
WhatsApp ایجاد شده است؛ قرارداد اجاره میان انواع مشترک است.

داربست مرجع پروژه Convex: `qa/convex-credential-broker/`

متغیرهای محیطی الزامی:

- `OPENCLAW_QA_CONVEX_SITE_URL` (برای مثال `https://your-deployment.convex.site`)
- یک رمز برای نقش انتخاب‌شده:
  - `OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` برای `maintainer`
  - `OPENCLAW_QA_CONVEX_SECRET_CI` برای `ci`
- انتخاب نقش اعتبارنامه:
  - CLI: `--credential-role maintainer|ci`
  - پیش‌فرض محیط: `OPENCLAW_QA_CREDENTIAL_ROLE` (در CI به‌طور پیش‌فرض `ci` و در غیر این صورت `maintainer`)

متغیرهای محیطی اختیاری:

- `OPENCLAW_QA_CREDENTIAL_LEASE_TTL_MS` (پیش‌فرض `1200000`)
- `OPENCLAW_QA_CREDENTIAL_HEARTBEAT_INTERVAL_MS` (پیش‌فرض `30000`)
- `OPENCLAW_QA_CREDENTIAL_ACQUIRE_TIMEOUT_MS` (پیش‌فرض `90000`)
- `OPENCLAW_QA_CREDENTIAL_HTTP_TIMEOUT_MS` (پیش‌فرض `15000`)
- `OPENCLAW_QA_CONVEX_ENDPOINT_PREFIX` (پیش‌فرض `/qa-credentials/v1`)
- `OPENCLAW_QA_CREDENTIAL_OWNER_ID` (شناسه اختیاری ردیابی)
- `OPENCLAW_QA_ALLOW_INSECURE_HTTP=1` به URLهای Convex حلقه‌بازگشت `http://` برای توسعه صرفاً محلی اجازه می‌دهد.

در عملیات عادی، `OPENCLAW_QA_CONVEX_SITE_URL` باید از `https://` استفاده کند.

فرمان‌های مدیریتی نگه‌دارندگان (افزودن/حذف/فهرست‌کردن مخزن) مشخصاً به
`OPENCLAW_QA_CONVEX_SECRET_MAINTAINER` نیاز دارند.

ابزارهای کمکی CLI برای نگه‌دارندگان:

```bash
pnpm openclaw qa credentials doctor
pnpm openclaw qa credentials add --kind telegram --payload-file qa/telegram-credential.json
pnpm openclaw qa credentials list --kind telegram
pnpm openclaw qa credentials remove --credential-id <credential-id>
```

پیش از اجراهای زنده از `doctor` استفاده کنید تا URL سایت Convex، رمزهای broker،
پیشوند نقطه پایانی، مهلت HTTP و دسترسی‌پذیری admin/list را بدون چاپ
مقادیر رمز بررسی کنید. برای خروجی قابل‌خواندن توسط ماشین در اسکریپت‌ها و ابزارهای CI
از `--json` استفاده کنید.

قرارداد پیش‌فرض نقطه پایانی (`OPENCLAW_QA_CONVEX_SITE_URL` + `/qa-credentials/v1`).
درخواست‌ها با یک هدر `Authorization: Bearer <role secret>` احراز هویت می‌شوند؛
بدنه‌های زیر آن هدر را حذف کرده‌اند:

- `POST /acquire`
  - درخواست: `{ kind, ownerId, actorRole, leaseTtlMs, heartbeatIntervalMs }`
  - موفقیت: `{ status: "ok", credentialId, leaseToken, payload, leaseTtlMs?, heartbeatIntervalMs? }`
  - تمام‌شده/قابل‌تلاش مجدد: `{ status: "error", code: "POOL_EXHAUSTED" | "NO_CREDENTIAL_AVAILABLE", ... }`
- `POST /payload-chunk`
  - درخواست: `{ kind, ownerId, actorRole, credentialId, leaseToken, index }`
  - موفقیت: `{ status: "ok", index, data }`
- `POST /heartbeat`
  - درخواست: `{ kind, ownerId, actorRole, credentialId, leaseToken, leaseTtlMs }`
  - موفقیت: `{ status: "ok" }` (یا `2xx` خالی)
- `POST /release`
  - درخواست: `{ kind, ownerId, actorRole, credentialId, leaseToken }`
  - موفقیت: `{ status: "ok" }` (یا `2xx` خالی)
- `POST /admin/add` (فقط رمز نگه‌دارنده)
  - درخواست: `{ kind, actorId, payload, note?, status? }`
  - موفقیت: `{ status: "ok", credential }`
- `POST /admin/remove` (فقط رمز نگه‌دارنده)
  - درخواست: `{ credentialId, actorId }`
  - موفقیت: `{ status: "ok", changed, credential }`
  - محافظ اجاره فعال: `{ status: "error", code: "LEASE_ACTIVE", ... }`
- `POST /admin/list` (فقط رمز نگه‌دارنده)
  - درخواست: `{ kind?, status?, includePayload?, limit? }`
  - موفقیت: `{ status: "ok", credentials, count }`

شکل payload برای نوع Telegram:

- `{ groupId: string, driverToken: string, sutToken: string }`
- `groupId` باید یک رشته شناسه عددی گفت‌وگوی Telegram باشد.
- `admin/add` این شکل را برای `kind: "telegram"` اعتبارسنجی و payloadهای بدشکل را رد می‌کند.

شکل payload برای نوع کاربر واقعی Telegram:

- `{ groupId: string, sutToken: string, testerUserId: string, testerUsername: string, telegramApiId: string, telegramApiHash: string, tdlibDatabaseEncryptionKey: string, tdlibArchiveBase64: string, tdlibArchiveSha256: string, desktopTdataArchiveBase64: string, desktopTdataArchiveSha256: string }`
- `groupId`، `testerUserId` و `telegramApiId` باید رشته‌های عددی باشند.
- `tdlibArchiveSha256` و `desktopTdataArchiveSha256` باید رشته‌های هگز SHA-256 باشند.
- `kind: "telegram-user"` برای گردش کار اثبات Telegram Desktop مربوط به Mantis رزرو شده است. مسیرهای عمومی آزمایشگاه QA هرگز نباید آن را دریافت کنند.

payloadهای چندکاناله اعتبارسنجی‌شده توسط broker:

- Discord: `{ guildId: string, channelId: string, driverBotToken: string, sutBotToken: string, sutApplicationId: string, voiceChannelId?: string }`
- WhatsApp: `{ driverPhoneE164: string, sutPhoneE164: string, driverAuthArchiveBase64: string, sutAuthArchiveBase64: string, groupJid?: string }`

مسیرهای Slack نیز می‌توانند از مخزن اجاره بگیرند، اما اعتبارسنجی payload مربوط به Slack
در حال حاضر به‌جای broker در اجراکننده QA مربوط به Slack قرار دارد. برای ردیف‌های Slack از
`{ channelId: string, driverBotToken: string, sutBotToken: string, sutAppToken: string }`
استفاده کنید.

### افزودن یک کانال به QA

معماری و نام ابزارهای کمکی سناریو برای آداپتورهای کانال جدید در
[نمای کلی QA — افزودن یک کانال](/fa/concepts/qa-e2e-automation#adding-a-channel) قرار دارند.
حداقل معیار: اجراکننده انتقال را روی seam میزبان مشترک `qa-lab`
پیاده‌سازی کنید، یک `adapterFactory` برای سناریوهای مشترک اضافه کنید، `qaRunners` را در
مانیفست Plugin اعلام کنید، به‌صورت `openclaw qa <runner>` mount کنید و سناریوها را زیر
`qa/scenarios/` بنویسید.

## مجموعه‌های آزمون (چه چیزی کجا اجرا می‌شود)

مجموعه‌ها را «واقع‌گرایی فزاینده» (و بی‌ثباتی/هزینه فزاینده) در نظر بگیرید.

### واحد / یکپارچه‌سازی (پیش‌فرض)

- فرمان: `pnpm test`
- پیکربندی: اجراهای بدون هدف از مجموعه shard مربوط به `vitest.full-*.config.ts` استفاده می‌کنند و ممکن است
  shardهای چندپروژه‌ای را برای زمان‌بندی موازی به پیکربندی‌های هر پروژه
  گسترش دهند
- فایل‌ها: موجودی‌های هسته/واحد زیر `src/**/*.test.ts`،
  `packages/**/*.test.ts` و `test/**/*.test.ts`؛ آزمون‌های واحد UI در
  shard اختصاصی `unit-ui` اجرا می‌شوند
- دامنه:
  - آزمون‌های واحد خالص
  - آزمون‌های یکپارچه‌سازی درون‌فرایندی (احراز هویت Gateway، مسیریابی، ابزارها، تجزیه، پیکربندی)
  - رگرسیون‌های قطعی برای باگ‌های شناخته‌شده
- انتظارات:
  - در CI اجرا می‌شود
  - به کلیدهای واقعی نیاز ندارد
  - باید سریع و پایدار باشد
  - آزمون‌های resolver و بارگذار سطح عمومی باید رفتار fallback گسترده `api.js` و
    `runtime-api.js` را با fixtureهای کوچک تولیدشده Plugin اثبات کنند،
    نه APIهای کد منبع Pluginهای همراه واقعی. بارگذاری APIهای Plugin واقعی به
    مجموعه‌های قرارداد/یکپارچه‌سازی تحت مالکیت Plugin تعلق دارد.

سیاست وابستگی بومی:

- نصب‌های آزمون پیش‌فرض، ساخت‌های اختیاری بومی opus مربوط به Discord را رد می‌کنند. صدای Discord
  از `libopus-wasm` همراه استفاده می‌کند و `@discordjs/opus` در
  `allowBuilds` غیرفعال می‌ماند تا آزمون‌های محلی و مسیرهای Testbox افزونه بومی
  را کامپایل نکنند.
- عملکرد opus بومی را در مخزن بنچمارک `libopus-wasm` مقایسه کنید، نه
  در حلقه‌های پیش‌فرض نصب/آزمون OpenClaw. در `allowBuilds` پیش‌فرض،
  `@discordjs/opus` را روی `true` تنظیم نکنید؛ این کار باعث می‌شود حلقه‌های نامرتبط نصب/آزمون
  کد بومی را کامپایل کنند.

<AccordionGroup>
  <Accordion title="پروژه‌ها، shardها و مسیرهای محدودشده">

    - اجرای بدون هدف `pnpm test` به‌جای یک فرایند بومی غول‌پیکر برای پروژهٔ ریشه، سیزده پیکربندی شارد کوچک‌تر (`core-unit-fast`، `core-unit-src`، `core-unit-security`، `core-unit-ui`، `core-unit-support`، `core-support-boundary`، `core-tooling`، `core-contracts`، `core-bundled`، `core-runtime`، `agentic`، `auto-reply`، `extensions`) را اجرا می‌کند. این کار اوج RSS را در ماشین‌های پربار کاهش می‌دهد و مانع از آن می‌شود که کارهای پاسخ خودکار/Plugin، مجموعه‌های نامرتبط را از منابع محروم کنند.
    - `pnpm test --watch` همچنان از گراف بومی پروژهٔ ریشهٔ `vitest.config.ts` استفاده می‌کند، زیرا حلقهٔ پایش چندشارده عملی نیست.
    - `pnpm test`، `pnpm test:watch` و `pnpm test:perf:imports` ابتدا اهداف صریح فایل/دایرکتوری را از مسیرهای محدود عبور می‌دهند، تا `pnpm test extensions/discord/src/monitor/message-handler.preflight.test.ts` هزینهٔ کامل راه‌اندازی پروژهٔ ریشه را نپردازد.
    - `pnpm test:changed` به‌طور پیش‌فرض مسیرهای تغییریافتهٔ git را به مسیرهای محدود و کم‌هزینه گسترش می‌دهد: ویرایش‌های مستقیم آزمون، فایل‌های هم‌جوار `*.test.ts`، نگاشت‌های صریح منبع و وابسته‌های گراف واردسازی محلی. ویرایش‌های پیکربندی/راه‌اندازی/بسته، آزمون‌ها را به‌صورت گسترده اجرا نمی‌کنند، مگر اینکه صراحتاً از `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` استفاده کنید.
    - `pnpm check:changed` دروازهٔ معمول و هوشمند بررسی محلی برای کارهای محدود است. این دستور تفاوت‌ها را به هسته، آزمون‌های هسته، افزونه‌ها، آزمون‌های افزونه، برنامه‌ها، مستندات، فرادادهٔ انتشار، ابزارهای زندهٔ Docker و ابزارسازی دسته‌بندی می‌کند، سپس فرمان‌های بررسی نوع، lint و محافظ متناظر را اجرا می‌کند. آزمون‌های Vitest را اجرا نمی‌کند؛ برای ارائهٔ مدرک آزمون، `pnpm test:changed` یا `pnpm test <target>` صریح را فراخوانی کنید. افزایش نسخه‌هایی که فقط فرادادهٔ انتشار را تغییر می‌دهند، بررسی‌های هدفمند نسخه/پیکربندی/وابستگی ریشه را اجرا می‌کنند و محافظی دارند که تغییرات بسته خارج از فیلد نسخهٔ سطح بالا را رد می‌کند.
    - ویرایش‌های هارنس زندهٔ Docker برای ACP بررسی‌های متمرکز اجرا می‌کنند: نحو پوسته برای اسکریپت‌های احراز هویت زندهٔ Docker و یک اجرای آزمایشی زمان‌بند زندهٔ Docker. تغییرات `package.json` فقط زمانی لحاظ می‌شوند که تفاوت به `scripts["test:docker:live-*"]` محدود باشد؛ ویرایش‌های وابستگی، خروجی، نسخه و سایر سطوح بسته همچنان از محافظ‌های گسترده‌تر استفاده می‌کنند.
    - آزمون‌های واحد با واردسازی سبک از عامل‌ها، فرمان‌ها، Pluginها، ابزارهای کمکی پاسخ خودکار، `plugin-sdk` و بخش‌های مشابهِ ابزارهای خالص، از مسیر `unit-fast` عبور می‌کنند که `test/setup-openclaw-runtime.ts` را رد می‌کند؛ فایل‌های حالت‌مند/سنگین از نظر زمان اجرا در مسیرهای موجود باقی می‌مانند.
    - فایل‌های منبع کمکی منتخب `plugin-sdk` و `commands` نیز اجراهای حالت تغییر را به آزمون‌های هم‌جوار صریح در همان مسیرهای سبک نگاشت می‌کنند، تا ویرایش ابزارهای کمکی باعث اجرای دوبارهٔ کل مجموعهٔ سنگین آن دایرکتوری نشود.
    - `auto-reply` سطل‌های اختصاصی برای ابزارهای کمکی سطح بالای هسته، آزمون‌های یکپارچه‌سازی سطح بالای `reply.*` و زیردرخت `src/auto-reply/reply/**` دارد. CI زیردرخت پاسخ را نیز به شاردهای اجراکنندهٔ عامل، ارسال و فرمان‌ها/مسیریابی حالت تقسیم می‌کند تا یک سطل با واردسازی سنگین، کل دنبالهٔ Node را در اختیار نگیرد.
    - پایپ‌لاین CI معمول PR/main عمداً پیمایش دسته‌ای Pluginهای همراه و شارد صرفاً انتشاری `agentic-plugins` را نادیده می‌گیرد. اعتبارسنجی کامل انتشار، گردش‌کار فرزند جداگانهٔ `Plugin Prerelease` را برای آن مجموعه‌های سنگین از نظر Plugin روی نامزدهای انتشار اجرا می‌کند.

  </Accordion>

  <Accordion title="پوشش اجراکنندهٔ تعبیه‌شده">

    - هنگام تغییر ورودی‌های کشف ابزار پیام یا زمینهٔ زمان اجرای Compaction،
      هر دو سطح پوشش را حفظ کنید.
    - برای مرزهای خالص مسیریابی و نرمال‌سازی، آزمون‌های رگرسیون متمرکزِ ابزارهای کمکی
      اضافه کنید.
    - مجموعه‌های یکپارچه‌سازی اجراکنندهٔ تعبیه‌شده را سالم نگه دارید:
      `src/agents/embedded-agent-runner/compact.hooks.test.ts`،
      `src/agents/embedded-agent-runner/run.overflow-compaction.test.ts` و
      `src/agents/embedded-agent-runner/run.overflow-compaction.loop.test.ts`.
    - این مجموعه‌ها تأیید می‌کنند که شناسه‌های محدود و رفتار Compaction همچنان
      از مسیرهای واقعی `run.ts` / `compact.ts` عبور می‌کنند؛ آزمون‌های صرفاً
      کمکی جایگزین کافی برای آن مسیرهای یکپارچه‌سازی نیستند.

  </Accordion>

  <Accordion title="پیش‌فرض‌های مخزن و جداسازی Vitest">

    - پیکربندی پایهٔ Vitest به‌طور پیش‌فرض از `threads` استفاده می‌کند.
    - پیکربندی مشترک Vitest، `isolate: false` را ثابت می‌کند و در پروژه‌های ریشه،
      پیکربندی‌های e2e و زنده از اجراکنندهٔ غیرایزوله استفاده می‌کند.
    - مسیر رابط کاربری ریشه، راه‌اندازی و بهینه‌ساز `jsdom` خود را حفظ می‌کند،
      اما آن نیز روی اجراکنندهٔ مشترک غیرایزوله اجرا می‌شود.
    - هر شارد `pnpm test` همان پیش‌فرض‌های `threads` + `isolate: false`
      را از پیکربندی مشترک Vitest به ارث می‌برد.
    - `scripts/run-vitest.mjs` به‌طور پیش‌فرض `--no-maglev` را برای فرایندهای فرزند Node
      در Vitest اضافه می‌کند تا سربار کامپایل V8 در اجراهای بزرگ محلی کاهش یابد.
      برای مقایسه با رفتار استاندارد V8، `OPENCLAW_VITEST_ENABLE_MAGLEV=1` را تنظیم کنید.
    - `scripts/run-vitest.mjs` اجراهای صریح و غیرپایشی Vitest را
      پس از 5 دقیقه بدون هیچ خروجی stdout یا stderr خاتمه می‌دهد. برای غیرفعال‌کردن
      ناظر در یک بررسی عمداً بی‌صدا، `OPENCLAW_VITEST_NO_OUTPUT_TIMEOUT_MS=0` را تنظیم کنید.

  </Accordion>

  <Accordion title="تکرار سریع محلی">

    - `pnpm changed:lanes` نشان می‌دهد یک تفاوت کدام مسیرهای معماری را فعال می‌کند.
    - قلاب پیش از commit فقط قالب‌بندی انجام می‌دهد. فایل‌های قالب‌بندی‌شده را دوباره stage
      می‌کند و lint، بررسی نوع یا آزمون‌ها را اجرا نمی‌کند.
    - هنگامی که به دروازهٔ هوشمند بررسی محلی نیاز دارید، پیش از تحویل یا push،
      `pnpm check:changed` را صراحتاً اجرا کنید.
    - `pnpm test:changed` به‌طور پیش‌فرض از مسیرهای محدود و کم‌هزینه عبور می‌کند. فقط زمانی از
      `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` استفاده کنید که عامل تشخیص دهد ویرایش هارنس، پیکربندی، بسته یا قرارداد
      واقعاً به پوشش گسترده‌تر Vitest نیاز دارد.
    - `pnpm test:max` و `pnpm test:changed:max` همان رفتار مسیریابی را حفظ می‌کنند،
      اما سقف worker بالاتری دارند.
    - مقیاس‌دهی خودکار workerهای محلی عمداً محافظه‌کارانه است و هنگامی که میانگین بار
      میزبان از قبل بالا باشد عقب‌نشینی می‌کند، تا چند اجرای هم‌زمان Vitest به‌طور پیش‌فرض
      آسیب کمتری وارد کنند.
    - پیکربندی پایهٔ Vitest، فایل‌های پروژه/پیکربندی را به‌عنوان
      `forceRerunTriggers` علامت‌گذاری می‌کند تا هنگام تغییر سیم‌کشی آزمون،
      اجرای دوباره در حالت تغییر همچنان درست باشد.
    - پیکربندی، `OPENCLAW_VITEST_FS_MODULE_CACHE` را روی میزبان‌های پشتیبانی‌شده فعال نگه می‌دارد؛
      برای تعیین یک مکان صریح کش جهت پروفایل‌گیری مستقیم،
      `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/abs/path` را تنظیم کنید.

  </Accordion>

  <Accordion title="اشکال‌زدایی کارایی">

    - `pnpm test:perf:imports` گزارش مدت واردسازی Vitest را به‌همراه
      خروجی تفکیک واردسازی فعال می‌کند.
    - `pnpm test:perf:imports:changed` همان نمای پروفایل‌گیری را به
      فایل‌های تغییریافته از زمان `origin/main` محدود می‌کند.
    - داده‌های زمان‌بندی شارد در `.artifacts/vitest-shard-timings.json` نوشته می‌شوند.
      اجراهای کل پیکربندی از مسیر پیکربندی به‌عنوان کلید استفاده می‌کنند؛ شاردهای CI
      دارای الگوی include، نام شارد را اضافه می‌کنند تا شاردهای فیلترشده بتوانند
      جداگانه ردیابی شوند.
    - هنگامی که یک آزمون داغ همچنان بیشتر زمان خود را صرف واردسازی‌های راه‌اندازی می‌کند،
      وابستگی‌های سنگین را پشت یک مرز محلی و محدود `*.runtime.ts` نگه دارید و
      به‌جای واردسازی عمیق ابزارهای کمکی زمان اجرا صرفاً برای عبور دادن آن‌ها از
      `vi.mock(...)`، همان مرز را مستقیماً mock کنید.
    - `pnpm test:perf:changed:bench -- --ref <git-ref>`، `test:changed` مسیریابی‌شده را با
      مسیر بومی پروژهٔ ریشه برای همان تفاوت commit‌شده مقایسه می‌کند و زمان واقعی
      به‌همراه حداکثر RSS در macOS را چاپ می‌کند.
    - `pnpm test:perf:changed:bench -- --worktree` درخت کثیف فعلی را با عبور دادن فهرست فایل‌های
      تغییریافته از `scripts/test-projects.mjs` و پیکربندی ریشهٔ Vitest بنچمارک می‌کند.
    - `pnpm test:perf:profile:main` یک پروفایل CPU از نخ اصلی برای
      سربار راه‌اندازی و تبدیل Vitest/Vite می‌نویسد.
    - `pnpm test:perf:profile:runner` پروفایل‌های CPU+heap اجراکننده را برای
      مجموعهٔ واحد، با موازی‌سازی فایل غیرفعال، می‌نویسد.

  </Accordion>
</AccordionGroup>

### پایداری (Gateway)

- فرمان: `pnpm test:stability:gateway`
- پیکربندی: `test/vitest/vitest.gateway.config.ts`، `test/vitest/vitest.logging.config.ts` و `test/vitest/vitest.infra.config.ts`، هرکدام اجباری با یک worker
- دامنه:
  - یک Gateway واقعی loopback را با عیب‌یابی فعال به‌طور پیش‌فرض راه‌اندازی می‌کند
  - چرخش مصنوعی پیام Gateway، حافظه و payload بزرگ را از مسیر رویداد عیب‌یابی عبور می‌دهد
  - از طریق RPC مبتنی بر WS در Gateway، `diagnostics.stability` را پرس‌وجو می‌کند
  - ابزارهای کمکی ماندگاری بستهٔ پایداری عیب‌یابی را پوشش می‌دهد
  - تأیید می‌کند که ضبط‌کننده محدود می‌ماند، نمونه‌های مصنوعی RSS زیر بودجهٔ فشار باقی می‌مانند و عمق صف هر نشست دوباره به صفر تخلیه می‌شود
- انتظارات:
  - ایمن برای CI و بدون نیاز به کلید
  - مسیری محدود برای پیگیری رگرسیون پایداری، نه جایگزینی برای مجموعهٔ کامل Gateway

### E2E (تجمیع مخزن)

- فرمان: `pnpm test:e2e`
- دامنه:
  - مسیر E2E دودِ Gateway را اجرا می‌کند
  - مسیر E2E مرورگر mock‌شدهٔ رابط کاربری کنترل را اجرا می‌کند
- انتظارات:
  - ایمن برای CI و بدون نیاز به کلید
  - نیازمند نصب بودن Chromium مربوط به Playwright است

### E2E (آزمون دود Gateway)

- فرمان: `pnpm test:e2e:gateway`
- پیکربندی: `test/vitest/vitest.e2e.config.ts`
- فایل‌ها: `src/**/*.e2e.test.ts`، `test/**/*.e2e.test.ts` و آزمون‌های E2E مربوط به Pluginهای همراه در `extensions/`
- پیش‌فرض‌های زمان اجرا:
  - از `threads` در Vitest همراه با `isolate: false` استفاده می‌کند که با بقیهٔ مخزن هم‌خوان است.
  - از workerهای تطبیقی استفاده می‌کند (CI: حداکثر 2، محلی: به‌طور پیش‌فرض 1).
  - برای کاهش سربار ورودی/خروجی کنسول، به‌طور پیش‌فرض در حالت بی‌صدا اجرا می‌شود.
- بازنویسی‌های مفید:
  - `OPENCLAW_E2E_WORKERS=<n>` برای اجبار تعداد workerها (با سقف 16).
  - `OPENCLAW_E2E_VERBOSE=1` برای فعال‌کردن دوبارهٔ خروجی مفصل کنسول.
- دامنه:
  - رفتار سرتاسری Gateway چندنمونه‌ای
  - سطوح WebSocket/HTTP، جفت‌سازی Node و شبکه‌سازی سنگین‌تر
- انتظارات:
  - در CI اجرا می‌شود (هنگامی که در پایپ‌لاین فعال باشد)
  - به کلید واقعی نیاز ندارد
  - اجزای متحرک بیشتری نسبت به آزمون‌های واحد دارد (ممکن است کندتر باشد)

### E2E (مرورگر mock‌شدهٔ رابط کاربری کنترل)

- فرمان: `pnpm test:ui:e2e`
- پیکربندی: `test/vitest/vitest.ui-e2e.config.ts`
- فایل‌ها: `ui/src/**/*.e2e.test.ts`
- دامنه:
  - رابط کاربری کنترل Vite را راه‌اندازی می‌کند
  - یک صفحهٔ واقعی Chromium را از طریق Playwright هدایت می‌کند
  - WebSocket مربوط به Gateway را با mockهای قطعی درون‌مرورگری جایگزین می‌کند
- انتظارات:
  - در CI به‌عنوان بخشی از `pnpm test:e2e` اجرا می‌شود
  - به Gateway واقعی، عامل‌ها یا کلیدهای ارائه‌دهنده نیاز ندارد
  - وابستگی مرورگر باید موجود باشد (`pnpm --dir ui exec playwright install chromium`)

### E2E: آزمون دود backend مربوط به OpenShell

- فرمان: `pnpm test:e2e:openshell`
- فایل: `extensions/openshell/src/backend.e2e.test.ts`
- دامنه:
  - از یک gateway محلی فعال OpenShell دوباره استفاده می‌کند
  - از یک Dockerfile محلی موقت، sandbox ایجاد می‌کند
  - backend مربوط به OpenShell در OpenClaw را از طریق `sandbox ssh-config` واقعی + اجرای SSH به کار می‌گیرد
  - رفتار سامانهٔ فایل استانداردِ راه‌دور را از طریق پل fs در sandbox تأیید می‌کند
- انتظارات:
  - فقط با انتخاب صریح؛ بخشی از اجرای پیش‌فرض `pnpm test:e2e` نیست
  - به یک CLI محلی `openshell` به‌همراه daemon فعال Docker نیاز دارد
  - به یک gateway محلی فعال OpenShell و منبع پیکربندی آن نیاز دارد
  - از `HOME` / `XDG_CONFIG_HOME` ایزوله استفاده می‌کند، سپس sandbox آزمون را از بین می‌برد
- بازنویسی‌های مفید:
  - `OPENCLAW_E2E_OPENSHELL=1` برای فعال‌کردن آزمون هنگام اجرای دستی مجموعهٔ گسترده‌تر e2e
  - `OPENCLAW_E2E_OPENSHELL_COMMAND=/path/to/openshell` برای اشاره به یک فایل اجرایی CLI یا اسکریپت wrapper غیراپیش‌فرض
  - `OPENCLAW_E2E_OPENSHELL_CONFIG_HOME=/path/to/config` برای در معرض قرار دادن پیکربندی gateway ثبت‌شده برای آزمون ایزوله
  - `OPENCLAW_E2E_OPENSHELL_HOST_IP=172.18.0.1` برای بازنویسی IP مربوط به gateway در Docker که fixture سیاست میزبان از آن استفاده می‌کند

### زنده (ارائه‌دهندگان واقعی + مدل‌های واقعی)

- دستور: `pnpm test:live`
- پیکربندی: `test/vitest/vitest.live.config.ts`
- فایل‌ها: `src/**/*.live.test.ts`، `test/**/*.live.test.ts` و آزمون‌های زندهٔ Pluginهای همراه در `extensions/`
- پیش‌فرض: با `pnpm test:live` **فعال** است (`OPENCLAW_LIVE_TEST=1` را تنظیم می‌کند)
- دامنه:
  - «آیا این ارائه‌دهنده/مدل واقعاً _امروز_ با اعتبارنامه‌های واقعی کار می‌کند؟»
  - تغییرات قالب ارائه‌دهنده، ظرافت‌های فراخوانی ابزار، مشکلات احراز هویت و رفتار محدودیت نرخ را شناسایی کنید
- انتظارات:
  - عمداً در CI پایدار نیست (شبکه‌های واقعی، سیاست‌های واقعی ارائه‌دهنده، سهمیه‌ها و قطعی‌ها)
  - هزینه دارد / از محدودیت‌های نرخ استفاده می‌کند
  - اجرای زیرمجموعه‌های محدودشده را به‌جای «همه‌چیز» ترجیح دهید
- اجراهای زنده از کلیدهای API ازپیش‌صادرشده و پروفایل‌های احراز هویت آماده‌شده استفاده می‌کنند.
- به‌طور پیش‌فرض، اجراهای زنده همچنان `HOME` را ایزوله می‌کنند و مواد پیکربندی/احراز هویت را در یک خانهٔ آزمایشی موقت کپی می‌کنند تا فیکسچرهای آزمون واحد نتوانند `~/.openclaw` واقعی شما را تغییر دهند.
- تنها زمانی `OPENCLAW_LIVE_USE_REAL_HOME=1` را تنظیم کنید که عمداً لازم است آزمون‌های زنده از دایرکتوری خانهٔ واقعی شما استفاده کنند.
- `pnpm test:live` به‌طور پیش‌فرض از حالت کم‌سروصداتری استفاده می‌کند: خروجی پیشرفت `[live] ...` را نگه می‌دارد و گزارش‌های راه‌اندازی Gateway/پیام‌های Bonjour را بی‌صدا می‌کند. اگر می‌خواهید گزارش‌های کامل راه‌اندازی بازگردند، `OPENCLAW_LIVE_TEST_QUIET=0` را تنظیم کنید.
- چرخش کلید API (مختص ارائه‌دهنده): `*_API_KEYS` را با قالب جداشده با ویرگول/نقطه‌ویرگول یا `*_API_KEY_1`، `*_API_KEY_2` (برای مثال `OPENAI_API_KEYS`، `ANTHROPIC_API_KEYS`، `GEMINI_API_KEYS`) تنظیم کنید، یا برای هر اجرای زنده از طریق `OPENCLAW_LIVE_*_KEY` بازنویسی کنید؛ آزمون‌ها هنگام دریافت پاسخ محدودیت نرخ دوباره تلاش می‌کنند.
- خروجی پیشرفت/Heartbeat:
  - مجموعه‌های زنده خطوط پیشرفت را در stderr منتشر می‌کنند تا فراخوانی‌های طولانی ارائه‌دهنده، حتی وقتی ضبط کنسول Vitest ساکت است، به‌وضوح فعال دیده شوند.
  - `test/vitest/vitest.live.config.ts` رهگیری کنسول Vitest را غیرفعال می‌کند تا خطوط پیشرفت ارائه‌دهنده/Gateway هنگام اجراهای زنده بلافاصله جریان یابند.
  - Heartbeatهای مستقیم مدل را با `OPENCLAW_LIVE_HEARTBEAT_MS` تنظیم کنید.
  - Heartbeatهای Gateway/کاوش را با `OPENCLAW_LIVE_GATEWAY_HEARTBEAT_MS` تنظیم کنید.

## کدام مجموعه را باید اجرا کنم؟

از این جدول تصمیم‌گیری استفاده کنید:

- ویرایش منطق/آزمون‌ها: `pnpm test` را اجرا کنید (و اگر تغییرات زیادی داده‌اید، `pnpm test:coverage` را نیز اجرا کنید)
- تغییر شبکهٔ Gateway / پروتکل WS / جفت‌سازی: `pnpm test:e2e` را اضافه کنید
- اشکال‌زدایی «ربات من از کار افتاده است» / خرابی‌های مختص ارائه‌دهنده / فراخوانی ابزار: یک `pnpm test:live` محدودشده اجرا کنید

## آزمون‌های زنده (دارای دسترسی به شبکه)

برای ماتریس مدل زنده، آزمون‌های دود CLI backend، آزمون‌های دود ACP، چارچوب
app-server مربوط به Codex و همهٔ آزمون‌های زندهٔ ارائه‌دهندگان رسانه (Deepgram، BytePlus، ComfyUI،
تصویر، موسیقی، ویدئو و چارچوب رسانه) ــ به‌علاوهٔ مدیریت اعتبارنامه برای اجراهای زنده

- [آزمون مجموعه‌های زنده](/fa/help/testing-live) را ببینید. برای چک‌لیست اختصاصی اعتبارسنجی
  به‌روزرسانی و Plugin،
  [آزمون به‌روزرسانی‌ها و Pluginها](/fa/help/testing-updates-plugins) را ببینید.

## اجراکننده‌های Docker (بررسی‌های اختیاری «در Linux کار می‌کند»)

این اجراکننده‌های Docker به دو دسته تقسیم می‌شوند:

- اجراکننده‌های مدل زنده: `test:docker:live-models` و `test:docker:live-gateway` فقط فایل زندهٔ کلید پروفایل متناظر خود را درون تصویر Docker مخزن اجرا می‌کنند (`src/agents/models.profiles.live.test.ts` و `src/gateway/gateway-models.profiles.live.test.ts`) و دایرکتوری پیکربندی محلی، فضای کاری و فایل اختیاری محیط پروفایل شما را mount می‌کنند. نقاط ورود محلی متناظر `test:live:models-profiles` و `test:live:gateway-profiles` هستند.
- اجراکننده‌های زندهٔ Docker در صورت نیاز سقف‌های عملی مختص خود را حفظ می‌کنند:
  `test:docker:live-models` به‌طور پیش‌فرض از مجموعهٔ منتخب، پشتیبانی‌شده و پربازده استفاده می‌کند و
  `test:docker:live-gateway` به‌طور پیش‌فرض شامل `OPENCLAW_LIVE_GATEWAY_SMOKE=1`،
  `OPENCLAW_LIVE_GATEWAY_MAX_MODELS=8`،
  `OPENCLAW_LIVE_GATEWAY_STEP_TIMEOUT_MS=45000` و
  `OPENCLAW_LIVE_GATEWAY_MODEL_TIMEOUT_MS=90000` است. زمانی که صراحتاً سقف کوچک‌تر یا پیمایش گسترده‌تری می‌خواهید، `OPENCLAW_LIVE_MAX_MODELS`
  یا متغیرهای محیطی Gateway را تنظیم کنید.
- `test:docker:all` تصویر زندهٔ Docker را یک‌بار از طریق `test:docker:live-build` می‌سازد، OpenClaw را یک‌بار از طریق `scripts/package-openclaw-for-docker.mjs` به‌صورت tarball مربوط به npm بسته‌بندی می‌کند و سپس دو تصویر `scripts/e2e/Dockerfile` را می‌سازد/دوباره استفاده می‌کند. تصویر پایه فقط اجراکنندهٔ Node/Git برای مسیرهای نصب/به‌روزرسانی/وابستگی Plugin است؛ این مسیرها tarball ازپیش‌ساخته‌شده را mount می‌کنند. تصویر کاربردی همان tarball را برای مسیرهای عملکرد برنامهٔ ساخته‌شده در `/app` نصب می‌کند. تعریف مسیرهای Docker در `scripts/lib/docker-e2e-scenarios.mjs` قرار دارد؛ منطق برنامه‌ریز در `scripts/lib/docker-e2e-plan.mjs` قرار دارد؛ `scripts/test-docker-all.mjs` برنامهٔ انتخاب‌شده را اجرا می‌کند. اجرای تجمیعی از زمان‌بند محلی وزن‌دار استفاده می‌کند: `OPENCLAW_DOCKER_ALL_PARALLELISM` شکاف‌های فرایند را کنترل می‌کند، درحالی‌که سقف منابع مانع شروع هم‌زمان همهٔ مسیرهای سنگین زنده، نصب npm و چندسرویسی می‌شود. اگر یک مسیر از سقف‌های فعال سنگین‌تر باشد، زمان‌بند همچنان می‌تواند آن را هنگام خالی‌بودن مخزن شروع کند و سپس تا زمانی که ظرفیت دوباره در دسترس شود، آن را به‌تنهایی در حال اجرا نگه می‌دارد. مقادیر پیش‌فرض 10 شکاف، `OPENCLAW_DOCKER_ALL_LIVE_LIMIT=9`، `OPENCLAW_DOCKER_ALL_NPM_LIMIT=5` و `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT=7` هستند؛ تنها زمانی `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` یا `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT` (و سایر بازنویسی‌های `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT`) را تنظیم کنید که میزبان Docker ظرفیت بیشتری داشته باشد. اجراکننده به‌طور پیش‌فرض پیش‌بررسی Docker را انجام می‌دهد، کانتینرهای قدیمی E2E مربوط به OpenClaw را حذف می‌کند، هر 30 ثانیه وضعیت را چاپ می‌کند، زمان‌بندی مسیرهای موفق را در `.artifacts/docker-tests/lane-timings.json` ذخیره می‌کند و در اجراهای بعدی از این زمان‌بندی‌ها برای شروع زودتر مسیرهای طولانی‌تر استفاده می‌کند. برای چاپ مانیفست وزن‌دار مسیرها بدون ساخت یا اجرای Docker از `OPENCLAW_DOCKER_ALL_DRY_RUN=1` استفاده کنید، یا برای چاپ برنامهٔ CI شامل مسیرهای انتخاب‌شده، نیازهای بسته/تصویر و اعتبارنامه‌ها از `node scripts/test-docker-all.mjs --plan-json` استفاده کنید.
- `Package Acceptance` دروازهٔ بومی GitHub برای بسته و پاسخ به این پرسش است: «آیا این tarball قابل‌نصب به‌عنوان یک محصول کار می‌کند؟» این دروازه یک بستهٔ نامزد را از `source=npm`، `source=ref`، `source=url`، `source=trusted-url` یا `source=artifact` تعیین می‌کند، آن را به‌عنوان `package-under-test` بارگذاری می‌کند و سپس مسیرهای قابل‌استفادهٔ مجدد Docker E2E را به‌جای بسته‌بندی مجدد ارجاع انتخاب‌شده، روی دقیقاً همان tarball اجرا می‌کند. پروفایل‌ها به‌ترتیب گستردگی مرتب شده‌اند: `smoke`، `package`، `product` و `full` (به‌علاوهٔ `custom` برای فهرست صریح مسیرها). برای قرارداد بسته/به‌روزرسانی/Plugin، ماتریس دوام پس از ارتقای منتشرشده، پیش‌فرض‌های انتشار و عیب‌یابی خرابی، [آزمون به‌روزرسانی‌ها و Pluginها](/fa/help/testing-updates-plugins) را ببینید.
- بررسی‌های ساخت و انتشار پس از tsdown، `scripts/check-cli-bootstrap-imports.mjs` را اجرا می‌کنند. این محافظ گراف ایستای ساخته‌شده را از `dist/entry.js` و `dist/cli/run-main.js` پیمایش می‌کند و اگر آن گراف راه‌اندازی پیش از اعزام دستور، هر بستهٔ خارجی را به‌صورت ایستا import کند (Commander، رابط کاربری اعلان، undici، گزارش‌گیری و وابستگی‌های سنگین مشابه هنگام راه‌اندازی همگی محاسبه می‌شوند)، ناموفق می‌شود؛ همچنین اندازهٔ قطعهٔ بسته‌بندی‌شدهٔ اجرای Gateway را به 70 KB محدود می‌کند و import ایستای مسیرهای سرد شناخته‌شدهٔ Gateway (`control-ui-assets`، `diagnostic-stability-bundle`، `onboard-helpers`، `process-respawn`، `restart-sentinel`، `server-close`، `server-reload-handlers`) را از آن قطعه رد می‌کند. `scripts/release-check.ts` نیز به‌طور جداگانه CLI بسته‌بندی‌شده را با `--help`، `onboard --help`، `doctor --help`، `status --json --timeout 1`، `config schema` و `models list --provider openai` آزمون دود می‌کند.
- سازگاری قدیمی Package Acceptance تا `2026.4.25` محدود است (`2026.4.25-beta.*` نیز شامل می‌شود). تا آن نقطهٔ پایانی، چارچوب فقط شکاف‌های فراداده‌ای بسته‌های منتشرشده را تحمل می‌کند: ورودی‌های حذف‌شدهٔ فهرست خصوصی QA، نبود `gateway install --wrapper`، نبود فایل‌های patch در فیکسچر git مشتق‌شده از tarball، نبود `update.channel` ماندگارشده، مکان‌های قدیمی رکورد نصب Plugin، نبود ماندگاری رکورد نصب بازار و مهاجرت فرادادهٔ پیکربندی هنگام `plugins update`. برای بسته‌های پس از `2026.4.25`، این مسیرها خطاهای سخت‌گیرانه محسوب می‌شوند.
- اجراکننده‌های آزمون دود کانتینر: `test:docker:openwebui`، `test:docker:onboard`، `test:docker:npm-onboard-channel-agent`، `test:docker:release-user-journey`، `test:docker:release-typed-onboarding`، `test:docker:release-media-memory`، `test:docker:release-upgrade-user-journey`، `test:docker:release-plugin-marketplace`، `test:docker:skill-install`، `test:docker:update-channel-switch`، `test:docker:upgrade-survivor`، `test:docker:published-upgrade-survivor`، `test:docker:session-runtime-context`، `test:docker:agents-delete-shared-workspace`، `test:docker:gateway-network`، `test:docker:browser-cdp-snapshot`، `test:docker:mcp-channels`، `test:docker:agent-bundle-mcp-tools`، `test:docker:cron-mcp-cleanup`، `test:docker:plugins`، `test:docker:plugin-update`، `test:docker:plugin-lifecycle-matrix` و `test:docker:config-reload` یک یا چند کانتینر واقعی را راه‌اندازی و مسیرهای یکپارچه‌سازی سطح‌بالاتر را تأیید می‌کنند.
- مسیرهای Docker/Bash E2E که tarball بسته‌بندی‌شدهٔ OpenClaw را از طریق `scripts/lib/openclaw-e2e-instance.sh` نصب می‌کنند، `npm install` را به `OPENCLAW_E2E_NPM_INSTALL_TIMEOUT` محدود می‌کنند (پیش‌فرض `600s`؛ برای غیرفعال‌کردن پوشش‌دهنده هنگام اشکال‌زدایی، `0` را تنظیم کنید).

اجراکننده‌های Docker مدل زنده همچنین فقط خانه‌های احراز هویت CLI موردنیاز
(یا همهٔ موارد پشتیبانی‌شده، وقتی اجرا محدود نشده است) را bind-mount می‌کنند و سپس پیش از اجرا آن‌ها را در
خانهٔ کانتینر کپی می‌کنند تا OAuth مربوط به CLI خارجی بتواند توکن‌ها را تازه‌سازی کند
بی‌آنکه مخزن احراز هویت میزبان تغییر کند:

- مدل‌های مستقیم: `pnpm test:docker:live-models` (اسکریپت: `scripts/test-live-models-docker.sh`)
- آزمون دود اتصال ACP: `pnpm test:docker:live-acp-bind` (اسکریپت: `scripts/test-live-acp-bind-docker.sh`؛ به‌طور پیش‌فرض Claude، Codex و Gemini را پوشش می‌دهد و پوشش سخت‌گیرانهٔ Droid/OpenCode از طریق `pnpm test:docker:live-acp-bind:droid` و `pnpm test:docker:live-acp-bind:opencode` ارائه می‌شود)
- آزمون دود CLI backend: `pnpm test:docker:live-cli-backend` (اسکریپت: `scripts/test-live-cli-backend-docker.sh`)
- آزمون دود چارچوب app-server مربوط به Codex: `pnpm test:docker:live-codex-harness` (اسکریپت: `scripts/test-live-codex-harness-docker.sh`)
- Gateway + عامل توسعه: `pnpm test:docker:live-gateway` (اسکریپت: `scripts/test-live-gateway-models-docker.sh`)
- آزمون‌های دود مشاهده‌پذیری: `pnpm qa:otel:smoke`، `pnpm qa:prometheus:smoke` و `pnpm qa:observability:smoke` مسیرهای خصوصی QA در وارسی کد منبع هستند. آن‌ها عمداً بخشی از مسیرهای انتشار Docker بسته نیستند، زیرا tarball مربوط به npm شامل QA Lab نمی‌شود.
- آزمون دود زندهٔ Open WebUI: `pnpm test:docker:openwebui` (اسکریپت: `scripts/e2e/openwebui-docker.sh`)
- جادوگر راه‌اندازی اولیه (TTY، داربست‌بندی کامل): `pnpm test:docker:onboard` (اسکریپت: `scripts/e2e/onboard-docker.sh`)
- آزمون دود راه‌اندازی اولیه/کانال/عامل با tarball مربوط به Npm: `pnpm test:docker:npm-onboard-channel-agent`، tarball بسته‌بندی‌شدهٔ OpenClaw را به‌صورت سراسری در Docker نصب می‌کند، OpenAI را از طریق راه‌اندازی اولیهٔ ارجاع محیطی و همچنین Telegram را به‌طور پیش‌فرض پیکربندی می‌کند، doctor را اجرا می‌کند و یک نوبت عامل شبیه‌سازی‌شدهٔ OpenAI را اجرا می‌کند. برای استفادهٔ مجدد از tarball ازپیش‌ساخته‌شده از `OPENCLAW_CURRENT_PACKAGE_TGZ=/path/to/openclaw-*.tgz`، برای ردکردن ساخت مجدد میزبان از `OPENCLAW_NPM_ONBOARD_HOST_BUILD=0` و برای تغییر کانال از `OPENCLAW_NPM_ONBOARD_CHANNEL=discord` یا `OPENCLAW_NPM_ONBOARD_CHANNEL=slack` استفاده کنید.

- آزمون دود مسیر کاربر انتشار: `pnpm test:docker:release-user-journey` بستهٔ tarball‌شدهٔ OpenClaw را به‌صورت سراسری در یک محیط خانگی پاک Docker نصب می‌کند، راه‌اندازی اولیه را اجرا می‌کند، یک ارائه‌دهندهٔ شبیه‌سازی‌شدهٔ OpenAI را پیکربندی می‌کند، یک نوبت عامل را اجرا می‌کند، Pluginهای خارجی را نصب/حذف نصب می‌کند، ClickClack را در برابر یک فیکسچر محلی پیکربندی می‌کند، پیام‌رسانی خروجی/ورودی را تأیید می‌کند، Gateway را بازراه‌اندازی می‌کند و doctor را اجرا می‌کند.
- آزمون دود راه‌اندازی اولیهٔ نوع‌دار انتشار: `pnpm test:docker:release-typed-onboarding` بستهٔ tarball‌شده را نصب می‌کند، `openclaw onboard` را از طریق یک TTY واقعی هدایت می‌کند، OpenAI را به‌عنوان ارائه‌دهندهٔ env-ref پیکربندی می‌کند، تأیید می‌کند که کلید خام ذخیره نمی‌شود و یک نوبت عامل شبیه‌سازی‌شده را اجرا می‌کند.
- آزمون دود رسانه/حافظهٔ انتشار: `pnpm test:docker:release-media-memory` بستهٔ tarball‌شده را نصب می‌کند و درک تصویر از یک پیوست PNG، خروجی تولید تصویر سازگار با OpenAI، بازیابی جست‌وجوی حافظه و دوام بازیابی پس از بازراه‌اندازی Gateway را تأیید می‌کند.
- آزمون دود مسیر کاربر ارتقای انتشار: `pnpm test:docker:release-upgrade-user-journey` به‌طور پیش‌فرض جدیدترین نسخهٔ پایهٔ منتشرشده‌ای را که از tarball نامزد قدیمی‌تر است نصب می‌کند، وضعیت ارائه‌دهنده/Plugin/ClickClack را روی بستهٔ منتشرشده پیکربندی می‌کند، به tarball نامزد ارتقا می‌دهد و سپس مسیر اصلی عامل/Plugin/کانال را دوباره اجرا می‌کند. اگر نسخهٔ پایهٔ منتشرشدهٔ قدیمی‌تری وجود نداشته باشد، نسخهٔ نامزد را دوباره استفاده می‌کند. نسخهٔ پایه را با `OPENCLAW_RELEASE_UPGRADE_BASELINE_SPEC=openclaw@<version>` بازنویسی کنید.
- آزمون دود بازار Plugin انتشار: `pnpm test:docker:release-plugin-marketplace` از یک بازار فیکسچر محلی نصب می‌کند، Plugin نصب‌شده را به‌روزرسانی می‌کند، آن را حذف نصب می‌کند و تأیید می‌کند که CLI مربوط به Plugin همراه با پاک‌سازی فرادادهٔ نصب ناپدید می‌شود.
- آزمون دود نصب Skill: `pnpm test:docker:skill-install` بستهٔ tarball‌شدهٔ OpenClaw را به‌صورت سراسری در Docker نصب می‌کند، نصب بایگانی‌های بارگذاری‌شده را در پیکربندی غیرفعال می‌کند، slug فعلی Skill زندهٔ ClawHub را از جست‌وجو به‌دست می‌آورد، آن را با `openclaw skills install` نصب می‌کند و Skill نصب‌شده به‌همراه فرادادهٔ مبدأ/قفل `.clawhub` را تأیید می‌کند.
- آزمون دود تغییر کانال به‌روزرسانی: `pnpm test:docker:update-channel-switch` بستهٔ tarball‌شدهٔ OpenClaw را به‌صورت سراسری در Docker نصب می‌کند، از بستهٔ `stable` به git `dev` تغییر می‌دهد، ماندگاری کانال و عملکرد Plugin پس از به‌روزرسانی را تأیید می‌کند، سپس به بستهٔ `stable` بازمی‌گردد و وضعیت به‌روزرسانی را بررسی می‌کند.
- آزمون دود دوام پس از ارتقا: `pnpm test:docker:upgrade-survivor` بستهٔ tarball‌شدهٔ OpenClaw را روی یک فیکسچر کثیف کاربر قدیمی شامل عامل‌ها، پیکربندی کانال، فهرست‌های مجاز Plugin، وضعیت کهنهٔ وابستگی Plugin و فایل‌های موجود فضای کاری/نشست نصب می‌کند. سپس بدون کلیدهای زندهٔ ارائه‌دهنده یا کانال، به‌روزرسانی بسته و doctor غیرتعاملی را اجرا می‌کند، یک Gateway حلقهٔ بازگشتی را راه‌اندازی می‌کند و حفظ پیکربندی/وضعیت به‌همراه بودجه‌های راه‌اندازی/وضعیت را بررسی می‌کند.
- آزمون دود دوام پس از ارتقای منتشرشده: `pnpm test:docker:published-upgrade-survivor` به‌طور پیش‌فرض `openclaw@latest` را نصب می‌کند، فایل‌های واقع‌گرایانهٔ کاربر موجود را مقداردهی اولیه می‌کند، آن نسخهٔ پایه را با یک دستورالعمل فرمان تعبیه‌شده پیکربندی می‌کند، پیکربندی حاصل را اعتبارسنجی می‌کند، نصب منتشرشده را به tarball نامزد به‌روزرسانی می‌کند، doctor غیرتعاملی را اجرا می‌کند، `.artifacts/upgrade-survivor/summary.json` را می‌نویسد، سپس یک Gateway حلقهٔ بازگشتی را راه‌اندازی می‌کند و قصدهای پیکربندی‌شده، حفظ وضعیت، راه‌اندازی، `/healthz`، `/readyz` و بودجه‌های وضعیت RPC را بررسی می‌کند. یک نسخهٔ پایه را با `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` بازنویسی کنید، از زمان‌بند تجمیعی بخواهید نسخه‌های پایهٔ محلی دقیق را با `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` مانند `openclaw@2026.5.2 openclaw@2026.4.23 openclaw@2026.4.15` گسترش دهد و فیکسچرهای مسئله‌محور را با `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS` مانند `reported-issues` گسترش دهد؛ مجموعهٔ مشکلات گزارش‌شده شامل `configured-plugin-installs` برای ترمیم خودکار نصب Plugin خارجی OpenClaw است. Package Acceptance این موارد را به‌صورت `published_upgrade_survivor_baseline`، `published_upgrade_survivor_baselines` و `published_upgrade_survivor_scenarios` ارائه می‌کند، توکن‌های فرای نسخهٔ پایه مانند `last-stable-4` یا `all-since-2026.4.23` را تفکیک می‌کند و Full Release Validation دروازهٔ بستهٔ آزمون ماندگاری انتشار را به `last-stable-4 2026.4.23 2026.5.2 2026.4.15` به‌علاوهٔ `reported-issues` گسترش می‌دهد.
- آزمون دود زمینهٔ زمان‌اجرای نشست: `pnpm test:docker:session-runtime-context` ماندگاری رونوشت زمینهٔ پنهان زمان‌اجرا و نیز ترمیم شاخه‌های تکراری آسیب‌دیدهٔ بازنویسی پرامپت توسط doctor را تأیید می‌کند.
- آزمون دود نصب سراسری Bun: `bash scripts/e2e/bun-global-install-smoke.sh` درخت فعلی را بسته‌بندی می‌کند، آن را با `bun install -g` در یک محیط خانگی ایزوله نصب می‌کند و تأیید می‌کند که `openclaw infer image providers --json` به‌جای معلق ماندن، ارائه‌دهندگان تصویر همراه بسته را برمی‌گرداند. یک tarball ازپیش‌ساخته‌شده را با `OPENCLAW_BUN_GLOBAL_SMOKE_PACKAGE_TGZ=/path/to/openclaw-*.tgz` دوباره استفاده کنید، ساخت میزبان را با `OPENCLAW_BUN_GLOBAL_SMOKE_HOST_BUILD=0` رد کنید یا `dist/` را با `OPENCLAW_BUN_GLOBAL_SMOKE_DIST_IMAGE=openclaw-dockerfile-smoke:local` از یک تصویر Docker ساخته‌شده کپی کنید.
- آزمون دود Docker نصب‌کننده: `bash scripts/test-install-sh-docker.sh` یک کش npm را میان کانتینرهای root، به‌روزرسانی و direct-npm خود به‌اشتراک می‌گذارد. آزمون دود به‌روزرسانی، پیش از ارتقا به tarball نامزد، به‌طور پیش‌فرض npm `latest` را نسخهٔ پایهٔ پایدار در نظر می‌گیرد. آن را به‌صورت محلی با `OPENCLAW_INSTALL_SMOKE_UPDATE_BASELINE=2026.4.22` یا در GitHub با ورودی `update_baseline_version` گردش‌کار Install Smoke بازنویسی کنید. بررسی‌های نصب‌کنندهٔ غیر root یک کش npm ایزوله نگه می‌دارند تا ورودی‌های کش متعلق به root رفتار نصب محلی کاربر را پنهان نکنند. برای استفادهٔ دوباره از کش root/update/direct-npm در اجرای مجدد محلی، `OPENCLAW_INSTALL_SMOKE_NPM_CACHE_DIR=/path/to/cache` را تنظیم کنید.
- پایپ‌لاین CI آزمون Install Smoke با `OPENCLAW_INSTALL_SMOKE_SKIP_NPM_GLOBAL=1` به‌روزرسانی سراسری تکراری direct-npm را رد می‌کند؛ هنگامی که پوشش مستقیم `npm install -g` لازم است، اسکریپت را به‌صورت محلی بدون آن متغیر محیطی اجرا کنید.
- آزمون دود CLI حذف فضای کاری مشترک عامل‌ها: `pnpm test:docker:agents-delete-shared-workspace` (اسکریپت: `scripts/e2e/agents-delete-shared-workspace-docker.sh`) به‌طور پیش‌فرض تصویر Dockerfile ریشه را می‌سازد، دو عامل را با یک فضای کاری در محیط خانگی ایزولهٔ کانتینر مقداردهی اولیه می‌کند، `agents delete --json` را اجرا می‌کند و JSON معتبر به‌همراه رفتار حفظ فضای کاری را تأیید می‌کند. تصویر install-smoke را با `OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_IMAGE=openclaw-dockerfile-smoke:local OPENCLAW_AGENTS_DELETE_SHARED_WORKSPACE_E2E_SKIP_BUILD=1` دوباره استفاده کنید.
- شبکه و چرخهٔ عمر میزبان Gateway: `pnpm test:docker:gateway-network` (اسکریپت: `scripts/e2e/gateway-network-docker.sh`) آزمون دود احراز هویت/سلامت WebSocket شبکهٔ محلی دوکانتینری را حفظ می‌کند، سپس از HTTP مدیریتی حلقهٔ بازگشتی برای اثبات حصارگذاری آماده‌سازی، دسترسی کنترلی حفظ‌شده، بازیابی از سرگیری و توقف/شروع آماده‌شده در همان کانتینر استفاده می‌کند. بررسی بازراه‌اندازی باید پیش از انقضای اجارهٔ اصلی تکمیل شود، تأیید می‌کند که وضعیت تعلیق محلیِ فرایند است، درحالی‌که پیکربندی ماندگار Gateway و هویت کانتینر باقی می‌مانند، و JSON زمان‌بندی فازها را به‌صورت ماشین‌خوان منتشر می‌کند.
- آزمون دود snapshot پروتکل CDP مرورگر: `pnpm test:docker:browser-cdp-snapshot` (اسکریپت: `scripts/e2e/browser-cdp-snapshot-docker.sh`) تصویر E2E منبع به‌همراه یک لایهٔ Chromium را می‌سازد، Chromium را با CDP خام راه‌اندازی می‌کند، `browser doctor --deep` را اجرا می‌کند و تأیید می‌کند که snapshotهای نقش CDP نشانی‌های اینترنتی پیوندها، عناصر کلیک‌پذیر ارتقایافته با نشانگر، ارجاع‌های iframe و فرادادهٔ قاب را پوشش می‌دهند.
- پسرفت استدلال حداقلی web_search در OpenAI Responses: `pnpm test:docker:openai-web-search-minimal` (اسکریپت: `scripts/e2e/openai-web-search-minimal-docker.sh`) یک سرور شبیه‌سازی‌شدهٔ OpenAI را از طریق Gateway اجرا می‌کند، تأیید می‌کند که `web_search` مقدار `reasoning.effort` را از `minimal` به `low` افزایش می‌دهد، سپس رد شدن شِمای ارائه‌دهنده را اجبار می‌کند و بررسی می‌کند که جزئیات خام در گزارش‌های Gateway ظاهر می‌شوند.
- پل کانال MCP (Gateway مقداردهی‌شده + پل stdio + آزمون دود قاب اعلان خام Claude): `pnpm test:docker:mcp-channels` (اسکریپت: `scripts/e2e/mcp-channels-docker.sh`)
- ابزارهای MCP بستهٔ OpenClaw (سرور واقعی MCP مبتنی بر stdio + آزمون دود اجازه/رد نمایهٔ تعبیه‌شدهٔ OpenClaw): `pnpm test:docker:agent-bundle-mcp-tools` (اسکریپت: `scripts/e2e/agent-bundle-mcp-tools-docker.sh`)
- پاک‌سازی MCP برای Cron/زیرعامل (Gateway واقعی + خاتمهٔ فرزند MCP مبتنی بر stdio پس از اجرای Cron ایزوله و زیرعامل یک‌باره): `pnpm test:docker:cron-mcp-cleanup` (اسکریپت: `scripts/e2e/cron-mcp-cleanup-docker.sh`)
- Pluginها (آزمون دود نصب/به‌روزرسانی برای مسیر محلی، `file:`، رجیستری npm با وابستگی‌های بالاکشیده‌شده، فرادادهٔ نادرست بستهٔ npm، ارجاع‌های متحرک git، مجموعهٔ جامع ClawHub، به‌روزرسانی‌های بازار و فعال‌سازی/بازرسی بستهٔ Claude): `pnpm test:docker:plugins` (اسکریپت: `scripts/e2e/plugins-docker.sh`)
  برای رد کردن بلوک ClawHub، `OPENCLAW_PLUGINS_E2E_CLAWHUB=0` را تنظیم کنید یا جفت پیش‌فرض بسته/زمان‌اجرای مجموعهٔ جامع را با `OPENCLAW_PLUGINS_E2E_CLAWHUB_SPEC` و `OPENCLAW_PLUGINS_E2E_CLAWHUB_ID` بازنویسی کنید. بدون `OPENCLAW_CLAWHUB_URL`/`CLAWHUB_URL`، آزمون از یک سرور فیکسچر محلی و محصور ClawHub استفاده می‌کند.
- آزمون دود بدون تغییر به‌روزرسانی Plugin: `pnpm test:docker:plugin-update` (اسکریپت: `scripts/e2e/plugin-update-unchanged-docker.sh`)
- آزمون دود ماتریس چرخهٔ عمر Plugin: `pnpm test:docker:plugin-lifecycle-matrix` بستهٔ tarball‌شدهٔ OpenClaw را در یک کانتینر خالی نصب می‌کند، یک Plugin مبتنی بر npm نصب می‌کند، وضعیت فعال/غیرفعال را تغییر می‌دهد، آن را از طریق یک رجیستری محلی npm ارتقا و تنزل می‌دهد، کد نصب‌شده را حذف می‌کند و سپس تأیید می‌کند که حذف نصب همچنان وضعیت کهنه را حذف می‌کند و هم‌زمان سنجه‌های RSS/CPU را برای هر فاز چرخهٔ عمر ثبت می‌کند.
- آزمون دود فرادادهٔ بارگذاری مجدد پیکربندی: `pnpm test:docker:config-reload` (اسکریپت: `scripts/e2e/config-reload-source-docker.sh`)
- Pluginها: `pnpm test:docker:plugins` آزمون دود نصب/به‌روزرسانی برای مسیر محلی، `file:`، رجیستری npm با وابستگی‌های بالاکشیده‌شده، ارجاع‌های متحرک git، فیکسچرهای ClawHub، به‌روزرسانی‌های بازار و فعال‌سازی/بازرسی بستهٔ Claude را پوشش می‌دهد. `pnpm test:docker:plugin-update` رفتار به‌روزرسانی بدون تغییر برای Pluginهای نصب‌شده را پوشش می‌دهد. `pnpm test:docker:plugin-lifecycle-matrix` نصب، فعال‌سازی، غیرفعال‌سازی، ارتقا، تنزل و حذف نصب در نبود کد را برای Plugin مبتنی بر npm با ردیابی منابع پوشش می‌دهد.

برای پیش‌ساخت و استفادهٔ مجدد دستی از تصویر عملکردی مشترک:

```bash
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local pnpm test:docker:e2e-build
OPENCLAW_DOCKER_E2E_IMAGE=openclaw-docker-e2e-functional:local OPENCLAW_SKIP_DOCKER_BUILD=1 pnpm test:docker:mcp-channels
```

بازنویسی‌های تصویر مختص مجموعه مانند `OPENCLAW_GATEWAY_NETWORK_E2E_IMAGE` در صورت تنظیم همچنان اولویت دارند. هنگامی که `OPENCLAW_SKIP_DOCKER_BUILD=1` به یک تصویر مشترک راه‌دور اشاره می‌کند، اسکریپت‌ها در صورت نبود آن در محیط محلی، تصویر را دریافت می‌کنند. آزمون‌های Docker مربوط به QR و نصب‌کننده، Dockerfileهای خود را نگه می‌دارند، زیرا رفتار بسته/نصب را اعتبارسنجی می‌کنند، نه زمان‌اجرای مشترک برنامهٔ ساخته‌شده را.

اجراکننده‌های Docker مدل زنده همچنین checkout فعلی را به‌صورت فقط‌خواندنی
bind-mount می‌کنند و آن را در یک پوشهٔ کاری موقت داخل کانتینر قرار می‌دهند. این کار
تصویر زمان‌اجرا را کم‌حجم نگه می‌دارد و درعین‌حال Vitest را دقیقاً روی
منبع/پیکربندی محلی شما اجرا می‌کند. مرحلهٔ آماده‌سازی از کش‌های بزرگ صرفاً محلی و خروجی‌های
ساخت برنامه مانند `.pnpm-store`، `.worktrees`، `__openclaw_vitest__` و
`.build` محلی برنامه یا پوشه‌های خروجی Gradle صرف‌نظر می‌کند تا اجرای زندهٔ Docker
دقایقی را صرف کپی‌کردن مصنوعات مختص دستگاه نکند. آن‌ها همچنین
`OPENCLAW_SKIP_CHANNELS=1` را تنظیم می‌کنند تا کاوشگرهای زندهٔ Gateway، workerهای واقعی کانال
Telegram/Discord/و غیره را داخل کانتینر راه‌اندازی نکنند.
`test:docker:live-models` همچنان `pnpm test:live` را اجرا می‌کند، بنابراین هنگامی که لازم است پوشش زندهٔ
Gateway را در آن مسیر Docker محدود یا مستثنا کنید، `OPENCLAW_LIVE_GATEWAY_*` را نیز
عبور دهید.

`test:docker:openwebui` یک آزمون دود سازگاری سطح‌بالاتر است: یک کانتینر
Gateway مربوط به OpenClaw را با endpointهای HTTP سازگار با OpenAI راه‌اندازی می‌کند،
یک کانتینر Open WebUI سنجاق‌شده را در برابر آن Gateway راه‌اندازی می‌کند، از طریق
Open WebUI وارد می‌شود، تأیید می‌کند که `/api/models`، `openclaw/default` را ارائه می‌دهد و سپس یک
درخواست گفت‌وگوی واقعی را از طریق پراکسی `/api/chat/completions` در Open WebUI ارسال می‌کند. برای بررسی‌های پایپ‌لاین CI مسیر انتشار که باید
پس از ورود به Open WebUI و کشف مدل متوقف شوند، بدون انتظار برای تکمیل مدل زنده،
`OPENWEBUI_SMOKE_MODE=models` را تنظیم کنید. اجرای نخست ممکن است به‌طور محسوسی کندتر باشد، زیرا Docker شاید نیاز داشته باشد
تصویر Open WebUI را دریافت کند و Open WebUI نیز ممکن است لازم باشد
راه‌اندازی سرد خود را کامل کند. این مسیر به یک کلید قابل‌استفادهٔ مدل زنده نیاز دارد که از طریق
محیط فرایند، نمایه‌های احراز هویت آماده‌شده یا یک
`OPENCLAW_PROFILE_FILE` صریح ارائه می‌شود. اجراهای موفق یک payload کوچک JSON مانند
`{ "ok": true, "model": "openclaw/default", ... }` را چاپ می‌کنند.

`test:docker:mcp-channels` عمداً قطعی است و به
حساب واقعی Telegram، Discord یا iMessage نیاز ندارد. یک کانتینر Gateway مقداردهی‌شده را راه‌اندازی می‌کند،
کانتینر دومی را آغاز می‌کند که `openclaw mcp serve` را ایجاد می‌کند، سپس
کشف مکالمهٔ مسیریابی‌شده، خواندن رونوشت، فرادادهٔ پیوست،
رفتار صف رویداد زنده، مسیریابی ارسال خروجی و اعلان‌های کانال و مجوز
به‌سبک Claude را از طریق پل واقعی MCP مبتنی بر stdio تأیید می‌کند. بررسی
اعلان، قاب‌های خام MCP مبتنی بر stdio را مستقیماً بازرسی می‌کند تا آزمون دود
آنچه پل واقعاً منتشر می‌کند را اعتبارسنجی کند، نه فقط آنچه یک SDK کلاینت خاص
اتفاقاً نمایان می‌کند.

`test:docker:agent-bundle-mcp-tools` قطعی است و به کلید مدل زنده نیاز ندارد. این فرایند تصویر Docker مخزن را می‌سازد، یک سرور واقعی کاوشگر MCP مبتنی بر stdio را درون کانتینر راه‌اندازی می‌کند، آن سرور را از طریق زمان‌اجرای MCP بسته تعبیه‌شده OpenClaw در دسترس قرار می‌دهد، ابزار را اجرا می‌کند و سپس تأیید می‌کند که
`coding` و `messaging` ابزارهای `bundle-mcp` را نگه می‌دارند، درحالی‌که `minimal` و
`tools.deny: ["bundle-mcp"]` آن‌ها را فیلتر می‌کنند.

`test:docker:cron-mcp-cleanup` قطعی است و به کلید مدل زنده نیاز ندارد. این فرایند یک Gateway ازپیش‌مقداردهی‌شده را همراه با یک سرور واقعی کاوشگر MCP مبتنی بر stdio راه‌اندازی می‌کند، یک نوبت Cron ایزوله و یک نوبت فرزند یک‌باره `sessions_spawn` را اجرا می‌کند و سپس تأیید می‌کند که فرایند فرزند MCP پس از هر اجرا خاتمه می‌یابد.

آزمون دود دستی رشته ACP با زبان طبیعی (نه CI):

- `bun scripts/dev/discord-acp-plain-language-smoke.ts --channel <discord-channel-id> ...`
- این اسکریپت را برای گردش‌کارهای رگرسیون/اشکال‌زدایی نگه دارید. ممکن است دوباره برای اعتبارسنجی مسیریابی رشته ACP لازم شود؛ بنابراین آن را حذف نکنید.

متغیرهای محیطی مفید:

- `OPENCLAW_CONFIG_DIR=...` (پیش‌فرض: `~/.openclaw`) که در `/home/node/.openclaw` سوار می‌شود
- `OPENCLAW_WORKSPACE_DIR=...` (پیش‌فرض: `~/.openclaw/workspace`) که در `/home/node/.openclaw/workspace` سوار می‌شود
- `OPENCLAW_PROFILE_FILE=...` که پیش از اجرای آزمون‌ها سوار و منبع‌دهی می‌شود
- `OPENCLAW_DOCKER_PROFILE_ENV_ONLY=1` برای تأیید اینکه فقط متغیرهای محیطی منبع‌دهی‌شده از `OPENCLAW_PROFILE_FILE` استفاده می‌شوند؛ با دایرکتوری‌های موقت پیکربندی/فضای کاری و بدون سوارکردن احراز هویت CLI خارجی
- `OPENCLAW_DOCKER_CLI_TOOLS_DIR=...` (پیش‌فرض: `~/.cache/openclaw/docker-cli-tools`، مگر اینکه اجرا از قبل از یک دایرکتوری اتصال CI/مدیریت‌شده استفاده کند) که برای نصب‌های کش‌شده CLI درون Docker در `/home/node/.npm-global` سوار می‌شود
- دایرکتوری‌ها/فایل‌های احراز هویت CLI خارجی زیر `$HOME` به‌صورت فقط‌خواندنی زیر `/host-auth...` سوار می‌شوند و سپس پیش از آغاز آزمون‌ها در `/home/node/...` کپی می‌شوند
  - دایرکتوری‌های پیش‌فرض (هنگامی‌که اجرا به ارائه‌دهندگان مشخص محدود نشده است): `.factory`، `.gemini`، `.minimax`
  - فایل‌های پیش‌فرض: `~/.codex/auth.json`، `~/.codex/config.toml`، `.claude.json`، `~/.claude/.credentials.json`، `~/.claude/settings.json`، `~/.claude/settings.local.json`
  - اجراهای محدودشده به ارائه‌دهنده فقط دایرکتوری‌ها/فایل‌های لازمِ استنباط‌شده از `OPENCLAW_LIVE_PROVIDERS` / `OPENCLAW_LIVE_GATEWAY_PROVIDERS` را سوار می‌کنند
  - با `OPENCLAW_DOCKER_AUTH_DIRS=all`، `OPENCLAW_DOCKER_AUTH_DIRS=none` یا فهرستی جداشده با ویرگول مانند `OPENCLAW_DOCKER_AUTH_DIRS=.claude,.codex` به‌صورت دستی بازنویسی کنید
- `OPENCLAW_LIVE_GATEWAY_MODELS=...` / `OPENCLAW_LIVE_MODELS=...` برای محدودکردن اجرا
- `OPENCLAW_LIVE_GATEWAY_PROVIDERS=...` / `OPENCLAW_LIVE_PROVIDERS=...` برای فیلترکردن ارائه‌دهندگان درون کانتینر
- `OPENCLAW_SKIP_DOCKER_BUILD=1` برای استفاده مجدد از تصویر موجود `openclaw:local-live` در اجراهای مجددی که به بازسازی نیاز ندارند
- `OPENCLAW_LIVE_REQUIRE_PROFILE_KEYS=1` برای اطمینان از اینکه اطلاعات اعتبارسنجی از مخزن پروفایل می‌آیند (نه از محیط)
- `OPENCLAW_OPENWEBUI_MODEL=...` برای انتخاب مدلی که Gateway برای آزمون دود Open WebUI ارائه می‌کند
- `OPENCLAW_OPENWEBUI_PROMPT=...` برای بازنویسی اعلان بررسی nonce که آزمون دود Open WebUI استفاده می‌کند
- `OPENWEBUI_IMAGE=...` برای بازنویسی برچسب تصویر سنجاق‌شده Open WebUI

## بررسی سلامت مستندات

پس از ویرایش مستندات، بررسی‌های مستندات را اجرا کنید: `pnpm check:docs`.
هنگامی‌که به بررسی سرفصل‌های درون‌صفحه‌ای نیز نیاز دارید، اعتبارسنجی کامل لنگرهای Mintlify را اجرا کنید: `pnpm docs:check-links:anchors`.

## رگرسیون آفلاین (ایمن برای CI)

این‌ها رگرسیون‌های «پایپ‌لاین واقعی» بدون ارائه‌دهندگان واقعی هستند:

- فراخوانی ابزار Gateway (OpenAI شبیه‌سازی‌شده، Gateway واقعی + حلقه عامل): `src/gateway/gateway.test.ts` (مورد: «فراخوانی ابزار OpenAI شبیه‌سازی‌شده را به‌صورت سرتاسری از طریق حلقه عامل Gateway اجرا می‌کند»)
- جادوگر Gateway (WS `wizard.start`/`wizard.next`، نوشتن پیکربندی + اعمال احراز هویت): `src/gateway/gateway.test.ts` (مورد: «جادوگر را از طریق ws اجرا می‌کند و پیکربندی توکن احراز هویت را می‌نویسد»)

## ارزیابی‌های قابلیت اطمینان عامل (Skills)

از قبل چند آزمون ایمن برای CI داریم که مانند «ارزیابی‌های قابلیت اطمینان عامل» عمل می‌کنند:

- فراخوانی ابزار شبیه‌سازی‌شده از طریق Gateway واقعی + حلقه عامل (`src/gateway/gateway.test.ts`).
- جریان‌های سرتاسری جادوگر که سیم‌کشی نشست و اثرات پیکربندی را اعتبارسنجی می‌کنند (`src/gateway/gateway.test.ts`).

مواردی که هنوز برای Skills وجود ندارند (به [Skills](/fa/tools/skills) مراجعه کنید):

- **تصمیم‌گیری:** وقتی Skills در اعلان فهرست می‌شوند، آیا عامل Skill درست را انتخاب می‌کند (یا از موارد نامرتبط اجتناب می‌کند)؟
- **انطباق:** آیا عامل پیش از استفاده، `SKILL.md` را می‌خواند و مراحل/آرگومان‌های الزامی را دنبال می‌کند؟
- **قراردادهای گردش‌کار:** سناریوهای چندنوبتی که ترتیب ابزارها، انتقال تاریخچه نشست و مرزهای جعبه شنی را بررسی می‌کنند.

ارزیابی‌های آینده باید ابتدا قطعی بمانند:

- یک اجراکننده سناریو با استفاده از ارائه‌دهندگان شبیه‌سازی‌شده برای بررسی فراخوانی ابزارها + ترتیب، خواندن فایل‌های Skill و سیم‌کشی نشست.
- یک مجموعه کوچک از سناریوهای متمرکز بر Skill (استفاده در برابر اجتناب، دروازه‌گذاری، تزریق اعلان).
- ارزیابی‌های زنده اختیاری (با انتخاب صریح و محدودشده با متغیر محیطی) فقط پس از آماده‌شدن مجموعه ایمن برای CI.

## آزمون‌های قرارداد (شکل Plugin و کانال)

آزمون‌های قرارداد تأیید می‌کنند که هر Plugin و کانال ثبت‌شده با
قرارداد رابط خود مطابقت دارد. آن‌ها روی همه Pluginهای کشف‌شده پیمایش می‌کنند و یک
مجموعه از بررسی‌های شکل و رفتار را اجرا می‌کنند. مسیر واحد پیش‌فرض `pnpm test`
عمداً از این فایل‌های مشترک اتصال و آزمون دود صرف‌نظر می‌کند؛ هنگامی‌که سطوح مشترک کانال یا ارائه‌دهنده را تغییر می‌دهید، فرمان‌های قرارداد را صریحاً اجرا کنید.

### فرمان‌ها

- همه قراردادها: `pnpm test:contracts`
- فقط قراردادهای کانال: `pnpm test:contracts:channels`
- فقط قراردادهای ارائه‌دهنده: `pnpm test:contracts:plugins`

### قراردادهای کانال

در `src/channels/plugins/contracts/*.contract.test.ts` قرار دارند. دسته‌های
سطح‌بالای کنونی:

- **کاتالوگ کانال** - فراداده ورودی کاتالوگ کانال بسته‌بندی‌شده/رجیستری
- **Plugin** (مبتنی بر رجیستری، بخش‌بندی‌شده) - شکل پایه ثبت Plugin
- **فقط سطوح** (مبتنی بر رجیستری، بخش‌بندی‌شده) - بررسی شکل هر سطح برای `actions`، `setup`، `status`، `outbound`، `messaging`، `threading`، `directory` و `gateway`
- **اتصال نشست** (مبتنی بر رجیستری) - رفتار اتصال نشست
- **محموله خروجی** - ساختار و نرمال‌سازی محموله پیام
- **سیاست گروه** (جایگزین) - اعمال سیاست پیش‌فرض گروه برای هر کانال
- **رشته‌بندی** (مبتنی بر رجیستری، بخش‌بندی‌شده) - مدیریت شناسه رشته
- **دایرکتوری** (مبتنی بر رجیستری، بخش‌بندی‌شده) - API دایرکتوری/فهرست اعضا
- **رجیستری** و **plugins-core.\*** - رجیستری Plugin کانال، بارگذار و جزئیات داخلی مجوز نوشتن پیکربندی

راهنماهای چارچوب ثبت توزیع ورودی و محموله خروجی که این
مجموعه‌ها استفاده می‌کنند، در داخل از طریق `src/plugin-sdk/channel-contract-testing.ts`
(مستثناشده از npm، نه یک زیرمسیر عمومی SDK) ارائه می‌شوند؛ هیچ فایل مستقلی با نام
`inbound.contract.test.ts` در این دایرکتوری وجود ندارد.

### قراردادهای ارائه‌دهنده

در `src/plugins/contracts/*.contract.test.ts` قرار دارند. دسته‌های کنونی
شامل موارد زیر هستند:

- **شکل** - شکل مانیفست Plugin، API و خروجی زمان‌اجرا
- **ثبت Plugin** (+ موازی) - موارد ثبت مانیفست
- **مانیفست بسته** - الزامات مانیفست بسته
- **بارگذار** - رفتار راه‌اندازی/جمع‌آوری بارگذار Plugin
- **رجیستری** - محتویات و جست‌وجوی رجیستری قرارداد Plugin
- **ارائه‌دهندگان** - رفتار مشترک ارائه‌دهنده در میان ارائه‌دهندگان بسته‌بندی‌شده، به‌علاوه ارائه‌دهندگان جست‌وجوی وب
- **انتخاب احراز هویت** - فراداده انتخاب احراز هویت و رفتار راه‌اندازی
- **منسوخ‌سازی کاتالوگ ارائه‌دهنده** - فراداده کاتالوگ ارائه‌دهنده منسوخ‌شده
- **تفکیک انتخاب جادوگر**، **انتخابگر مدل جادوگر**، **گزینه‌های راه‌اندازی جادوگر** - قراردادهای جادوگر راه‌اندازی ارائه‌دهنده
- **ارائه‌دهنده جاسازی**، **ارائه‌دهنده جاسازی حافظه**، **ارائه‌دهنده واکشی وب**، **تبدیل متن به گفتار** - قراردادهای ارائه‌دهنده ویژه قابلیت
- **کنش‌های نشست**، **پیوست‌های نشست**، **فرافکنی ورودی نشست** - قراردادهای وضعیت نشست تحت مالکیت Plugin
- **نوبت‌های زمان‌بندی‌شده** - فراداده نوبت زمان‌بندی‌شده Plugin و محدوده‌های مهر زمانی
- **قلاب‌های میزبان**، **چرخه حیات زمینه اجرا**، **اثرات جانبی واردکردن زمان‌اجرا**، **اتصال‌های زمان‌اجرا** - قراردادهای چرخه حیات میزبان/زمان‌اجرای Plugin و مرزهای واردکردن
- **وابستگی‌های زمان‌اجرای افزونه** - جای‌گذاری وابستگی زمان‌اجرا برای افزونه‌ها

### زمان اجرا

- پس از تغییر خروجی‌ها یا زیرمسیرهای plugin-sdk
- پس از افزودن یا تغییر یک Plugin کانال یا ارائه‌دهنده
- پس از بازآرایی ثبت یا کشف Plugin

آزمون‌های قرارداد در CI اجرا می‌شوند و به کلیدهای API واقعی نیاز ندارند.

## افزودن رگرسیون‌ها (راهنما)

هنگامی‌که یک مشکل ارائه‌دهنده/مدل کشف‌شده در محیط زنده را رفع می‌کنید:

- در صورت امکان یک رگرسیون ایمن برای CI اضافه کنید (ارائه‌دهنده شبیه‌سازی‌شده/بدلی، یا ثبت تبدیل دقیق شکل درخواست)
- اگر ذاتاً فقط در محیط زنده رخ می‌دهد (محدودیت نرخ، سیاست‌های احراز هویت)، آزمون زنده را محدود و با متغیرهای محیطی مبتنی بر انتخاب صریح نگه دارید
- ترجیحاً کوچک‌ترین لایه‌ای را هدف بگیرید که خطا را تشخیص می‌دهد:
  - خطای تبدیل/بازپخش درخواست ارائه‌دهنده -> آزمون مستقیم مدل‌ها
  - خطای پایپ‌لاین نشست/تاریخچه/ابزار Gateway -> آزمون دود زنده Gateway یا آزمون شبیه‌سازی‌شده Gateway ایمن برای CI
- محافظ پیمایش SecretRef:
  - `src/secrets/exec-secret-ref-id-parity.test.ts` بر اساس فراداده رجیستری (`listSecretTargetRegistryEntries()`) برای هر کلاس SecretRef یک هدف نمونه استخراج می‌کند و سپس بررسی می‌کند که شناسه‌های اجرای دارای بخش پیمایش رد می‌شوند.
  - اگر یک خانواده هدف SecretRef جدید `includeInPlan` را در `src/secrets/target-registry-data.ts` اضافه کردید، `classifyTargetClass` را در آن آزمون به‌روزرسانی کنید. آزمون عمداً برای شناسه‌های هدف طبقه‌بندی‌نشده شکست می‌خورد تا کلاس‌های جدید نتوانند بی‌سروصدا نادیده گرفته شوند.

## مرتبط

- [آزمون در محیط زنده](/fa/help/testing-live)
- [آزمون به‌روزرسانی‌ها و Pluginها](/fa/help/testing-updates-plugins)
- [CI](/fa/ci)
