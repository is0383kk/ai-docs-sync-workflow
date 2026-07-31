---
read_when:
    - اجرای تست‌ها یا رفع اشکال آن‌ها
summary: نحوه اجرای محلی آزمون‌ها (vitest) و زمان استفاده از حالت‌های force/coverage
title: آزمون‌ها
x-i18n:
    generated_at: "2026-07-27T17:07:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 391185703e853bb523e1396eb22da4693d10d47b1644d3b2a51707d329f67dae
    source_path: reference/test.md
    workflow: 16
---

- مجموعه کامل آزمایش (مجموعه‌ها، زنده، Docker): [آزمایش](/fa/help/testing)
- اعتبارسنجی به‌روزرسانی و بسته Plugin: [آزمایش به‌روزرسانی‌ها و Pluginها](/fa/help/testing-updates-plugins)

## پیش‌فرض عامل

نشست‌های عامل فقط برای منبع قابل‌اعتماد و زمانی که نصب وابستگی‌های موجود
آماده باشد، یک یا چند آزمایش متمرکز و بررسی ایستای کم‌هزینه را به‌صورت محلی اجرا می‌کنند. هرگز
ابزارهای مخزن غیرقابل‌اعتماد را به‌صورت محلی اجرا نکنید. مجموعه‌های بزرگ‌تر، گیت‌های تغییرکرده با
توزیع موازی بررسی نوع/لینت، ساخت‌ها، Docker، مسیرهای بسته، E2E، اثبات زنده و
اعتبارسنجی چندسکویی از راه دور و از طریق Crabbox اجرا می‌شوند. اثبات سنگینِ
نگه‌دارنده قابل‌اعتماد به‌طور پیش‌فرض از Blacksmith Testbox استفاده می‌کند. گردش‌کار پیکربندی‌شده Testbox
اعتبارنامه‌ها را بارگذاری می‌کند؛ بنابراین کد مشارکت‌کننده یا فورک غیرقابل‌اعتماد باید در عوض از
CI فورک بدون راز یا AWS Crabbox مستقیم و پاک‌سازی‌شده استفاده کند.

برای کار پیش‌بینی‌شده از پیش گرم نکنید. وقتی
نخستین فرمان سنگین آماده شد، بک‌اند را به‌صورت تنبل دریافت کنید، شناسه `tbx_...` بازگردانده‌شده را برای فرمان‌های سنگین بعدی
دوباره استفاده کنید، در هر اجرا نسخه کاری فعلی را همگام‌سازی کنید و پیش از تحویل آن را متوقف کنید.

پس از نخستین استفاده مجدد موفق، پوشش‌دهنده اثرانگشت مبنا،
وابستگی و گردش‌کار Testbox اجاره را در `.crabbox/testbox-leases/` ثبت می‌کند.
ویرایش‌های صرفاً منبعی همچنان از جعبه گرم‌شده استفاده می‌کنند. تغییر مبنای ادغام، فایل قفل،
ورودی مدیر بسته، پوشش‌دهنده یا گردش‌کار Testbox به‌صورت بسته شکست می‌خورد و به
اجاره‌ای تازه نیاز دارد. هر اجرا همچنان نسخه کاری فعلی را همگام‌سازی می‌کند.
`OPENCLAW_TESTBOX_ALLOW_STALE=1` فقط برای عیب‌یابی عمدی است، نه
اثبات انتشار.

فرمان‌های آزمایش محلی زیر برای گردش‌کارهای انسانی و اثبات محدود عامل هستند.
در دسترس نبودن ارائه‌دهنده راه دور باید گزارش شود؛ این وضعیت اجازه
اجرای بی‌سروصدای یک گیت محلی گسترده را نمی‌دهد.

برای اثبات سنگین غیرقابل‌اعتماد، به‌صورت تنبل با `--provider aws` گرم کنید. هر اجرا باید
`CRABBOX_ENV_ALLOW=CI` را تنظیم کند، `--provider aws --no-hydrate` را ارسال کند و پیش از نصب وابستگی‌ها یا اجرای
آزمایش‌ها از یک `HOME` موقت و تازه راه دور استفاده کند. از اجاره‌ای تازه‌گرم‌شده و مختص همان منبع غیرقابل‌اعتماد استفاده کنید؛ هرگز
اجاره قابل‌اعتماد یا قبلاً بارگذاری‌شده را دوباره استفاده نکنید. یک باینری نصب‌شده و قابل‌اعتماد Crabbox را
از یک نسخه کاری پاک و قابل‌اعتماد `main` راه‌اندازی کنید و فقط PR راه دور را با
`--fresh-pr` دریافت کنید؛ هرگز پوشش‌دهنده یا پیکربندی نسخه کاری غیرقابل‌اعتماد را به‌صورت محلی اجرا نکنید.
`CRABBOX_AWS_INSTANCE_PROFILE` را لغو تنظیم کنید و مگر اینکه مقدار حل‌شده
`aws.instanceProfile` خالی باشد، به‌صورت بسته شکست بخورید. پیش از هر نصب/آزمایش، با ابزارهای قابل‌اعتماد
دارای مسیر مطلق، وجود توکن IMDSv2 را الزامی کنید، ثابت کنید نقطه پایانی اعتبارنامه‌های IAM
404 برمی‌گرداند و تأیید کنید `git rev-parse HEAD` راه دور با SHA کامل
سر PR بازبینی‌شده برابر است. اجاره را به آن SHA مقید کنید و هنگام تغییر سر، آن را متوقف و دوباره گرم کنید.
`scripts/crabbox-untrusted-bootstrap.sh` قابل‌اعتماد را از
`main` پاک در کنار `--fresh-pr` بارگذاری کنید؛ این اسکریپت Node/pnpm سنجاق‌شده را نصب می‌کند، SHA
و سنجاق مدیر بسته را تأیید می‌کند، `HOME` را ایزوله می‌کند، وابستگی‌ها را نصب می‌کند و سپس
آزمایش درخواستی را اجرا می‌کند. اگر کارگزار نتواند نبود نقش یا وجود نداشتن PR راه دور را اثبات کند،
از CI فورک بدون راز استفاده کنید. از `hydrate-github`، `--no-sync` یا
گردش‌کار Testbox بارگذاری‌شده با اعتبارنامه استفاده نکنید.
همه بازنویسی‌های `CRABBOX_TAILSCALE*` را لغو تنظیم کنید، `--network public
--tailscale=false` را اجباری کنید، پرچم‌های گره خروج/LAN را پاک کنید و پیش از بارگذاری هر اسکریپت، الزام کنید `crabbox inspect`
شبکه عمومی بدون وضعیت Tailscale را گزارش دهد.

## ترتیب معمول محلی

1. `pnpm test:changed` برای اثبات Vitest با دامنه تغییرکرده.
2. `pnpm test <path-or-filter>` برای یک فایل، پوشه یا هدف صریح.
3. `pnpm test` فقط زمانی که عمداً به مجموعه کامل محلی Vitest نیاز دارید.

در یک درخت کاری Codex یا نسخه کاری پیوندی/تنک، عامل‌ها از اجرای مستقیم محلی
`pnpm test*` / `pnpm check*` / `pnpm crabbox:run` پرهیز می‌کنند:

- اثبات متمرکز محدود با وابستگی‌های آماده:
  `node scripts/run-vitest.mjs <path-or-filter>`.
- بررسی تغییرکرده با طبقه‌بندی در ابتدا: `node scripts/check-changed.mjs`؛ طرح‌های صرفاً مستندات،
  بدون تغییر و فراداده کوچک، هنگام آماده بودن وابستگی‌ها محلی می‌مانند،
  درحالی‌که طرح‌های سنگین یا فاقد وابستگی به Testbox واگذار می‌شوند.
- اثبات گسترده صریح با اجاره نگه‌داشته‌شده: `node scripts/crabbox-wrapper.mjs run --provider blacksmith-testbox ... -- env OPENCLAW_CHECK_CHANGED_REMOTE_CHILD=1 OPENCLAW_CHANGED_LANES_RAW_SYNC=1 corepack pnpm check:changed` تا pnpm درون Testbox اجرا شود.
- `exitCode` نهایی پوشش‌دهنده و JSON زمان‌بندی، نتیجه فرمان هستند. یک اجرای واگذارشده Blacksmith در GitHub Actions ممکن است پس از فرمان موفق SSH، `cancelled` را نشان دهد، زیرا Testbox از بیرون کنش زنده‌نگه‌دار متوقف می‌شود؛ پیش از شکست تلقی کردن آن، خلاصه پوشش‌دهنده و خروجی فرمان را بررسی کنید.
- `OPENCLAW_HEAVY_CHECK_LOCK_SCOPE=worktree <local-heavy-check command>`: سری‌سازی بررسی‌های سنگین را برای فرمان‌هایی مانند `pnpm check:changed` و `pnpm test ...` هدفمند، به‌جای پوشه مشترک Git در درخت کاری فعلی نگه می‌دارد. فقط زمانی از آن استفاده کنید که عمداً بررسی‌های مستقل را در درخت‌های کاری پیوندی روی میزبان‌های محلی پرظرفیت اجرا می‌کنید.

## فرمان‌های اصلی

اجرای پوشش‌دهنده آزمایش با یک خلاصه کوتاه `[test] passed|failed|skipped ... in ...` پایان می‌یابد؛ خط مدت‌زمان خود Vitest همچنان جزئیات هر شارد است.

| فرمان                                           | کاری که انجام می‌دهد                                                                                                                                                                                                                                                                                                                                                    |
| ------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test`                                       | اهداف صریح فایل/پوشه از مسیرهای محدوده‌بندی‌شده Vitest عبور می‌کنند. اجراهای بدون هدف، اثبات مجموعه کامل هستند: گروه‌های ثابت شارد برای اجرای موازی محلی به پیکربندی‌های برگ گسترش می‌یابند و توزیع موازی مورد انتظار شارد پیش از شروع چاپ می‌شود. گروه افزونه همیشه به‌جای یک فرایند عظیم پروژه ریشه، به پیکربندی‌های شارد مجزا برای هر افزونه گسترش می‌یابد.           |
| `pnpm test:changed`                               | اجرای هوشمند و کم‌هزینه آزمایش‌های تغییرکرده: اهداف دقیق از ویرایش‌های مستقیم آزمایش، فایل‌های هم‌خانواده `*.test.ts`، نگاشت‌های صریح منبع و گراف واردسازی محلی. تغییرات گسترده پیکربندی/بسته نادیده گرفته می‌شوند، مگر اینکه به آزمایش‌های دقیقی نگاشت شوند.                                                                                                                               |
| `OPENCLAW_TEST_CHANGED_BROAD=1 pnpm test:changed` | اجرای صریح و گسترده آزمایش‌های تغییرکرده؛ زمانی استفاده کنید که ویرایش ابزار آزمایش/پیکربندی/بسته باید به رفتار گسترده‌تر آزمایش تغییرکرده Vitest بازگردد.                                                                                                                                                                                                                        |
| `pnpm test:force`                                 | پورت پیکربندی‌شده Gateway متعلق به OpenClaw (پیش‌فرض `18789`) را آزاد می‌کند، سپس مجموعه کامل را با یک پورت Gateway ایزوله اجرا می‌کند تا آزمایش‌های سرور با نمونه در حال اجرا تداخل نداشته باشند.                                                                                                                                                                                    |
| `pnpm test:coverage`                              | یک گزارش پوشش اطلاعاتی V8 برای مسیر واحد پیش‌فرض (`vitest.unit.config.ts`) تولید می‌کند؛ هیچ آستانه پوششی اعمال نمی‌شود.                                                                                                                                                                                                                             |
| `pnpm test:coverage:changed`                      | پوشش واحد فقط برای فایل‌هایی که از `origin/main` تغییر کرده‌اند.                                                                                                                                                                                                                                                                                                       |
| `pnpm changed:lanes`                              | مسیرهای معماری فعال‌شده با تفاوت نسبت به `origin/main` را نشان می‌دهد.                                                                                                                                                                                                                                                                                      |
| `pnpm check:changed`                              | پیش از انتخاب اجرا، مسیرهای تغییرکرده را طبقه‌بندی می‌کند. طرح‌های صرفاً مستندات، بدون تغییر و فراداده کوچک هنگام آماده بودن وابستگی‌ها محلی می‌مانند؛ طرح‌های دارای توزیع موازی بررسی نوع/لینت، دیگر مسیرهای سنگین یا وابستگی‌های محلی مفقود، خارج از CI به Crabbox/Testbox واگذار می‌شوند. Vitest را اجرا نمی‌کند؛ برای اثبات آزمایش از `pnpm test:changed` یا `pnpm test <target>` استفاده کنید. |

## وضعیت مشترک آزمایش و ابزارهای کمکی فرایند

- `src/test-utils/openclaw-test-state.ts`: زمانی در Vitest استفاده کنید که یک آزمایش به `HOME`، `OPENCLAW_STATE_DIR`، `OPENCLAW_CONFIG_PATH`، فیکسچر پیکربندی، فضای کاری، پوشه عامل یا مخزن پروفایل احراز هویت ایزوله نیاز دارد.
- `pnpm test:env-mutations:report`: گزارش غیرمسدودکننده آزمایش‌ها/ابزارهایی که مستقیماً `HOME`، `OPENCLAW_STATE_DIR`، `OPENCLAW_CONFIG_PATH`، `OPENCLAW_WORKSPACE_DIR` یا کلیدهای محیطی مرتبط را تغییر می‌دهند. برای یافتن نامزدهای مهاجرت به ابزار کمکی وضعیت مشترک آزمایش از آن استفاده کنید.
- `test/helpers/openclaw-test-instance.ts`: آزمایش‌های E2E در سطح فرایند که به Gateway در حال اجرا، محیط CLI، ثبت گزارش و پاک‌سازی در یک مکان نیاز دارند.
- مسیرهای E2E در Docker/Bash که `scripts/lib/docker-e2e-image.sh` را منبع می‌کنند، می‌توانند `docker_e2e_test_state_shell_b64 <label> <scenario>` را به کانتینر ارسال و آن را با `scripts/lib/openclaw-e2e-instance.sh` رمزگشایی کنند؛ اسکریپت‌های چندخانه‌ای می‌توانند `docker_e2e_test_state_function_b64` را ارسال کنند و در هر جریان `openclaw_test_state_create <label> <scenario>` را فراخوانی کنند. `node scripts/lib/openclaw-test-state.mjs -- create --label <name> --scenario <name> --env-file <path> --json` یک فایل محیط میزبان قابل منبع‌گیری می‌نویسد (`--` پیش از `create` مانع از آن می‌شود که زمان‌اجراهای جدیدتر Node، `--env-file` را پرچم Node تلقی کنند). مسیرهایی که Gateway راه‌اندازی می‌کنند می‌توانند `scripts/lib/openclaw-e2e-instance.sh` را برای حل نقطه ورود، راه‌اندازی شبیه‌سازی‌شده OpenAI، اجرای پیش‌زمینه/پس‌زمینه، کاوش‌های آمادگی، صدور محیط وضعیت، تخلیه گزارش‌ها و پاک‌سازی فرایند منبع کنند.

## مسیرهای رابط کاربری کنترل، TUI و افزونه

- **E2E شبیه‌سازی‌شده رابط کنترل:** `pnpm test:ui:e2e` مسیر Vitest + Playwright را اجرا می‌کند که رابط کنترل Vite را راه‌اندازی کرده و یک صفحه واقعی Chromium را در برابر WebSocket شبیه‌سازی‌شده Gateway هدایت می‌کند. آزمون‌ها در `ui/src/**/*.e2e.test.ts` قرار دارند؛ شبیه‌سازی‌ها/کنترل‌های مشترک در `ui/src/test-helpers/control-ui-e2e.ts` قرار دارند. `pnpm test:e2e` این مسیر را شامل می‌شود. اجرای عامل‌ها، از جمله اثبات هدفمند، به‌طور پیش‌فرض در Testbox/Crabbox انجام می‌شود؛ از `node scripts/run-vitest.mjs run --config test/vitest/vitest.ui-e2e.config.ts --configLoader runner ui/src/ui/e2e/chat-flow.e2e.test.ts` فقط برای بازگشت صریح به اجرای محلی استفاده کنید.
- **آزمون‌های PTY در TUI:** `node scripts/run-vitest.mjs run --config test/vitest/vitest.tui-pty.config.ts` مسیر سریع PTY با بک‌اند جعلی را اجرا می‌کند. `OPENCLAW_TUI_PTY_INCLUDE_LOCAL=1` یا `pnpm tui:pty:test:watch --mode local` آزمون دود کندتر `tui --local` را اجرا می‌کند که فقط نقطه پایانی مدل خارجی را شبیه‌سازی می‌کند. متن قابل‌مشاهده پایدار یا فراخوانی‌های فیکسچر را بررسی کنید، نه اسنپ‌شات‌های خام ANSI.
- `pnpm test:extensions` و `pnpm test extensions` همه شاردهای افزونه/Plugin را اجرا می‌کنند. Pluginهای سنگین کانال، Plugin مرورگر و OpenAI به‌صورت شاردهای اختصاصی اجرا می‌شوند؛ سایر گروه‌های Plugin به‌صورت دسته‌ای باقی می‌مانند. `pnpm test extensions/<id>` مسیر یک Plugin همراه را اجرا می‌کند.
- فایل‌های منبع دارای آزمون هم‌جوار، پیش از بازگشت به الگوهای گسترده‌تر دایرکتوری، به همان آزمون هم‌جوار نگاشت می‌شوند. ویرایش ابزارهای کمکی در `src/channels/plugins/contracts/test-helpers`، `src/plugin-sdk/test-helpers` و `src/plugins/contracts` از گراف محلی import استفاده می‌کند تا وقتی مسیر وابستگی دقیق است، به‌جای اجرای گسترده همه شاردها، آزمون‌های importکننده اجرا شوند.
- اهداف دایرکتوری قرارداد میان مسیرهای قرارداد خود توزیع می‌شوند: `pnpm test src/channels/plugins/contracts` چهار پیکربندی قرارداد کانال را اجرا می‌کند و `pnpm test src/plugins/contracts` پیکربندی قراردادهای Plugin را اجرا می‌کند، زیرا پروژه‌های عمومی `channels`/`plugins`، `contracts/**` را مستثنا می‌کنند.
- `auto-reply` به سه پیکربندی اختصاصی (`core`، `top-level`، `reply`) تقسیم می‌شود تا زیرساخت پاسخ بر آزمون‌های سبک‌تر وضعیت/توکن/ابزار کمکی در سطح بالا غالب نشود.
- فایل‌های آزمون منتخب `plugin-sdk` و `commands` از مسیرهای سبک اختصاصی عبور می‌کنند که فقط `test/setup.ts` را نگه می‌دارند و موارد سنگین از نظر زمان اجرا را در مسیرهای موجودشان باقی می‌گذارند.
- پیکربندی پایه Vitest به‌طور پیش‌فرض از `pool: "threads"` و `isolate: false` استفاده می‌کند و اجراکننده مشترکِ غیراجداسازی‌شده در همه پیکربندی‌های مخزن فعال است.
- `pnpm test:channels`، `vitest.channels.config.ts` را اجرا می‌کند.

## Gateway و E2E

- یکپارچه‌سازی Gateway اختیاری است: `OPENCLAW_TEST_INCLUDE_GATEWAY=1 pnpm test` یا `pnpm test:gateway`.
- `pnpm test:e2e`: تجمیع E2E مخزن = `pnpm test:e2e:gateway && pnpm test:ui:e2e`.
- `pnpm test:e2e:gateway`: آزمون‌های دود سرتاسری Gateway (جفت‌سازی چندنمونه‌ای WS/HTTP/Node). به‌طور پیش‌فرض از `threads` + `isolate: false` با workerهای تطبیقی در `vitest.e2e.config.ts` استفاده می‌کند؛ با `OPENCLAW_E2E_WORKERS=<n>` تنظیم کنید و برای گزارش‌های پرجزئیات از `OPENCLAW_E2E_VERBOSE=1` استفاده کنید.
- `pnpm test:live`: آزمون‌های زنده ارائه‌دهنده (Claude/Minimax/DeepSeek/z.ai/و غیره، مشروط به `*.live.test.ts`). برای خارج‌شدن از حالت ردشده، کلیدهای API و `LIVE=1` (یا `OPENCLAW_LIVE_TEST=1`) لازم است؛ خروجی پرجزئیات با `OPENCLAW_LIVE_TEST_QUIET=0`.

## مجموعه کامل Docker (`pnpm test:docker:all`)

تصویر مشترک آزمون زنده را می‌سازد، OpenClaw را یک‌بار به‌صورت tarball در npm بسته‌بندی می‌کند، یک تصویر اجراکننده ساده Node/Git و نیز تصویری عملیاتی را که آن tarball را در `/app` نصب می‌کند می‌سازد/دوباره استفاده می‌کند، سپس مسیرهای دود Docker را از طریق زمان‌بند وزن‌دار اجرا می‌کند. `scripts/package-openclaw-for-docker.mjs` تنها بسته‌بند محلی/CI است و tarball به‌همراه `dist/postinstall-inventory.json` را پیش از مصرف توسط Docker اعتبارسنجی می‌کند.

- تصویر ساده (`OPENCLAW_DOCKER_E2E_BARE_IMAGE`): مسیرهای نصب‌کننده/به‌روزرسانی/وابستگی Plugin؛ به‌جای منابع کپی‌شده مخزن، tarball ازپیش‌ساخته‌شده را mount می‌کند.
- تصویر عملیاتی (`OPENCLAW_DOCKER_E2E_FUNCTIONAL_IMAGE`): مسیرهای عملکرد عادی برنامه ساخته‌شده.
- تعریف مسیرها: `scripts/lib/docker-e2e-scenarios.mjs`. برنامه‌ریز: `scripts/lib/docker-e2e-plan.mjs`. اجراکننده: `scripts/test-docker-all.mjs`.
- `node scripts/test-docker-all.mjs --plan-json` بدون ساختن یا اجرای Docker، برنامه CI متعلق به زمان‌بند (مسیرها، انواع تصویر، نیازهای بسته/تصویر زنده، سناریوهای وضعیت و بررسی اعتبارنامه‌ها) را تولید می‌کند.

گزینه‌های تنظیم زمان‌بندی (متغیرهای محیطی، مقادیر پیش‌فرض در پرانتز):

| متغیر محیطی                                                                                                         | پیش‌فرض             | هدف                                                                                                                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------------- | ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DOCKER_ALL_PARALLELISM`                                                                               | 10                  | جایگاه‌های پردازش.                                                                                                                                                                                                                                                                             |
| `OPENCLAW_DOCKER_ALL_TAIL_PARALLELISM`                                                                          | 10                  | مخزن انتهایی حساس به ارائه‌دهنده.                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_LIVE_LIMIT`                                                                                | 9                   | سقف مسیر سنگین ارائه‌دهنده زنده.                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_NPM_LIMIT`                                                                                 | 5                   | سقف مسیر منابع npm.                                                                                                                                                                                                                                                                     |
| `OPENCLAW_DOCKER_ALL_SERVICE_LIMIT`                                                                             | 7                   | سقف مسیر منابع سرویس.                                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_LIVE_CLAUDE_LIMIT` / `_CODEX_LIMIT` / `_GEMINI_LIMIT` / `_DROID_LIMIT` / `_OPENCODE_LIMIT` | 4                   | سقف مسیرهای سنگین برای هر ارائه‌دهنده.                                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_LIVE_OPENAI_LIMIT` / `_TELEGRAM_LIMIT`                                                     | 1                   | سقف‌های محدودتر برای هر ارائه‌دهنده.                                                                                                                                                                                                                                                                |
| `OPENCLAW_DOCKER_ALL_WEIGHT_LIMIT` / `OPENCLAW_DOCKER_ALL_DOCKER_LIMIT`                                         | -                   | بازنویسی برای میزبان‌های بزرگ‌تر.                                                                                                                                                                                                                                                                 |
| `OPENCLAW_DOCKER_ALL_START_STAGGER_MS`                                                                          | 2000                | تأخیر میان شروع مسیرها؛ از هجوم عملیات ایجاد در daemon محلی Docker جلوگیری می‌کند.                                                                                                                                                                                                                       |
| `OPENCLAW_DOCKER_ALL_LANE_TIMEOUT_MS`                                                                           | 7,200,000 (120 دقیقه) | مهلت بازگشتی هر مسیر؛ مسیرهای زنده/انتهایی منتخب سقف‌های سخت‌گیرانه‌تری دارند.                                                                                                                                                                                                                      |
| `OPENCLAW_DOCKER_ALL_LIVE_RETRIES`                                                                              | 1                   | تعداد تلاش‌های مجدد برای شکست‌های گذرای ارائه‌دهنده زنده.                                                                                                                                                                                                                                              |
| `OPENCLAW_DOCKER_ALL_DRY_RUN`                                                                                   | خاموش                 | مانیفست مسیرها را بدون اجرای Docker چاپ می‌کند.                                                                                                                                                                                                                                            |
| `OPENCLAW_DOCKER_ALL_STATUS_INTERVAL_MS`                                                                        | 30000               | فاصله چاپ وضعیت مسیرهای فعال.                                                                                                                                                                                                                                                         |
| `OPENCLAW_DOCKER_ALL_TIMINGS`                                                                                   | روشن                  | استفاده مجدد از `.artifacts/docker-tests/lane-timings.json` برای ترتیب طولانی‌ترین-اول؛ برای غیرفعال‌سازی روی `0` تنظیم کنید.                                                                                                                                                                                       |
| `OPENCLAW_DOCKER_ALL_LIVE_MODE`                                                                                 | -                   | `skip` فقط برای مسیرهای قطعی/محلی و `only` فقط برای مسیرهای ارائه‌دهنده زنده. نام‌های مستعار: `pnpm test:docker:local:all`، `pnpm test:docker:live:all`. حالت فقط‌زنده مسیرهای زنده اصلی و انتهایی را در یک مخزن طولانی‌ترین-اول ادغام می‌کند تا سطل‌های ارائه‌دهنده کارهای Claude/Codex/Gemini را کنار هم بسته‌بندی کنند. |
| `OPENCLAW_LIVE_CLI_BACKEND_SETUP_TIMEOUT_SECONDS`                                                               | 180                 | مهلت راه‌اندازی Docker در بک‌اند CLI.                                                                                                                                                                                                                                                          |

الگوی متغیر محیطی برای سقف منابع `OPENCLAW_DOCKER_ALL_<RESOURCE>_LIMIT` است (نام منبع با حروف بزرگ و نویسه‌های غیرالفبایی‌عددی تبدیل‌شده به `_`).

رفتارهای دیگر: اجراکننده به‌طور پیش‌فرض Docker را پیش‌بررسی می‌کند، کانتینرهای قدیمی E2E مربوط به OpenClaw را پاک می‌کند، کش ابزارهای CLI ارائه‌دهنده را میان laneهای سازگار به اشتراک می‌گذارد و پس از نخستین شکست، زمان‌بندی laneهای تجمیع‌شده جدید را متوقف می‌کند، مگر اینکه `OPENCLAW_DOCKER_ALL_FAIL_FAST=0` تنظیم شده باشد. اگر یک lane از سقف مؤثر وزن/منابع در میزبانی با موازی‌سازی کم فراتر رود، همچنان می‌تواند از یک pool خالی آغاز شود و تا زمان آزادسازی ظرفیت، به‌تنهایی اجرا شود. گزارش‌های هر lane، `summary.json`، `failures.json` و زمان‌بندی فازها در `.artifacts/docker-tests/<run-id>/` نوشته می‌شوند؛ برای بررسی laneهای کند از `pnpm test:docker:timings <summary.json>` و برای چاپ فرمان‌های ارزان اجرای مجدد هدفمند از `pnpm test:docker:rerun <run-id|summary.json|failures.json>` استفاده کنید.

### laneهای شاخص Docker

| فرمان                                                                     | موارد مورد راستی‌آزمایی                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `pnpm test:docker:browser-cdp-snapshot`                                     | کانتینر E2E منبع مبتنی بر Chromium با CDP خام و Gateway ایزوله؛ snapshotهای نقش CDP در `browser doctor --deep` شامل URL پیوندها، عناصر قابل‌کلیک ارتقایافته با نشانگر، ارجاع‌های iframe و فراداده فریم هستند.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| `pnpm test:docker:skill-install`                                            | tarball بسته‌بندی‌شده را با `skills.install.allowUploadedArchives: false` در یک اجراکننده Docker خام نصب می‌کند، slug فعلی یک skill را از جست‌وجوی زنده ClawHub به‌دست می‌آورد، آن را از طریق `openclaw skills install` نصب می‌کند و `SKILL.md`، `.clawhub/origin.json`، `.clawhub/lock.json` و `skills info --json` را راستی‌آزمایی می‌کند.                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `pnpm test:docker:live-cli-backend:claude`، `:claude:resume`، `:claude:mcp` | کاوش‌های زنده متمرکز backend مربوط به CLI؛ Gemini دارای aliasهای متناظر `:resume` و `:mcp` است.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `pnpm test:docker:openwebui`                                                | OpenClaw و Open WebUI کانتینری‌شده: ورود به سیستم، بررسی `/api/models` و اجرای یک گفت‌وگوی واقعی پروکسی‌شده از طریق `/api/chat/completions`. به یک کلید مدل زنده قابل‌استفاده نیاز دارد و یک image خارجی را دریافت می‌کند؛ انتظار نمی‌رود مانند مجموعه‌های unit/e2e در CI پایدار باشد.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `pnpm test:docker:mcp-channels`                                             | کانتینر Gateway ازپیش‌مقداردهی‌شده به‌همراه کانتینر client که `openclaw mcp serve` را راه‌اندازی می‌کند: کشف مکالمه مسیریابی‌شده، خواندن transcript، فراداده پیوست، رفتار صف رویداد زنده، مسیریابی ارسال خروجی و اعلان‌های کانال و مجوز به‌سبک Claude از طریق پل واقعی stdio (assertion فریم‌های خام stdio MCP را مستقیماً می‌خواند).                                                                                                                                                                                                                                                                                                                                                                                                               |
| `pnpm test:docker:upgrade-survivor`                                         | tarball بسته‌بندی‌شده را روی fixture کثیف یک کاربر قدیمی نصب می‌کند، به‌روزرسانی بسته و doctor غیرتعاملی را بدون کلیدهای زنده ارائه‌دهنده/کانال اجرا می‌کند، یک Gateway حلقه‌بازگشتی را راه‌اندازی می‌کند و بررسی می‌کند که عامل‌ها، پیکربندی کانال، فهرست‌های مجاز Plugin، فایل‌های workspace/session، وضعیت قدیمی وابستگی Plugin منسوخ، راه‌اندازی و وضعیت RPC حفظ شوند.                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `pnpm test:docker:published-upgrade-survivor`                               | به‌طور پیش‌فرض `openclaw@latest` را نصب می‌کند، فایل‌های واقع‌گرایانه کاربر موجود را مقداردهی می‌کند، با یک دستورالعمل ازپیش‌ساخته‌شده `openclaw config set` پیکربندی می‌کند، به tarball بسته‌بندی‌شده به‌روزرسانی می‌کند، doctor غیرتعاملی را اجرا می‌کند، `.artifacts/upgrade-survivor/summary.json` را می‌نویسد و `/healthz`، `/readyz` و وضعیت RPC را بررسی می‌کند. با `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPEC` بازنویسی کنید، با `OPENCLAW_UPGRADE_SURVIVOR_BASELINE_SPECS` یک ماتریس را گسترش دهید، یا با `OPENCLAW_UPGRADE_SURVIVOR_SCENARIOS=reported-issues` fixtureهای سناریو اضافه کنید (شامل `configured-plugin-installs` و `stale-source-plugin-shadow`). Package Acceptance این موارد را به‌شکل `published_upgrade_survivor_baseline(s)` / `_scenarios` ارائه می‌کند و توکن‌های meta مانند `last-stable-4` یا `all-since-2026.4.23` را resolve می‌کند. |
| `pnpm test:docker:update-migration`                                         | چارچوب آزمون بقای ارتقا از نسخه منتشرشده در سناریوی `plugin-deps-cleanup` که به‌طور پیش‌فرض از `openclaw@2026.4.23` آغاز می‌شود. workflow مربوط به `Update Migration` این مورد را با `baselines=all-since-2026.4.23` گسترش می‌دهد تا پاک‌سازی وابستگی Plugin پیکربندی‌شده را خارج از Full Release CI اثبات کند.                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `pnpm test:docker:plugins`                                                  | آزمون دود نصب/به‌روزرسانی برای مسیر محلی، `file:`، بسته‌های رجیستری npm با وابستگی‌های hoist‌شده، ارجاع‌های متحرک git، fixtureهای ClawHub، به‌روزرسانی‌های marketplace و فعال‌سازی/بازرسی بسته Claude.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

## گیت محلی PR

برای بررسی‌های محلی گیت/فرود PR، اجرا کنید:

- `pnpm check:changed`
- `pnpm check`
- `pnpm check:test-types`
- `pnpm build`
- `pnpm test`
- `pnpm check:docs`

اگر `pnpm test` روی یک میزبان پرترافیک دچار شکست ناپایدار شد، پیش از درنظرگرفتن آن به‌عنوان پسرفت، یک‌بار دیگر اجرا کنید و سپس با `pnpm test <path/to/test>` آن را ایزوله کنید. برای میزبان‌های دارای محدودیت حافظه:

- `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`
- `OPENCLAW_VITEST_FS_MODULE_CACHE_PATH=/tmp/openclaw-vitest-cache pnpm test:changed`

## ابزارهای کارایی آزمون

- `pnpm test:perf:imports`: گزارش‌دهی مدت import و تفکیک import در Vitest را فعال می‌کند، درحالی‌که برای هدف‌های صریح فایل/دایرکتوری همچنان از مسیریابی lane با دامنه محدود استفاده می‌کند. `pnpm test:perf:imports:changed` همین پروفایل‌گیری را به فایل‌های تغییرکرده از زمان `origin/main` محدود می‌کند.
- `pnpm test:perf:changed:bench -- --ref <git-ref>` مسیر حالت تغییرکرده مسیریابی‌شده را برای همان diff ثبت‌شده git در برابر اجرای بومی پروژه ریشه benchmark می‌کند؛ `pnpm test:perf:changed:bench -- --worktree` مجموعه تغییرات worktree فعلی را بدون commit قبلی benchmark می‌کند.
- `pnpm test:perf:profile:main` یک پروفایل CPU برای thread اصلی Vitest می‌نویسد (`.artifacts/vitest-main-profile`)؛ `pnpm test:perf:profile:runner` پروفایل‌های CPU و heap را برای اجراکننده unit می‌نویسد (`.artifacts/vitest-runner-profile`).
- `pnpm test:perf:groups --full-suite --allow-failures --output .artifacts/test-perf/baseline-before.json`: هر پیکربندی نهایی Vitest در مجموعه کامل را به‌صورت ترتیبی اجرا می‌کند و داده‌های مدت‌زمان گروه‌بندی‌شده را به‌همراه artifactهای JSON/log برای هر پیکربندی می‌نویسد. گزارش‌های مجموعه کامل به‌طور پیش‌فرض فایل‌ها را ایزوله می‌کنند تا گراف‌های ماژول حفظ‌شده و مکث‌های GC ناشی از فایل‌های قبلی به assertionهای بعدی منظور نشوند؛ فقط هنگامی `-- --no-isolate` را ارائه کنید که عمداً در حال پروفایل‌گیری انباشت worker مشترک هستید. Test Performance Agent پیش از تلاش برای اصلاح آزمون‌های کند، از این مورد به‌عنوان مبنا استفاده می‌کند. `pnpm test:perf:groups:compare .artifacts/test-perf/baseline-before.json .artifacts/test-perf/after-agent.json` گزارش‌های گروه‌بندی‌شده را پس از یک تغییر متمرکز بر کارایی مقایسه می‌کند.
- اجراهای shard مربوط به مجموعه کامل، افزونه و الگوی include، داده‌های زمان‌بندی محلی را در `.artifacts/vitest-shard-timings.json` به‌روزرسانی می‌کنند؛ اجراهای بعدی کل پیکربندی از این زمان‌بندی‌ها برای متعادل‌کردن shardهای کند و سریع استفاده می‌کنند. shardهای CI با الگوی include نام shard را به کلید زمان‌بندی اضافه می‌کنند، در نتیجه زمان‌بندی shardهای فیلترشده بدون جایگزینی داده‌های زمان‌بندی کل پیکربندی قابل‌مشاهده می‌ماند. برای نادیده‌گرفتن artifact زمان‌بندی محلی، `OPENCLAW_TEST_PROJECTS_TIMINGS=0` را تنظیم کنید.

## بنچمارک‌ها

<Accordion title="تأخیر مدل (scripts/bench-model.ts)">

```bash
pnpm tsx scripts/bench-model.ts --runs 10
```

متغیرهای محیطی اختیاری: `MINIMAX_API_KEY`، `MINIMAX_BASE_URL`، `MINIMAX_MODEL`، `ANTHROPIC_API_KEY`. پرامپت پیش‌فرض: «با یک واژه پاسخ دهید: ok. بدون نشانه‌گذاری یا متن اضافی.»

</Accordion>

<Accordion title="راه‌اندازی CLI (scripts/bench-cli-startup.ts)">

```bash
pnpm test:startup:bench
pnpm test:startup:bench:smoke
pnpm test:startup:bench:save
pnpm test:startup:bench:update
pnpm test:startup:bench:check
pnpm tsx scripts/bench-cli-startup.ts --runs 12
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --case gatewayStatus --runs 3
pnpm tsx scripts/bench-cli-startup.ts --entry openclaw.mjs --entry-secondary dist/entry.js --preset all
```

پیش‌تنظیم‌ها:

- `startup`: `--version`، `--help`، `health`، `health --json`، `status --json`، `status`
- `real`: `health`، `status`، `status --json`، `sessions`، `sessions --json`، `tasks --json`، `tasks list --json`، `tasks audit --json`، `agents list --json`، `gateway status`، `gateway status --json`، `gateway health --json`، `config get gateway.port`
- `all`: ترکیب هر دو پیش‌تنظیم

خروجی شامل `sampleCount`، میانگین، p50، p95، کمینه/بیشینه، توزیع کد خروج/سیگنال و بیشینه RSS برای هر فرمان است. `--cpu-prof-dir` / `--heap-prof-dir` برای هر اجرا پروفایل‌های V8 می‌نویسند.

خروجی ذخیره‌شده: `pnpm test:startup:bench:smoke` در `.artifacts/cli-startup-bench-smoke.json` می‌نویسد؛ `pnpm test:startup:bench:save` در `.artifacts/cli-startup-bench-all.json` می‌نویسد (`runs=5 warmup=1`). فیکسچر ثبت‌شده در مخزن: `test/fixtures/cli-startup-bench.json`، که با `pnpm test:startup:bench:update` به‌روزرسانی و با `pnpm test:startup:bench:check` مقایسه می‌شود.

</Accordion>

<Accordion title="راه‌اندازی Gateway (scripts/bench-gateway-startup.ts)">

به‌طور پیش‌فرض از ورودی CLI ساخته‌شده در `dist/entry.js` استفاده می‌کند؛ ابتدا `pnpm build` را اجرا کنید. برای اندازه‌گیری اجراکنندهٔ منبع، به‌جای آن `--entry scripts/run-node.mjs` را ارسال کنید و نتایجش را از خط‌مبناهای ورودی ساخته‌شده جدا نگه دارید.

```bash
pnpm test:startup:gateway -- --runs 5 --warmup 1
pnpm test:startup:gateway -- --case skipChannels --case fiftyPlugins --runs 5
node --import tsx scripts/bench-gateway-startup.ts --case default --runs 5 --output .artifacts/gateway-startup.json
```

شناسه‌های حالت: `default`، `skipChannels` (راه‌اندازی کانال نادیده گرفته می‌شود)، `oneInternalHook`، `allInternalHooks`، `fiftyPlugins` (50 پلاگین مانیفست)، `fiftyStartupLazyPlugins` (50 پلاگین مانیفست با راه‌اندازی تنبل).

خروجی شامل نخستین خروجی فرایند، `/healthz`، `/readyz`، زمان لاگ گوش‌دادن HTTP، زمان لاگ آماده‌شدن Gateway، زمان CPU، نسبت هستهٔ CPU، بیشینه RSS، هیپ، سنجه‌های ردگیری راه‌اندازی، تأخیر حلقهٔ رویداد و سنجه‌های جزئیات جدول جست‌وجوی پلاگین است. اسکریپت `OPENCLAW_GATEWAY_STARTUP_TRACE=1` را در محیط Gateway فرزند تنظیم می‌کند.

`/healthz` نشان‌دهندهٔ زنده‌بودن است (سرور HTTP می‌تواند پاسخ دهد). `/readyz` نشان‌دهندهٔ آمادگی قابل‌استفاده است (سایدکارهای پلاگین راه‌اندازی، کانال‌ها و کارهای پس از اتصالِ حیاتی برای آمادگی به وضعیت پایدار رسیده‌اند). هوک‌های راه‌اندازی به‌صورت ناهمگام ارسال می‌شوند و بخشی از تضمین آمادگی نیستند. زمان لاگ آمادگی، مُهر زمانی داخلی Gateway است که برای انتساب در سمت فرایند کاربرد دارد، اما جایگزین پروب خارجی `/readyz` نیست.

هنگام مقایسهٔ تغییرات، از خروجی JSON یا `--output` استفاده کنید. تنها زمانی از `--cpu-prof-dir` استفاده کنید که خروجی ردگیری به کارهای واردکردن، کامپایل یا پردازش‌های محدودشده به CPU اشاره کند که زمان‌بندی مرحله‌ها به‌تنهایی قادر به توضیح آن‌ها نیست.

</Accordion>

<Accordion title="راه‌اندازی مجدد Gateway (scripts/bench-gateway-restart.ts)">

فقط macOS و Linux (برای راه‌اندازی مجدد درون‌فرایندی از SIGUSR1 استفاده می‌کند؛ در Windows بلافاصله با شکست مواجه می‌شود). پیش‌فرض ورودی ساخته‌شده و بازنویسی `--entry scripts/run-node.mjs` همانند راه‌اندازی Gateway در بالا است.

```bash
pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5
pnpm test:restart:gateway -- --case default --runs 3 --restarts 3 --warmup 1
```

شناسه‌های حالت: `skipChannels`، `skipChannelsAcpxProbe` (پروب راه‌اندازی ACPX روشن)، `skipChannelsNoAcpxProbe` (پروب خاموش)، `default`، `fiftyPlugins`.

خروجی شامل `/healthz` بعدی، `/readyz` بعدی، زمان ازکارافتادگی، زمان‌بندی آمادگی پس از راه‌اندازی مجدد، CPU، RSS، سنجه‌های ردگیری راه‌اندازی برای فرایند جایگزین و سنجه‌های ردگیری راه‌اندازی مجدد برای مدیریت سیگنال، تخلیهٔ کار فعال، مرحله‌های بستن، شروع بعدی، زمان‌بندی آمادگی و اسنپ‌شات‌های حافظه است. اسکریپت `OPENCLAW_GATEWAY_STARTUP_TRACE=1` و `OPENCLAW_GATEWAY_RESTART_TRACE=1` را تنظیم می‌کند.

هنگامی از این بنچمارک استفاده کنید که تغییری بر سیگنال‌دهی راه‌اندازی مجدد، کنترل‌گرهای بستن، راه‌اندازی پس از راه‌اندازی مجدد، خاموش‌کردن سایدکار، تحویل سرویس یا آمادگی پس از راه‌اندازی مجدد اثر می‌گذارد. برای جداسازی سازوکارهای Gateway از راه‌اندازی کانال، با `skipChannels` شروع کنید؛ تنها پس از آنکه حالت محدود مسیر راه‌اندازی مجدد را توضیح داد، از `default` یا حالت‌های سنگین از نظر پلاگین استفاده کنید. سنجه‌های ردگیری سرنخ‌هایی برای انتساب‌اند، نه حکم نهایی — تغییر راه‌اندازی مجدد را بر پایهٔ چندین نمونه، محدودهٔ مالک متناظر، رفتار `/healthz`/`/readyz` و قرارداد راه‌اندازی مجدد قابل‌مشاهده برای کاربر ارزیابی کنید.

</Accordion>

## E2E راه‌اندازی اولیه (Docker)

اختیاری؛ فقط برای آزمون‌های دود راه‌اندازی اولیه در کانتینر لازم است. جریان کامل شروع سرد در یک کانتینر پاک Linux:

```bash
scripts/e2e/onboard-docker.sh
```

ویزارد تعاملی را از طریق یک شبه‌ترمینال هدایت می‌کند، فایل‌های پیکربندی/فضای کاری/نشست را تأیید می‌کند، سپس Gateway را راه‌اندازی کرده و `openclaw health` را اجرا می‌کند.

## آزمون دود واردکردن QR (Docker)

اطمینان می‌دهد که راهنمای زمان‌اجرای QR نگه‌داری‌شده تحت زمان‌های اجرای پشتیبانی‌شدهٔ Docker Node بارگذاری می‌شود (Node 24 پیش‌فرض، سازگار با Node 22):

```bash
pnpm test:docker:qr
```

## مرتبط

- [آزمایش](/fa/help/testing)
- [آزمایش زنده](/fa/help/testing-live)
- [آزمایش به‌روزرسانی‌ها و پلاگین‌ها](/fa/help/testing-updates-plugins)
