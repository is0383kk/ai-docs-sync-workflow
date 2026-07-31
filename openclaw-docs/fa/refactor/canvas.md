---
read_when:
    - انتقال مالکیت میزبان، ابزارها، فرمان‌ها، مستندات یا پروتکل Canvas
    - ممیزی این‌که آیا Canvas همچنان تحت مالکیت هسته است یا خیر
    - آماده‌سازی یا بازبینی PR آزمایشی Plugin بوم نقاشی
summary: برنامه و چک‌لیست ممیزی برای انتقال Canvas از هسته به یک Plugin آزمایشی همراه‌شده.
title: بازآرایی Plugin بوم
x-i18n:
    generated_at: "2026-07-27T14:34:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ead3f865ea80acb1e47f45a5ab07acf19a6470035c00c81006b2b1230bedd71e
    source_path: refactor/canvas.md
    workflow: 16
---

# بازآرایی Plugin مربوط به Canvas

Canvas کم‌استفاده و آزمایشی است. با آن به‌عنوان یک Plugin همراه رفتار کنید، نه یک قابلیت هسته‌ای. هسته می‌تواند زیرساخت عمومی Gateway، Node، HTTP، احراز هویت، پیکربندی و کلاینت بومی را نگه دارد، اما رفتار مختص Canvas باید زیر `extensions/canvas` قرار گیرد.

## هدف

انتقال مالکیت Canvas به `extensions/canvas` با حفظ رفتار فعلی Node جفت‌شده:

- ابزار `canvas` که در اختیار عامل قرار می‌گیرد، توسط Plugin مربوط به Canvas ثبت شود
- دستورهای Node مربوط به Canvas فقط زمانی مجاز باشند که Plugin مربوط به Canvas آن‌ها را ثبت کند
- فایل‌های میزبان/منبع A2UI زیر Plugin مربوط به Canvas قرار گیرند
- مادی‌سازی سند Canvas زیر Plugin مربوط به Canvas قرار گیرد
- پیاده‌سازی دستور CLI زیر Plugin مربوط به Canvas قرار گیرد یا از طریق یک barrel زمان اجرای متعلق به Plugin واگذار شود
- مستندات و فهرست Pluginها، Canvas را آزمایشی و متکی بر Plugin توصیف کنند

## اهداف خارج از محدوده

- در این بازآرایی، رابط کاربری Canvas در برنامه بومی را بازطراحی نکنید.
- پشتیبانی پروتکل/کلاینت Canvas را از iOS، Android یا macOS حذف نکنید، مگر اینکه تصمیم محصول جداگانه‌ای حذف Canvas را مقرر کند.
- صرفاً برای Canvas یک چارچوب گسترده خدمات Plugin نسازید، مگر اینکه دست‌کم یک Plugin همراه دیگر نیز به همان درگاه نیاز داشته باشد.

## وضعیت شاخه فعلی

انجام‌شده:

- بسته Plugin همراه در `extensions/canvas` افزوده شد.
- `extensions/canvas/openclaw.plugin.json` افزوده شد.
- ابزار `canvas` عامل از `src/agents/tools/canvas-tool.ts` به `extensions/canvas/src/tool.ts` منتقل شد.
- ثبت هسته‌ای `createCanvasTool` از `src/agents/openclaw-tools.ts` حذف شد.
- پیاده‌سازی میزبان Canvas از `src/canvas-host` به `extensions/canvas/src/host` منتقل شد.
- `extensions/canvas/runtime-api.ts` به‌عنوان barrel سازگاری متعلق به Plugin برای آزمون‌ها، بسته‌بندی و ابزارهای کمکی عمومی و خارجی Canvas حفظ شد.
- مادی‌سازی سند Canvas از `src/gateway/canvas-documents.ts` به `extensions/canvas/src/documents.ts` منتقل شد.
- پیاده‌سازی CLI مربوط به Canvas و ابزارهای کمکی JSONL مربوط به A2UI به `extensions/canvas/src/cli.ts` منتقل شدند.
- نشانی URL میزبان Canvas و ابزارهای کمکی قابلیتِ محدوده‌بندی‌شده به `extensions/canvas/src` منتقل شدند.
- پیش‌فرض‌های دستور Node مربوط به Canvas از فهرست‌های هسته‌ای کدنویسی‌شده خارج و به `nodeInvokePolicies` در Plugin منتقل شدند.
- پیکربندی میزبان Canvas متعلق به Plugin در `plugins.entries.canvas.config.host` افزوده شد.
- ارائه HTTP مربوط به Canvas و A2UI پشت ثبت مسیر HTTP در Plugin مربوط به Canvas قرار گرفت.
- ارسال عمومی ارتقای WebSocket برای مسیرهای HTTP متعلق به Plugin افزوده شد.
- نشانی URL میزبان مختص Canvas در Gateway و احراز هویت قابلیت Node با سطح میزبانی‌شده عمومی Plugin و ابزارهای کمکی قابلیت Node جایگزین شدند.
- حل‌کننده‌های رسانه میزبانی‌شده متعلق به Plugin افزوده شدند تا نشانی‌های URL اسناد Canvas از طریق Plugin مربوط به Canvas حل شوند، نه با واردکردن جزئیات داخلی سند Canvas توسط هسته.
- `api.registerNodeCliFeature(...)` افزوده شد تا Canvas بتواند `openclaw nodes canvas` را بدون نوشتن دستی مسیر دستور والد، به‌عنوان قابلیت Node متعلق به Plugin اعلام کند.
- واردکردن‌های تولیدی `src/**` از `extensions/canvas/runtime-api.js` حذف شدند.
- منبع بسته A2UI از `apps/shared/OpenClawKit/Tools/CanvasA2UI` به `extensions/canvas/src/host/a2ui-app` منتقل شد.
- پیاده‌سازی ساخت/کپی A2UI به زیر `extensions/canvas/scripts` منتقل و سیم‌کشی ساخت ریشه با hookهای عمومی دارایی Plugin همراه جایگزین شد.
- نام مستعار زمان اجرای قدیمی و سطح‌بالای پیکربندی `canvasHost` حذف شد.
- مهاجرت doctor مربوط به Canvas حفظ شد تا `openclaw doctor --fix` پیکربندی‌های قدیمی `canvasHost` را به `plugins.entries.canvas.config.host` بازنویسی کند.
- سازگاری پروتکل Canvas برای عامل‌های قدیمی، در پشت پروتکل Gateway نسخه v4 حذف شد. اکنون کلاینت‌های بومی و Gatewayها فقط از `pluginSurfaceUrls.canvas` به‌همراه `node.pluginSurface.refresh` استفاده می‌کنند؛ مسیر منسوخ `canvasHostUrl`، `canvasCapability` و `node.canvas.capability.refresh` در این بازآرایی آزمایشی عمداً پشتیبانی نمی‌شود.
- فهرست تولیدشده Pluginها برای گنجاندن Canvas به‌روزرسانی شد.
- مستندات مرجع Plugin در `docs/plugins/reference/canvas.md` افزوده شد.

سطوح شناخته‌شده باقی‌مانده Canvas که در مالکیت هسته‌اند:

- کنترل‌کننده‌های Canvas برنامه بومی زیر `apps/` همچنان عمداً سطح Plugin مربوط به Canvas را مصرف می‌کنند
- کنترل‌کننده‌های پروتکل/کلاینت Canvas در برنامه بومی زیر `apps/`
- خروجی محصول منتشرشده برای جست‌وجوی زمان اجرای سازگار با نسخه‌های پیشین همچنان از `dist/canvas-host/a2ui` استفاده می‌کند، اما مرحله کپی اکنون متعلق به Plugin است

## ساختار هدف

`extensions/canvas` باید مالک موارد زیر باشد:

- مانیفست Plugin و فراداده بسته
- ثبت ابزار عامل
- سیاست دستور فراخوانی Node
- میزبان Canvas و زمان اجرای A2UI
- منبع بسته A2UI مربوط به Canvas و اسکریپت‌های ساخت/کپی دارایی
- ایجاد سند Canvas و حل دارایی
- پیاده‌سازی CLI مربوط به Canvas
- صفحه مستندات Canvas و مدخل فهرست Pluginها

هسته فقط باید مالک درگاه‌های عمومی زیر باشد:

- کشف و ثبت Plugin
- رجیستری عمومی ابزار عامل
- رجیستری عمومی سیاست فراخوانی Node
- HTTP/احراز هویت عمومی Gateway و ارسال ارتقای WebSocket
- حل عمومی نشانی URL سطح میزبانی‌شده Plugin
- ثبت عمومی حل‌کننده رسانه میزبانی‌شده
- انتقال عمومی قابلیت Node
- زیرساخت عمومی پیکربندی
- کشف عمومی hook دارایی Plugin همراه

برنامه‌های بومی می‌توانند کنترل‌کننده‌های دستور Canvas را به‌عنوان کلاینت‌های پروتکل نگه دارند. آن‌ها مالک زمان اجرای Plugin نیستند.

## مراحل مهاجرت

1. با `plugins.entries.canvas.config.host` به‌عنوان سطح پیکربندی متعلق به Plugin رفتار کنید.
2. مستندات را به‌روزرسانی کنید تا Canvas به‌عنوان یک Plugin همراه آزمایشی توصیف شود.
3. آزمون‌های متمرکز Canvas، بررسی‌های فهرست Pluginها، بررسی‌های API مربوط به SDK Plugin و دروازه‌های ساخت/نوع متأثر از مرزهای زمان اجرا را اجرا کنید.

## چک‌لیست ممیزی

پیش از کامل اعلام‌کردن بازآرایی:

- `rg "src/canvas-host|../canvas-host"` هیچ واردکردن منبع فعالی برنمی‌گرداند.
- `rg "canvas-tool|createCanvasTool" src` هیچ پیاده‌سازی ابزار Canvas متعلق به هسته‌ای پیدا نمی‌کند.
- `rg "canvas.present|canvas.snapshot|canvas.a2ui" src/gateway` هیچ پیش‌فرض فهرست مجاز کدنویسی‌شده‌ای خارج از آزمون‌های سیاست عمومی Plugin پیدا نمی‌کند.
- `rg "extensions/canvas/runtime-api" src --glob '!**/*.test.ts'` خالی است.
- `rg "canvas-documents" src` خالی است.
- `rg "registerNodesCanvasCommands|nodes-canvas" src` خالی است؛ Plugin مربوط به Canvas، `openclaw nodes canvas` را از طریق فراداده تودرتوی CLI مربوط به Plugin ثبت می‌کند.
- `rg "createCanvasHostHandler|handleA2uiHttpRequest" src/gateway` هیچ مالکیت زمان اجرای Gateway برنمی‌گرداند.
- `rg "apps/shared/OpenClawKit/Tools/CanvasA2UI|canvas-a2ui-copy|extensions/canvas/src/host/a2ui" scripts .github package.json` فقط پوشش‌های سازگاری یا مسیرهای متعلق به Plugin را پیدا می‌کند.
- `pnpm plugins:inventory:check` با موفقیت اجرا می‌شود.
- `pnpm plugin-sdk:api:check` با موفقیت اجرا می‌شود، یا رکوردهای تولیدشده قرارداد API عمداً به‌روزرسانی و بازبینی می‌شوند.
- آزمون‌های هدفمند Canvas با موفقیت اجرا می‌شوند.
- آزمون‌های مسیرهای تغییریافته برای مسیرهای میزبان Canvas/A2UI با موفقیت اجرا می‌شوند.
- بدنه PR صراحتاً بیان می‌کند که Canvas آزمایشی و متکی بر Plugin است.

## دستورهای راستی‌آزمایی

هنگام تکرار چرخه توسعه، از بررسی‌های محلی هدفمند استفاده کنید:

```sh
pnpm test extensions/canvas/src/host/server.test.ts extensions/canvas/src/host/server.state-dir.test.ts extensions/canvas/src/host/file-resolver.test.ts
pnpm test src/gateway/server.plugin-node-capability-auth.test.ts src/gateway/server-import-boundary.test.ts
pnpm test extensions/canvas/src/config-migration.test.ts src/commands/doctor-legacy-config.migrations.test.ts
pnpm test test/scripts/changed-lanes.test.ts test/scripts/build-all.test.ts extensions/canvas/scripts/bundle-a2ui.test.ts test/scripts/bundled-plugin-assets.test.ts extensions/canvas/scripts/copy-a2ui.test.ts src/infra/run-node.test.ts
pnpm tsgo:extensions
pnpm plugins:inventory:check
pnpm plugin-sdk:api:check
```

اگر barrel زمان اجرا، واردکردن تنبل، بسته‌بندی یا سطوح منتشرشده Plugin تغییر می‌کنند، پیش از push کردن `pnpm build` را اجرا کنید.
