---
read_when:
    - انتخاب زیرمسیر مناسب plugin-sdk برای import در یک Plugin
    - ممیزی زیرمسیرهای Pluginهای همراه و سطوح توابع کمکی
summary: 'کاتالوگ زیربخش‌های SDK افزونه: هر import در کجا قرار دارد، گروه‌بندی‌شده بر اساس حوزه'
title: زیرمسیرهای SDK افزونه
x-i18n:
    generated_at: "2026-07-27T16:52:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 58df43436d0e26f1ffa1383be47fd108655e57d61cf5534d650a4fa2fb7b364c
    source_path: plugins/sdk-subpaths.md
    workflow: 16
---

SDK افزونه شامل زیرمسیرهای عمومی محدود و کمک‌کننده‌های بسته‌بندی‌شدهٔ مختص مخزن
در `openclaw/plugin-sdk/` است. این صفحه هر دو را فهرست می‌کند و ورودی‌های
خصوصی-محلی را صریحاً برچسب می‌زند. سه فایل مرز را تعریف می‌کنند:

- `scripts/lib/plugin-sdk-entrypoints.json`: فهرست نگه‌داری‌شدهٔ نقاط ورود
  که فرایند ساخت آن را کامپایل می‌کند.
- `scripts/lib/plugin-sdk-private-local-only-subpaths.json`: زیرمسیرهای داخلی
  که از SDK نوع‌دار و مستندشده کنار گذاشته شده‌اند. ورودی‌های عملیاتی به‌عنوان
  خروجی‌های زمان اجرای میزبانِ صرفاً JavaScript برای افزونه‌های رسمیِ جداگانه
  منتشرشده همچنان در دسترس‌اند؛ ورودی‌های صرفاً آزمایشی صادر نمی‌شوند.
- `src/plugin-sdk/entrypoints.ts`: فرادادهٔ طبقه‌بندی برای زیرمسیرهای
  منسوخ‌شده، کمک‌کننده‌های بسته‌بندی‌شدهٔ رزروشده، رابط‌های بسته‌بندی‌شدهٔ
  پشتیبانی‌شده و سطوح عمومی تحت مالکیت افزونه.

نگه‌دارندگان تعداد خروجی‌های عمومی را با `pnpm plugin-sdk:surface` و
زیرمسیرهای فعالِ کمک‌کننده‌های رزروشده را با `pnpm plugins:boundary-report:summary`
ممیزی می‌کنند؛ خروجی‌های رزروشدهٔ استفاده‌نشدهٔ کمک‌کننده به‌جای باقی‌ماندن در
SDK عمومی به‌عنوان بدهی سازگاری غیرفعال، باعث شکست گزارش CI می‌شوند.

برای راهنمای ساخت افزونه، [نمای کلی SDK افزونه](/fa/plugins/sdk-overview) را ببینید.

## ورودی افزونه

| زیرمسیر                        | خروجی‌های کلیدی                                                                                                                                                                                             |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/plugin-entry`      | `definePluginEntry`                                                                                                                                                                                     |
| `plugin-sdk/core`              | `defineChannelPluginEntry`, `createChatChannelPlugin`, `createChannelPluginBase`, `defineSetupPluginEntry`, `buildChannelConfigSchema`, `buildJsonChannelConfigSchema`, `resolveTailscalePublishedHost` |
| `plugin-sdk/provider-entry`    | خصوصی-محلی پس از ژوئیهٔ 2026؛ `defineSingleProviderPluginEntry`                                                                                                                                        |
| `plugin-sdk/migration`         | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های مورد ارائه‌دهندهٔ مهاجرت مانند `createMigrationItem`، ثابت‌های دلیل، نشانگرهای وضعیت مورد، کمک‌کننده‌های حذف اطلاعات حساس و `summarizeMigrationItems`                   |
| `plugin-sdk/migration-runtime` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های مهاجرت زمان اجرا مانند `copyMigrationFileItem`، `resolvePlannedMigrationTargets`، `withCachedMigrationConfigRuntime` و `writeMigrationReport`              |
| `plugin-sdk/health`            | ثبت بررسی سلامت Doctor، تشخیص، تعمیر، انتخاب، شدت و انواع یافته برای مصرف‌کنندگان بسته‌بندی‌شدهٔ سلامت                                                                                |

### کمک‌کننده‌های سازگاری و خصوصی-محلی

فقط زیرمسیرهای منسوخ‌شدهٔ پنجره‌های زمانی دیرتر همچنان صادر می‌شوند. نام‌های مستعار ژوئیهٔ 2026 و
زیرمسیرهای استفاده‌نشده حذف شدند، درحالی‌که کمک‌کننده‌های صرفاً بسته‌بندی‌شده از
بستهٔ عمومی حذف شده‌اند و در ادامه خصوصی-محلی برچسب‌گذاری می‌شوند. فهرست نگه‌داری‌شده
`scripts/lib/plugin-sdk-deprecated-public-subpaths.json` است؛ CI موارد بسته‌بندی‌شده را رد می‌کند.
`plugin-sdk/text-runtime` فقط برای سازگاری هستند و `plugin-sdk/zod` یک
بازصدور سازگاری است: `zod` را مستقیماً از `zod` وارد کنید. barrelهای گستردهٔ دامنه
یعنی `plugin-sdk/agent-runtime`، `plugin-sdk/channel-lifecycle`،
`plugin-sdk/conversation-runtime`، `plugin-sdk/hook-runtime`،
`plugin-sdk/media-runtime`، `plugin-sdk/plugin-runtime` و
`plugin-sdk/security-runtime` نیز به‌نفع زیرمسیرهای متمرکز
منسوخ شده‌اند.

زیرمسیرهای کمک‌کنندهٔ آزمونِ مبتنی بر Vitest در OpenClaw فقط مختص مخزن‌اند و دیگر
جزو خروجی‌های بسته نیستند: `agent-runtime-test-contracts`،
`channel-contract-testing`، `channel-target-testing`، `channel-test-helpers`،
`plugin-state-test-runtime`، `plugin-test-api`، `plugin-test-contracts`،
`plugin-test-runtime`، `provider-http-test-mocks`، `provider-test-contracts`،
`reply-payload-testing`، `sqlite-runtime-testing`، `test-env`، `test-fixtures`،
`test-live`، `test-live-auth`، `test-media-generation`،
`test-media-understanding`، `test-node-mocks` و `testing`. سطوح خصوصیِ کمک‌کننده‌های بسته‌بندی‌شدهٔ
`ssrf-runtime-internal` و `codex-native-task-runtime` نیز فقط
مختص مخزن‌اند.

### زیرمسیرهای کمک‌کنندهٔ افزونهٔ بسته‌بندی‌شده

ماژول‌های کمک‌کنندهٔ صرفاً بسته‌بندی‌شده پس از پاک‌سازی ژوئیهٔ 2026 خصوصی-محلی هستند. واردکردن بین مالکان با محافظ‌های قرارداد بسته مسدود می‌شود. `src/plugin-sdk/entrypoints.ts` رابط‌های بسته‌بندی‌شدهٔ پشتیبانی‌شده‌ای را که عمومی باقی می‌مانند جداگانه ردیابی می‌کند؛ این‌ها نقاط ورود SDK هستند که تا زمان جایگزینی
قراردادهای عمومی، افزونهٔ بسته‌بندی‌شدهٔ آن‌ها پشتیبانی‌شان می‌کند.
`plugin-sdk/qa-runner-runtime` و `plugin-sdk/telegram-account`
برای کد جدید منسوخ شده‌اند؛ یادداشت‌های هر ردیف را در ادامه ببینید.

<AccordionGroup>
  <Accordion title="زیرمسیرهای کانال">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/channel-core` | `defineChannelPluginEntry`، `defineSetupPluginEntry`، `createChatChannelPlugin`، `createChannelPluginBase`، `createChannelConfigUiHints` |
    | `plugin-sdk/json-schema-runtime` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کنندهٔ اعتبارسنجی JSON Schema ذخیره‌شده در حافظهٔ نهان برای طرح‌واره‌های تحت مالکیت افزونه |
    | `plugin-sdk/channel-setup` | `defineChannelSetupContract`، انواع فیلد/ورودی راه‌اندازی تحت مالکیت کانال، `createOptionalChannelSetupSurface`، `createOptionalChannelSetupAdapter`، `createOptionalChannelSetupWizard`، به‌علاوهٔ `DEFAULT_ACCOUNT_ID`، `createTopLevelChannelDmPolicy`، `setSetupChannelEnabled`، `splitSetupEntries` |
    | `plugin-sdk/setup` | کمک‌کننده‌های مشترک جادوگر راه‌اندازی، مترجم راه‌اندازی، اعلان‌های فهرست مجاز، سازنده‌های وضعیت راه‌اندازی |
    | `plugin-sdk/setup-runtime` | `defineChannelSetupContract`، `createSetupTranslator`، `createPatchedAccountSetupAdapter`، `createEnvPatchedAccountSetupAdapter`، `createSetupInputPresenceValidator`، `noteChannelLookupFailure`، `noteChannelLookupSummary`، `promptResolvedAllowFrom`، `splitSetupEntries`، `createAllowlistSetupWizardProxy`، `createDelegatedSetupWizardProxy` |
    | `plugin-sdk/setup-tools` | `formatCliCommand`، `detectBinary`، `extractArchive`، `resolveBrewExecutable`، `formatDocsLink`، `CONFIG_DIR` |
    | `plugin-sdk/account-core` | کمک‌کننده‌های پیکربندی چندحسابی/دروازهٔ کنش، کمک‌کننده‌های پس‌گرد حساب پیش‌فرض |
    | `plugin-sdk/account-id` | `DEFAULT_ACCOUNT_ID`، کمک‌کننده‌های نرمال‌سازی شناسهٔ حساب |
    | `plugin-sdk/account-resolution` | کمک‌کننده‌های جست‌وجوی حساب + پس‌گرد پیش‌فرض |
    | `plugin-sdk/account-helpers` | کمک‌کننده‌های محدود فهرست حساب/کنش حساب |
    | `plugin-sdk/access-groups` | خصوصی-محلی پس از ژوئیهٔ 2026؛ تجزیهٔ فهرست مجاز گروه دسترسی و کمک‌کننده‌های عیب‌یابی گروه با اطلاعات حساس حذف‌شده |
    | `plugin-sdk/channel-pairing` | `createChannelPairingController` |
    | `plugin-sdk/channel-reply-pipeline` | رابط سازگاری منسوخ‌شده. از `plugin-sdk/channel-outbound` استفاده کنید. |
    | `plugin-sdk/channel-config-helpers` | `createHybridChannelConfigAdapter`، `resolveChannelDmAccess`، `resolveChannelDmAllowFrom`، `resolveChannelDmPolicy`، `normalizeChannelDmPolicy`، `normalizeLegacyDmAliases` |
    | `plugin-sdk/channel-config-schema` | مؤلفه‌های پایهٔ طرح‌وارهٔ پیکربندی کانال مشترک، به‌علاوهٔ Zod و سازنده‌های مستقیم JSON/TypeBox |
    | `plugin-sdk/bundled-channel-config-schema` | خصوصی-محلی پس از ژوئیهٔ 2026؛ طرح‌واره‌های پیکربندی کانال OpenClaw بسته‌بندی‌شده فقط برای افزونه‌های بسته‌بندی‌شدهٔ نگه‌داری‌شده |
    | `plugin-sdk/chat-channel-ids` | خصوصی-محلی پس از ژوئیهٔ 2026؛ `BUNDLED_CHAT_CHANNEL_IDS`، `BUNDLED_CHAT_CHANNEL_ENVELOPE_PREFIXES`، `ChatChannelId`. شناسه‌های متعارف کانال گفت‌وگوی بسته‌بندی‌شده/رسمی، به‌علاوهٔ برچسب‌ها/نام‌های مستعار قالب‌بند برای افزونه‌هایی که باید متن دارای پیشوند پاکت را بدون کدنویسی ثابت جدول خود تشخیص دهند. |
    | `plugin-sdk/channel-policy` | `resolveChannelGroupRequireMention` |
    | `plugin-sdk/channel-ingress-runtime` | حل‌کنندهٔ آزمایشی سطح‌بالای زمان اجرای ورودی کانال، حل‌کنندهٔ سیاست اشارهٔ ضمنی و سازنده‌های واقعیت مسیر برای مسیرهای دریافت کانال مهاجرت‌یافته. این روش را به ساخت فهرست‌های مجاز مؤثر، فهرست‌های مجاز فرمان و تصویرسازی‌های قدیمی در هر افزونه ترجیح دهید. [API ورودی کانال](/fa/plugins/sdk-channel-ingress) را ببینید. |
    | `plugin-sdk/channel-lifecycle` | رابط سازگاری منسوخ‌شده. از `plugin-sdk/channel-outbound` استفاده کنید. |
    | `plugin-sdk/channel-outbound` | قراردادهای چرخهٔ عمر پیام، به‌علاوهٔ گزینه‌های پایپ‌لاین پاسخ، رسیدها، پیش‌نمایش زنده/جریان‌دهی، کمک‌کننده‌های چرخهٔ عمر، هویت خروجی، برنامه‌ریزی بار داده، ارسال‌های پایدار و کمک‌کننده‌های زمینهٔ ارسال پیام. [API خروجی کانال](/fa/plugins/sdk-channel-outbound) را ببینید. |
    | `plugin-sdk/channel-message` | نام مستعار سازگاری منسوخ‌شده برای `plugin-sdk/channel-outbound`. |
    | `plugin-sdk/inbound-envelope` | کمک‌کننده‌های مشترک سازندهٔ مسیر ورودی + پاکت |
    | `plugin-sdk/inbound-reply-dispatch` | رابط سازگاری منسوخ‌شده. برای اجراکننده‌های ورودی و گزاره‌های ارسال از `plugin-sdk/channel-inbound` و برای کمک‌کننده‌های تحویل پیام از `plugin-sdk/channel-outbound` استفاده کنید. |
    | `plugin-sdk/messaging-targets` | نام مستعار منسوخ‌شدهٔ تجزیهٔ مقصد؛ از `plugin-sdk/channel-targets` استفاده کنید |
    | `plugin-sdk/outbound-media` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های مشترک بارگذاری رسانهٔ خروجی و وضعیت رسانهٔ میزبانی‌شده |
    | `plugin-sdk/poll-runtime` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های محدود نرمال‌سازی نظرسنجی |
    | `plugin-sdk/thread-bindings-runtime` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های چرخهٔ عمر و سازگارکنندهٔ اتصال رشته |
    | `plugin-sdk/agent-media-payload` | رابط سازگاری منسوخ‌شده برای تصویرسازی بار دادهٔ قدیمی `Media*`. واقعیت‌های مرتب‌شده را از طریق `MsgContext.media` / `toInboundMediaFacts(...)` عبور دهید؛ سیاست ریشهٔ محلی را از `plugin-sdk/media-local-roots` وارد کنید. |
    | `plugin-sdk/conversation-runtime` | barrel گستردهٔ منسوخ‌شده برای اتصال گفت‌وگو/رشته، جفت‌سازی و کمک‌کننده‌های اتصال پیکربندی‌شده؛ زیرمسیرهای متمرکز اتصال مانند `plugin-sdk/thread-bindings-runtime` و `plugin-sdk/session-binding-runtime` را ترجیح دهید |
    | `plugin-sdk/runtime-group-policy` | کمک‌کننده‌های تفکیک سیاست گروه در زمان اجرا |
    | `plugin-sdk/channel-status` | کمک‌کننده‌های مشترک تصویر لحظه‌ای/خلاصهٔ وضعیت کانال |
    | `plugin-sdk/channel-config-primitives` | مؤلفه‌های پایهٔ محدود طرح‌وارهٔ پیکربندی کانال |
    | `plugin-sdk/channel-config-writes` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های مجوز نوشتن پیکربندی کانال |
    | `plugin-sdk/channel-plugin-common` | خروجی‌های پیش‌درآمد مشترک افزونهٔ کانال |
    | `plugin-sdk/allowlist-config-edit` | کمک‌کننده‌های ویرایش/خواندن پیکربندی فهرست مجاز |
    | `plugin-sdk/group-access` | کمک‌کننده‌های منسوخ‌شدهٔ تصمیم‌گیری دسترسی گروه؛ از `resolveChannelMessageIngress` در `plugin-sdk/channel-ingress-runtime` استفاده کنید |
    | `plugin-sdk/direct-dm-guard-policy` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های محدود سیاست محافظ پیش از رمزنگاری پیام مستقیم |
    | `plugin-sdk/discord` | رابط سازگاری منسوخ‌شدهٔ Discord برای `@openclaw/discord@2026.3.13` منتشرشده و سازگاری ردیابی‌شدهٔ مالک؛ افزونه‌های جدید باید از زیرمسیرهای عمومی SDK کانال استفاده کنند |
    | `plugin-sdk/telegram-account` | رابط سازگاری منسوخ‌شدهٔ تفکیک حساب Telegram برای سازگاری ردیابی‌شدهٔ مالک؛ افزونه‌های جدید باید از کمک‌کننده‌های تزریق‌شدهٔ زمان اجرا یا زیرمسیرهای عمومی SDK کانال استفاده کنند |
    | `plugin-sdk/interactive-runtime` | کمک‌کننده‌های معنایی ارائه و تحویل پیام و پاسخ تعاملی قدیمی. [ارائهٔ پیام](/fa/plugins/message-presentation) را ببینید |
    | `plugin-sdk/question-gateway-runtime` | انتخاب‌های `ask_user` ایجادشده در زمان اجرا را از مدیریت‌کننده‌های تعامل کانال، از طریق Gateway تفکیک کنید |
    | `plugin-sdk/channel-inbound` | کمک‌کننده‌های مشترک ورودی برای طبقه‌بندی رویداد، ساخت زمینه، قالب‌بندی، ریشه‌ها، رفع جهش، تطبیق اشاره، سیاست اشاره و ثبت گزارش ورودی |
    | `plugin-sdk/channel-inbound-debounce` | کمک‌کننده‌های محدود رفع جهش ورودی |
    | `plugin-sdk/channel-mention-gating` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های محدود سیاست اشاره، نشانگر اشاره و متن اشاره بدون سطح گسترده‌تر زمان اجرای ورودی |
    | `plugin-sdk/channel-streaming` | رابط سازگاری منسوخ‌شده. از `plugin-sdk/channel-outbound` استفاده کنید. |
    | `plugin-sdk/channel-send-result` | انواع نتیجهٔ پاسخ |
    | `plugin-sdk/channel-actions` | کمک‌کننده‌های کنش پیام کانال، به‌علاوهٔ کمک‌کننده‌های منسوخ‌شدهٔ طرح‌وارهٔ بومی که برای سازگاری افزونه حفظ شده‌اند |
    | `plugin-sdk/channel-route` | خصوصی-محلی پس از ژوئیهٔ 2026؛ نرمال‌سازی مشترک مسیر، تفکیک مقصد مبتنی بر تجزیه‌گر، رشته‌سازی شناسهٔ رشته، کلیدهای مسیر حذف تکرار/فشرده، انواع مقصد تجزیه‌شده و کمک‌کننده‌های مقایسهٔ مسیر/مقصد |
    | `plugin-sdk/channel-targets` | خصوصی-محلی پس از ژوئیهٔ 2026؛ کمک‌کننده‌های تجزیهٔ مقصد؛ فراخواننده‌های مقایسهٔ مسیر باید از `plugin-sdk/channel-route` استفاده کنند |
    | `plugin-sdk/channel-contract` | انواع قرارداد کانال |
    | `plugin-sdk/channel-feedback` | سیم‌کشی بازخورد/واکنش |
  </Accordion>

زیرمسیرهای سازگاری کانال مربوط به پنجره‌های زمانی دیرتر فقط تا تاریخ‌های ثبت‌شدهٔ خود
عمومی باقی می‌مانند. نام‌های مستعار ژوئیه مانند دسترسی مستقیم به پیام خصوصی، گزینه‌های پاسخ، مسیرهای
جفت‌سازی و انشعاب‌های زمان اجرای کانال حذف شده‌اند؛ کمک‌کننده‌های صرفاً بسته‌بندی‌شده
خصوصی-محلی هستند.

  <Accordion title="زیرمسیرهای ارائه‌دهنده">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/provider-entry` | محلی و خصوصی پس از ژوئیهٔ 2026؛ `defineSingleProviderPluginEntry` |
    | `plugin-sdk/provider-setup` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی منتخب برای راه‌اندازی ارائه‌دهندهٔ محلی/خودمیزبان |
    | `plugin-sdk/cli-backend` | محلی و خصوصی پس از ژوئیهٔ 2026؛ پیش‌فرض‌های بک‌اند CLI و ثابت‌های پایش‌گر |
    | `plugin-sdk/provider-auth-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی زمان اجرای احراز هویت ارائه‌دهنده: جریان بازگشتی OAuth، مبادلهٔ توکن، ماندگارسازی احراز هویت و رفع کلید API |
    | `plugin-sdk/provider-oauth-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ انواع عمومی فراخوانی برگشتی OAuth ارائه‌دهنده، رندر صفحهٔ فراخوانی برگشتی، ابزارهای کمکی PKCE/وضعیت، تجزیهٔ ورودی مجوزدهی، ابزارهای کمکی انقضای توکن و لغو |
    | `plugin-sdk/provider-auth-api-key` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی آغازبه‌کار با کلید API/نوشتن نمایه، مانند `upsertApiKeyProfile` |
    | `plugin-sdk/provider-auth-result` | محلی و خصوصی پس از ژوئیهٔ 2026؛ سازندهٔ استاندارد نتیجهٔ احراز هویت OAuth |
    | `plugin-sdk/provider-env-vars` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی جست‌وجوی متغیر محیطی احراز هویت ارائه‌دهنده |
    | `plugin-sdk/provider-auth` | `createProviderApiKeyAuthMethod`، `ensureApiKeyFromOptionEnvOrPrompt`، `upsertAuthProfile`، `upsertApiKeyProfile`، `writeOAuthCredentials`، ابزارهای کمکی واردکردن احراز هویت OpenAI Codex، خروجی سازگاری منسوخ‌شدهٔ `resolveOpenClawAgentDir` |
    | `plugin-sdk/provider-model-shared` | محلی و خصوصی پس از ژوئیهٔ 2026؛ `ProviderReplayFamily`، `buildProviderReplayFamilyHooks`، `selectPreferredLocalModelId`، `normalizeModelCompat`، سازنده‌های مشترک سیاست بازپخش، ابزارهای کمکی نقطهٔ پایانی ارائه‌دهنده و ابزارهای مشترک عادی‌سازی شناسهٔ مدل |
    | `plugin-sdk/provider-catalog-live-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی کاتالوگ زندهٔ مدل ارائه‌دهنده برای کشف محافظت‌شده به سبک `/models`: ‏`buildLiveModelProviderConfig`، `fetchLiveProviderModelRows`، `getCachedLiveProviderModelRows`، `fetchLiveProviderModelIds`، `LiveModelCatalogHttpError`، `clearLiveCatalogCacheForTests`، پالایش شناسهٔ مدل، کش TTL و بازگشت ایستای جایگزین |
    | `plugin-sdk/provider-catalog-runtime` | قلاب زمان اجرای تکمیل کاتالوگ ارائه‌دهنده و درزهای رجیستری ارائه‌دهندهٔ Plugin برای آزمون‌های قرارداد |
    | `plugin-sdk/provider-catalog-shared` | محلی و خصوصی پس از ژوئیهٔ 2026؛ `findCatalogTemplate`، `buildSingleProviderApiKeyCatalog`، `buildManifestModelProviderConfig`، `supportsNativeStreamingUsageCompat`، `applyProviderNativeStreamingUsageCompat` |
    | `plugin-sdk/provider-http` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای عمومی قابلیت HTTP/نقطهٔ پایانی ارائه‌دهنده، خطاهای HTTP ارائه‌دهنده و ابزارهای کمکی فرم چندبخشی رونویسی صوت |
    | `plugin-sdk/provider-web-fetch-contract` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی محدود قرارداد پیکربندی/انتخاب واکشی وب، مانند `enablePluginInConfig` و `WebFetchProviderPlugin` |
    | `plugin-sdk/provider-web-fetch` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی ثبت/کش ارائه‌دهندهٔ واکشی وب |
    | `plugin-sdk/provider-web-search-config-contract` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی محدود پیکربندی/اعتبارنامهٔ جست‌وجوی وب برای ارائه‌دهندگانی که به سیم‌کشی فعال‌سازی Plugin نیاز ندارند |
    | `plugin-sdk/provider-web-search-contract` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی محدود قرارداد پیکربندی/اعتبارنامهٔ جست‌وجوی وب، مانند `createWebSearchProviderContractFields`، `enablePluginInConfig`، `resolveProviderWebSearchPluginConfig` و تنظیم‌کننده‌ها/دریافت‌کننده‌های اعتبارنامه با دامنهٔ محدود |
    | `plugin-sdk/provider-web-search` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی ثبت/کش/زمان اجرای ارائه‌دهندهٔ جست‌وجوی وب |
    | `plugin-sdk/embedding-providers` | محلی و خصوصی پس از ژوئیهٔ 2026؛ انواع عمومی ارائه‌دهندهٔ تعبیه‌سازی و ابزارهای کمکی خواندن، شامل `EmbeddingProviderAdapter`، `getEmbeddingProvider(...)` و `listEmbeddingProviders(...)`؛ Pluginها ارائه‌دهندگان را از طریق `api.registerEmbeddingProvider(...)` ثبت می‌کنند تا مالکیت مانیفست اعمال شود |
    | `plugin-sdk/provider-tools` | محلی و خصوصی پس از ژوئیهٔ 2026؛ `ProviderToolCompatFamily`، `buildProviderToolCompatFamilyHooks` و پاک‌سازی شِما و عیب‌یابی DeepSeek/Gemini/OpenAI |
    | `plugin-sdk/provider-usage` | محلی و خصوصی پس از ژوئیهٔ 2026؛ انواع عکس فوری مصرف ارائه‌دهنده، ابزارهای مشترک واکشی مصرف و واکشی‌کننده‌های ارائه‌دهنده، مانند `fetchClaudeUsage` |
    | `plugin-sdk/provider-stream` | محلی و خصوصی پس از ژوئیهٔ 2026؛ `ProviderStreamFamily`، `buildProviderStreamFamilyHooks`، `composeProviderStreamWrappers`، انواع پوشش‌دهندهٔ جریان، سازگاری فراخوانی ابزار با متن ساده و ابزارهای مشترک پوشش‌دهندهٔ Anthropic/Google/Kilocode/MiniMax/Moonshot/OpenAI/OpenRouter/Z.AI |
    | `plugin-sdk/provider-stream-shared` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای عمومی و مشترک پوشش‌دهندهٔ جریان ارائه‌دهنده، شامل `composeProviderStreamWrappers`، `createOpenAICompatibleCompletionsThinkingOffWrapper`، `createPlainTextToolCallCompatWrapper`، `createPayloadPatchStreamWrapper`، `createToolStreamWrapper`، `normalizeOpenAICompatibleReasoningPayload`، `setQwenChatTemplateThinking` و ابزارهای جریان سازگار با Anthropic/DeepSeek/OpenAI |
    | `plugin-sdk/provider-transport-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی انتقال بومی ارائه‌دهنده، مانند واکشی محافظت‌شده، استخراج متن نتیجهٔ ابزار، تبدیل پیام‌های انتقال و جریان‌های رویداد انتقال قابل‌نوشتن |
    | `plugin-sdk/provider-onboard` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی وصلهٔ پیکربندی آغازبه‌کار |
    | `plugin-sdk/global-singleton` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی تک‌نمونه/نگاشت/کش محلی فرایند |
    | `plugin-sdk/group-activation` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی محدود حالت فعال‌سازی گروه و تجزیهٔ فرمان |
  </Accordion>

عکس‌های فوری مصرف ارائه‌دهنده معمولاً یک یا چند `windows` سهمیه را گزارش می‌کنند که هرکدام
دارای یک برچسب، درصد مصرف‌شده و زمان بازنشانی اختیاری هستند. ارائه‌دهندگانی که به‌جای بازه‌های سهمیهٔ
قابل‌بازنشانی، موجودی یا متن وضعیت حساب را ارائه می‌کنند، باید `summary` را با آرایهٔ
خالی `windows` برگردانند و درصدهای ساختگی تولید نکنند.
OpenClaw آن متن خلاصه را در خروجی وضعیت نمایش می‌دهد؛ تنها زمانی از `error` استفاده کنید که
نقطهٔ پایانی مصرف شکست خورده یا هیچ دادهٔ مصرف قابل‌استفاده‌ای برنگردانده باشد.

  <Accordion title="زیرمسیرهای احراز هویت و امنیت">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/command-auth` | سطح گسترده و منسوخ‌شدهٔ مجوزدهی فرمان (`resolveControlCommandGate`، ابزارهای کمکی رجیستری فرمان شامل قالب‌بندی پویای منوی آرگومان، ابزارهای کمکی مجازسازی فرستنده)؛ از مجوزدهی ورودی/زمان اجرای کانال یا ابزارهای کمکی وضعیت فرمان استفاده کنید |
    | `plugin-sdk/command-status` | سازنده‌های پیام فرمان/راهنما، مانند `buildCommandsMessagePaginated` و `buildHelpMessage` |
    | `plugin-sdk/approval-auth-runtime` | ابزارهای کمکی تعیین تأییدکننده و احراز هویت کنش در همان گفت‌وگو |
    | `plugin-sdk/approval-client-runtime` | ابزارهای کمکی نمایه/پالایهٔ تأیید اجرای بومی |
    | `plugin-sdk/approval-delivery-runtime` | آداپتورهای قابلیت/تحویل تأیید بومی |
    | `plugin-sdk/approval-gateway-runtime` | رفع‌کنندهٔ مشترک Gateway تأیید |
    | `plugin-sdk/approval-reference-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزار کمکی مکان‌یاب ماندگار و قطعی برای فراخوانی‌های برگشتی تأیید با محدودیت انتقال |
    | `plugin-sdk/approval-handler-adapter-runtime` | ابزارهای سبک بارگذاری آداپتور تأیید بومی برای نقاط ورود پرترافیک کانال |
    | `plugin-sdk/approval-handler-runtime` | ابزارهای گسترده‌تر زمان اجرای کنترل‌گر تأیید؛ هنگامی که درزهای محدودتر آداپتور/Gateway کافی هستند، آن‌ها را ترجیح دهید |
    | `plugin-sdk/approval-native-runtime` | ابزارهای کمکی هدف تأیید بومی، پیوند حساب، دروازهٔ مسیر، بازگشت جایگزین ارسال و سرکوب اعلان محلی اجرای بومی |
    | `plugin-sdk/approval-reaction-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ پیوندهای واکنش تأیید سخت‌کدشده، بارهای اعلان واکنش، ذخیره‌گاه‌های هدف واکنش، ابزارهای کمکی متن راهنمای واکنش و خروجی سازگاری برای سرکوب اعلان محلی اجرای بومی |
    | `plugin-sdk/approval-reply-runtime` | ابزارهای کمکی بار پاسخ تأیید اجرا/Plugin |
    | `plugin-sdk/approval-runtime` | ابزارهای کمکی بار تأیید اجرا/Plugin، سازنده‌های قابلیت تأیید، ابزارهای کمکی احراز هویت/نمایهٔ تأیید، ابزارهای مسیریابی/زمان اجرای تأیید بومی و ابزارهای کمکی نمایش ساختاریافتهٔ تأیید، مانند `formatApprovalDisplayPath` |
    | `plugin-sdk/command-auth-native` | ابزارهای کمکی احراز هویت فرمان بومی، قالب‌بندی پویای منوی آرگومان و هدف‌گیری نشست بومی |
    | `plugin-sdk/command-detection` | ابزارهای مشترک تشخیص فرمان |
    | `plugin-sdk/command-primitives-runtime` | گزاره‌های سبک متن فرمان برای مسیرهای پرترافیک کانال |
    | `plugin-sdk/command-surface` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی عادی‌سازی بدنهٔ فرمان و سطح فرمان |
    | `plugin-sdk/allow-from` | `formatAllowFromLowercase` |
    | `plugin-sdk/provider-auth-login-flow-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی جریان ورود تنبل احراز هویت ارائه‌دهنده برای جفت‌سازی کد دستگاه در کانال خصوصی و رابط کاربری وب |
    | `plugin-sdk/channel-secret-runtime` | سطح گسترده و منسوخ‌شدهٔ قرارداد راز (`collectSimpleChannelFieldAssignments`، `getChannelSurface`، `pushAssignment`، انواع هدف راز)؛ زیرمسیرهای متمرکز زیر را ترجیح دهید |
    | `plugin-sdk/channel-secret-basic-runtime` | خروجی‌های محدود قرارداد راز و سازنده‌های رجیستری هدف برای سطوح راز کانال/Plugin غیر TTS |
    | `plugin-sdk/channel-secret-tts-runtime` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای کمکی محدود تخصیص راز TTS تو‌در‌توی کانال |
    | `plugin-sdk/secret-ref-runtime` | نوع‌دهی محدود SecretRef، رفع آن و جست‌وجوی مسیر هدف طرح برای تجزیهٔ قرارداد راز/پیکربندی |
    | `plugin-sdk/security-runtime` | بشکهٔ گسترده و منسوخ‌شده برای اعتماد، دروازه‌بندی پیام مستقیم، ابزارهای کمکی فایل/مسیر محدود به ریشه شامل نوشتن صرفاً هنگام ایجاد، جایگزینی اتمی همگام/ناهمگام فایل، نوشتن موقت هم‌سطح، بازگشت جایگزین جابه‌جایی میان‌دستگاهی، ابزارهای کمکی ذخیره‌گاه فایل خصوصی، محافظ‌های والد پیوند نمادین، محتوای خارجی، پنهان‌سازی متن حساس، مقایسهٔ زمان‌ثابت راز و ابزارهای جمع‌آوری راز؛ زیرمسیرهای متمرکز امنیت/SSRF/راز را ترجیح دهید |
    | `plugin-sdk/ssrf-policy` | ابزارهای کمکی فهرست مجاز میزبان و سیاست SSRF شبکهٔ خصوصی |
    | `plugin-sdk/ssrf-dispatcher` | محلی و خصوصی پس از ژوئیهٔ 2026؛ ابزارهای محدود توزیع‌کنندهٔ سنجاق‌شده بدون سطح گستردهٔ زمان اجرای زیرساخت |
    | `plugin-sdk/ssrf-runtime` | ابزارهای کمکی توزیع‌کنندهٔ سنجاق‌شده، واکشی محافظت‌شده در برابر SSRF، خطای SSRF و سیاست SSRF |
    | `plugin-sdk/secret-input` | ابزارهای کمکی تجزیهٔ ورودی راز |
    | `plugin-sdk/webhook-ingress` | ابزارهای کمکی درخواست/هدف Webhook و تبدیل اجباری وب‌سوکت/بدنهٔ خام |
    | `plugin-sdk/webhook-request-guards` | ابزارهای کمکی اندازه/مهلت زمانی بدنهٔ درخواست و `runDetachedWebhookWork` برای پردازش رهگیری‌شده پس از تأیید دریافت |
  </Accordion>

  <Accordion title="Runtime and storage subpaths">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/runtime` | ابزارهای کمکی زمان اجرا/ثبت گزارش/پشتیبان‌گیری، هشدارهای مسیر نصب Plugin و ابزارهای کمکی فرایند |
    | `plugin-sdk/runtime-env` | ابزارهای کمکی محدودِ محیط زمان اجرا، ثبت‌کننده، مهلت زمانی، تلاش مجدد و پس‌روی |
    | `plugin-sdk/browser-config` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ نمای پشتیبانی‌شدهٔ پیکربندی مرورگر برای پروفایل/پیش‌فرض‌های نرمال‌شده، تجزیهٔ URL مربوط به CDP و ابزارهای کمکی احراز هویت کنترل مرورگر |
    | `plugin-sdk/agent-harness-task-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی عمومی چرخهٔ عمر وظیفه و تحویل تکمیل برای عامل‌های متکی بر هارنس که از دامنهٔ وظیفهٔ صادرشده توسط میزبان استفاده می‌کنند |
    | `plugin-sdk/codex-mcp-projection` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی رزروشدهٔ Codex همراه برای نگاشت پیکربندی سرور MCP کاربر به پیکربندی رشتهٔ Codex؛ نه برای Pluginهای شخص ثالث |
    | `plugin-sdk/codex-native-task-runtime` | ابزار کمکی Codex همراه و محلیِ مخزن برای سیم‌کشی بومی آینهٔ وظیفه/زمان اجرا؛ خروجی پکیج نیست |
    | `plugin-sdk/channel-runtime-context` | ابزارهای کمکی عمومی ثبت و جست‌وجوی زمینهٔ زمان اجرای کانال |
    | `plugin-sdk/matrix` | نمای سازگاری منسوخ‌شدهٔ Matrix برای پکیج‌های کانال شخص ثالث قدیمی‌تر؛ Pluginهای جدید باید `plugin-sdk/run-command` را مستقیماً وارد کنند |
    | `plugin-sdk/runtime-store` | `createPluginRuntimeStore` |
    | `plugin-sdk/plugin-runtime` | خروجی تجمیعی گسترده و منسوخ‌شده برای ابزارهای کمکی فرمان/هوک/HTTP/تعاملی Plugin؛ زیرمسیرهای متمرکز زمان اجرای Plugin ترجیح داده می‌شوند |
    | `plugin-sdk/hook-runtime` | خروجی تجمیعی گسترده و منسوخ‌شده برای ابزارهای کمکی پایپ‌لاین Webhook/هوک داخلی؛ زیرمسیرهای متمرکز زمان اجرای هوک/Plugin ترجیح داده می‌شوند |
    | `plugin-sdk/lazy-runtime` | ابزارهای کمکی واردکردن/اتصال تنبل زمان اجرا مانند `createLazyRuntimeModule`، `createLazyRuntimeMethod` و `createLazyRuntimeSurface` |
    | `plugin-sdk/process-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی اجرای فرایند |
    | `plugin-sdk/node-host` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی تفکیک فایل اجرایی میزبان Node و ازسرگیری PTY |
    | `plugin-sdk/cli-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ خروجی تجمیعی گسترده و منسوخ‌شده برای ابزارهای کمکی قالب‌بندی CLI، انتظار، نسخه، فراخوانی آرگومان و گروه فرمان تنبل؛ زیرمسیرهای متمرکز CLI/زمان اجرا ترجیح داده می‌شوند |
    | `plugin-sdk/qa-runner-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ نمای پشتیبانی‌شده‌ای که سناریوهای تضمین کیفیت Plugin را از طریق سطح فرمان CLI ارائه می‌کند |
    | `plugin-sdk/tts-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ نمای پشتیبانی‌شده برای شِمای پیکربندی تبدیل متن به گفتار و ابزارهای کمکی زمان اجرا |
    | `plugin-sdk/gateway-method-runtime` | ابزار کمکی رزروشدهٔ توزیع متد Gateway برای مسیرهای HTTP افزونه که `contracts.gatewayMethodDispatch: ["authenticated-request"]` را اعلام می‌کنند |
    | `plugin-sdk/gateway-runtime` | کلاینت Gateway، ابزار کمکی شروع کلاینت آمادهٔ حلقهٔ رویداد، RPC خط فرمان Gateway، خطاهای پروتکل Gateway، تفکیک میزبان LAN اعلام‌شده و ابزارهای کمکی اصلاح وضعیت کانال |
    | `plugin-sdk/config-contracts` | سطح پیکربندی متمرکز و صرفاً نوعی برای شکل‌های پیکربندی Plugin مانند `OpenClawConfig` و انواع پیکربندی کانال/ارائه‌دهنده |
    | `plugin-sdk/plugin-config-runtime` | نمای سازگاری منسوخ‌شده برای ابزارهای کمکی پیکربندی Plugin در زمان اجرا؛ Pluginهای جدید از `api.pluginConfig` به‌همراه قراردادهای متمرکز پیکربندی، عکس‌های فوری و ابزارهای کمکی تغییر استفاده می‌کنند |
    | `plugin-sdk/config-mutation` | ابزارهای کمکی تغییر تراکنشی پیکربندی مانند `mutateConfigFile`، `replaceConfigFile` و `logConfigUpdated` |
    | `plugin-sdk/message-tool-delivery-hints` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ رشته‌های راهنمای مشترک فرادادهٔ تحویل ابزار پیام |
    | `plugin-sdk/runtime-config-snapshot` | ابزارهای کمکی عکس فوری پیکربندی فرایند جاری مانند `getRuntimeConfig`، `getRuntimeConfigSnapshot` و تنظیم‌کننده‌های عکس فوری آزمون |
    | `plugin-sdk/text-autolink-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ تشخیص پیوند خودکار ارجاع فایل بدون خروجی تجمیعی گستردهٔ متن |
    | `plugin-sdk/reply-runtime` | ابزارهای کمکی مشترک زمان اجرای ورودی/پاسخ، قطعه‌بندی، توزیع، Heartbeat و برنامه‌ریز پاسخ |
    | `plugin-sdk/reply-dispatch-runtime` | ابزارهای کمکی محدودِ توزیع/نهایی‌سازی پاسخ و برچسب مکالمه |
    | `plugin-sdk/reply-history` | ابزارهای کمکی مشترک تاریخچهٔ پاسخ در بازهٔ کوتاه. کد جدید نوبت پیام باید از `createChannelHistoryWindow` استفاده کند؛ ابزارهای کمکی سطح پایین‌تر نقشه فقط به‌عنوان خروجی‌های سازگاری منسوخ‌شده باقی می‌مانند |
    | `plugin-sdk/reply-reference` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ `createReplyReferencePlanner` |
    | `plugin-sdk/reply-chunking` | ابزارهای کمکی محدودِ قطعه‌بندی متن/Markdown |
    | `plugin-sdk/session-store-runtime` | ابزارهای کمکی گردش‌کار نشست (`getSessionEntry`، `listSessionEntries`، `patchSessionEntry`، `upsertSessionEntry`)، ابزارهای کمکی ترمیم/چرخهٔ عمر (`deleteSessionEntry`، `cleanupSessionLifecycleArtifacts`، `resolveSessionStoreBackupPaths`)، ابزارهای کمکی نشانگر برای مقادیر انتقالی `sessionFile`، خواندن محدود متن رونوشت اخیر کاربر/دستیار بر اساس هویت نشست، ابزارهای کمکی مسیر ذخیره‌گاه نشست/کلید نشست و خواندن زمان به‌روزرسانی، بدون واردکردن نگهداشت/نوشتن گستردهٔ پیکربندی |
    | `plugin-sdk/session-transcript-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ هویت رونوشت، مکان‌نماهای محدود خام و قابل‌مشاهده، ابزارهای کمکی هدف/خواندن/نوشتن دامنه‌بندی‌شده، نگاشت ورودی پیام قابل‌مشاهده، انتشار به‌روزرسانی، قفل‌های نوشتن و کلیدهای برخورد حافظهٔ رونوشت |
    | `plugin-sdk/sqlite-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی متمرکز شِما، مسیر و تراکنش SQLite عامل برای زمان اجرای طرف اول، بدون کنترل‌های چرخهٔ عمر پایگاه داده |
    | `plugin-sdk/cron-store-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی مسیر/بارگذاری/ذخیرهٔ مخزن Cron |
    | `plugin-sdk/state-paths` | ابزارهای کمکی مسیر پوشهٔ وضعیت/OAuth |
    | `plugin-sdk/plugin-state-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ قراردادهای وضعیت کلیددارِ دامنه‌بندی‌شده برای Plugin، BLOB و اجارهٔ مشارکتی SQLite، به‌همراه pragma اتصال، نگهداشت تأییدشدهٔ WAL و ابزارهای کمکی مهاجرت اتمی شِمای STRICT. فراخوان‌های بازگشتی اجاره یک سیگنال لغو دریافت می‌کنند و خطاهای نوع‌دار میان پایان مهلت، لغو، ازدست‌رفتن مالکیت، ورودی نامعتبر و خرابی ذخیره‌سازی تمایز می‌گذارند |
    | `plugin-sdk/routing` | ابزارهای کمکی اتصال مسیر/کلید نشست/حساب مانند `resolveAgentRoute`، `buildAgentSessionKey` و `resolveDefaultAgentBoundAccountId` |
    | `plugin-sdk/status-helpers` | ابزارهای کمکی مشترک خلاصهٔ وضعیت کانال/حساب، پیش‌فرض‌های وضعیت زمان اجرا و ابزارهای کمکی فرادادهٔ مشکل |
    | `plugin-sdk/target-resolver-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی مشترک تفکیک هدف |
    | `plugin-sdk/string-normalization-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی نرمال‌سازی شناسهٔ کوتاه/رشته |
    | `plugin-sdk/request-url` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ استخراج URLهای رشته‌ای از ورودی‌های مشابه fetch/request |
    | `plugin-sdk/run-command` | اجراکنندهٔ زمان‌بندی‌شدهٔ فرمان با نتایج نرمال‌شدهٔ stdout/stderr |
    | `plugin-sdk/param-readers` | خواننده‌های رایج پارامتر ابزار/CLI |
    | `plugin-sdk/tool-plugin` | تعریف یک Plugin ساده و نوع‌دار برای ابزار عامل و ارائهٔ فرادادهٔ ایستا برای تولید مانیفست |
    | `plugin-sdk/tool-payload` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ استخراج بارهای دادهٔ نرمال‌شده از اشیای نتیجهٔ ابزار |
    | `plugin-sdk/tool-send` | استخراج فیلدهای متعارف هدف ارسال از آرگومان‌های ابزار |
    | `plugin-sdk/sandbox` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ انواع بک‌اند سندباکس و ابزارهای کمکی فرمان SSH/OpenShell، از جمله پیش‌بررسی فرمان اجرایی با توقف سریع |
    | `plugin-sdk/temp-path` | ابزارهای کمکی مشترک مسیر بارگیری موقت و فضاهای کاری موقت خصوصی و امن |
    | `plugin-sdk/logging-core` | ابزارهای کمکی ثبت‌کنندهٔ زیرسامانه و حذف اطلاعات حساس |
    | `plugin-sdk/markdown-table-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی حالت و تبدیل جدول Markdown |
    | `plugin-sdk/model-session-runtime` | ابزارهای کمکی بازنویسی مدل/نشست مانند `applyModelOverrideToSessionEntry` و `resolveAgentMaxConcurrent` |
    | `plugin-sdk/talk-config-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی تفکیک پیکربندی ارائه‌دهندهٔ گفت‌وگو |
    | `plugin-sdk/json-store` | ابزارهای کمکی کوچک خواندن/نوشتن وضعیت JSON |
    | `plugin-sdk/json-unsafe-integers` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی تجزیهٔ JSON که لیترال‌های عدد صحیح ناامن را به‌صورت رشته حفظ می‌کنند |
    | `plugin-sdk/file-lock` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی قفل فایل بازگشتی، به‌همراه بازپس‌گیری ایمن برای Doctor از فایل‌های جانبی قفل بازنشسته‌ای که قطعاً کهنه و بدون تغییرند |
    | `plugin-sdk/persistent-dedupe` | ابزارهای کمکی حافظهٔ نهان حذف تکرار مبتنی بر دیسک |
    | `plugin-sdk/ingress-effect-once` | محافظ پایدار ادعا/ثبت برای عوارض جانبی ورودی غیرایدِمپوتنت |
    | `plugin-sdk/acp-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی زمان اجرا/نشست ACP و توزیع پاسخ |
    | `plugin-sdk/acp-runtime-backend` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی سبک ثبت بک‌اند ACP و توزیع پاسخ برای Pluginهای بارگذاری‌شده هنگام راه‌اندازی |
    | `plugin-sdk/acp-binding-resolve-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ تفکیک فقط‌خواندنی اتصال ACP بدون واردکردن راه‌اندازی چرخهٔ عمر |
    | `plugin-sdk/agent-config-primitives` | مبانی منسوخ‌شدهٔ شِمای پیکربندی زمان اجرای عامل؛ مبانی شِما را از یک سطح نگه‌داری‌شده و متعلق به Plugin وارد کنید |
    | `plugin-sdk/boolean-param` | خوانندهٔ سهل‌گیر پارامتر بولی |
    | `plugin-sdk/dangerous-name-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی تفکیک تطبیق نام خطرناک |
    | `plugin-sdk/device-bootstrap` | ابزارهای کمکی راه‌اندازی اولیهٔ دستگاه و توکن جفت‌سازی، شامل `BOOTSTRAP_HANDOFF_OPERATOR_SCOPES` |
    | `plugin-sdk/extension-shared` | مبانی مشترک ابزارهای کمکی کانال غیرفعال، وضعیت و پروکسی محیطی |
    | `plugin-sdk/models-provider-runtime` | ابزارهای کمکی پاسخ فرمان/ارائه‌دهندهٔ `/models` |
    | `plugin-sdk/skill-commands-runtime` | ابزارهای کمکی فهرست‌کردن فرمان Skill |
    | `plugin-sdk/native-command-registry` | ابزارهای کمکی رجیستری/ساخت/سریال‌سازی فرمان بومی |
    | `plugin-sdk/agent-harness` | سطح آزمایشی Plugin مورداعتماد برای هارنس‌های سطح پایین عامل: انواع هارنس، ابزارهای کمکی هدایت/لغو اجرای فعال، ابزارهای کمکی پل ابزار OpenClaw، ابزارهای کمکی سیاست ابزار برنامهٔ زمان اجرا، طبقه‌بندی نتیجهٔ پایانه، ابزارهای کمکی قالب‌بندی/جزئیات پیشرفت ابزار و ابزارهای نتیجهٔ تلاش |
    | `plugin-sdk/async-lock-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی قفل ناهمگام محلی فرایند برای فایل‌های کوچک وضعیت زمان اجرا |
    | `plugin-sdk/channel-activity-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی تله‌متری فعالیت کانال |
    | `plugin-sdk/concurrency-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی هم‌روندی محدود وظیفهٔ ناهمگام |
    | `plugin-sdk/dedupe-runtime` | ابزارهای کمکی حافظهٔ نهان حذف تکرار درون‌حافظه‌ای و متکی بر ذخیره‌سازی پایدار |
    | `plugin-sdk/delivery-queue-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی تخلیهٔ تحویل‌های خروجی در انتظار |
    | `plugin-sdk/file-access-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی مسیر امن فایل محلی و منبع رسانه |
    | `plugin-sdk/heartbeat-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی بیدارسازی، رویداد و مشاهده‌پذیری Heartbeat |
    | `plugin-sdk/expect-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی بررسی مقدار الزامی برای ناورداهای اثبات‌پذیر زمان اجرا |
    | `plugin-sdk/number-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی تبدیل اجباری عددی |
    | `plugin-sdk/secure-random-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی امن توکن/UUID |
    | `plugin-sdk/system-event-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی صف رویداد سیستم |
    | `plugin-sdk/transport-ready-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزار کمکی انتظار برای آمادگی انتقال |
    | `plugin-sdk/exec-approvals-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی فایل سیاست تأیید اجرا، بدون خروجی تجمیعی گستردهٔ زمان اجرای زیرساخت |
    | `plugin-sdk/infra-runtime` | شیم سازگاری منسوخ‌شده؛ از زیرمسیرهای متمرکز زمان اجرا در بالا استفاده کنید |
    | `plugin-sdk/collection-runtime` | ابزارهای کمکی حافظهٔ نهان کوچک و محدود |
    | `plugin-sdk/diagnostic-runtime` | ابزارهای کمکی پرچم تشخیصی، رویداد و زمینهٔ ردیابی |
    | `plugin-sdk/error-runtime` | ابزارهای کمکی گراف خطا، قالب‌بندی، تبدیل اجباری مقدار ناشناخته، طبقه‌بندی مشترک خطا، `PlatformMessageNotDispatchedError`، `isApprovalNotFoundError` |
    | `plugin-sdk/fetch-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی fetch پوشانده‌شده، پروکسی، گزینهٔ EnvHttpProxyAgent و جست‌وجوی سنجاق‌شده |
    | `plugin-sdk/runtime-fetch` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ fetch زمان اجرای آگاه از توزیع‌کننده بدون واردکردن proxy/guarded-fetch |
    | `plugin-sdk/inline-image-data-url-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ ابزارهای کمکی پاک‌سازی URL دادهٔ تصویر درون‌خطی و تشخیص امضا، بدون سطح گستردهٔ زمان اجرای رسانه |
    | `plugin-sdk/response-limit-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ خواننده‌های بدنهٔ پاسخ محدودشده بر اساس بایت، بیکاری و مهلت نهایی، بدون سطح گستردهٔ زمان اجرای رسانه |
    | `plugin-sdk/session-binding-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ وضعیت اتصال مکالمهٔ جاری بدون مسیریابی اتصال پیکربندی‌شده یا ذخیره‌گاه‌های جفت‌سازی |
    | `plugin-sdk/context-visibility-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ تفکیک مشاهده‌پذیری زمینه و پالایش زمینهٔ تکمیلی بدون واردکردن گستردهٔ پیکربندی/امنیت |
    | `plugin-sdk/string-coerce-runtime` | ابزارهای کمکی محدودِ تبدیل اجباری و نرمال‌سازی رکورد/رشتهٔ اولیه، بدون واردکردن Markdown/ثبت گزارش |
    | `plugin-sdk/html-entity-runtime` | محلی-خصوصی پس از ژوئیهٔ ۲۰۲۶؛ رمزگشایی تک‌گذری موجودیت HTML5 پایان‌یافته با نقطه‌ویرگول، بدون ابزارهای گستردهٔ متن |
    | `plugin-sdk/text-utility-runtime` | پس از ژوئیهٔ 2026 خصوصی-محلی؛ ابزارهای سطح‌پایین متن و مسیر، شامل گریزدهی HTML برای پنج موجودیت |
    | `plugin-sdk/widget-html` | تشخیص سند کامل، اعتبارسنجی اندازه و خطاهای ورودی ابزار برای ویجت‌های HTML خودبسنده |
    | `plugin-sdk/host-runtime` | پس از ژوئیهٔ 2026 خصوصی-محلی؛ ابزارهای نرمال‌سازی نام میزبان و میزبان SCP |
    | `plugin-sdk/retry-runtime` | پس از ژوئیهٔ 2026 خصوصی-محلی؛ ابزارهای پیکربندی تلاش مجدد و اجراکنندهٔ تلاش مجدد |
    | `plugin-sdk/agent-runtime` | barrel گستردهٔ منسوخ برای ابزارهای دایرکتوری عامل/هویت/فضای کاری، شامل `resolveAgentDir`، `resolveDefaultAgentDir` و خروجی سازگاری منسوخ `resolveOpenClawAgentDir`؛ زیربخش‌های متمرکز عامل/زمان اجرا ترجیح داده می‌شوند |
    | `plugin-sdk/directory-runtime` | پرس‌وجو/حذف موارد تکراری دایرکتوری مبتنی بر پیکربندی |
    | `plugin-sdk/keyed-async-queue` | پس از ژوئیهٔ 2026 خصوصی-محلی؛ `KeyedAsyncQueue` |
  </Accordion>

  <Accordion title="زیرمسیرهای قابلیت و آزمایش">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/media-runtime` | مجموعهٔ گسترده و منسوخ رسانه، شامل `saveRemoteMedia`، `saveResponseMedia`، `readRemoteMediaBuffer` و `fetchRemoteMedia` منسوخ؛ `plugin-sdk/media-store`، `plugin-sdk/media-mime`، `plugin-sdk/outbound-media` و زیرمسیرهای زمان‌اجرای قابلیت را ترجیح دهید، و هنگامی که یک URL باید به رسانهٔ OpenClaw تبدیل شود، پیش از خواندن بافر از کمک‌تابع‌های ذخیره‌سازی استفاده کنید |
    | `plugin-sdk/media-local-roots` | کمک‌تابع‌های متمرکز `getAgentScopedMediaLocalRoots(...)` و آگاه از خط‌مشی `getAgentScopedMediaLocalRootsForSources(...)` برای خواندن رسانهٔ محلی تحت مالکیت Plugin |
    | `plugin-sdk/media-mime` | کمک‌تابع‌های محدود برای نرمال‌سازی MIME، نگاشت پسوند فایل، تشخیص MIME و نوع رسانه |
    | `plugin-sdk/media-store` | کمک‌تابع‌های محدود ذخیره‌سازی رسانه، مانند `saveMediaBuffer` و `saveMediaStream` |
    | `plugin-sdk/media-generation-runtime` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های مشترک جایگزینی هنگام خرابی در تولید رسانه، انتخاب گزینهٔ نامزد و پیام‌رسانی دربارهٔ مدل مفقود |
    | `plugin-sdk/media-understanding` | نمای سازگاری منسوخ برای انواع و کمک‌تابع‌های ارائه‌دهندهٔ درک رسانه؛ ارائه‌دهندگان جدید از طریق API تزریق‌شدهٔ Plugin ثبت می‌شوند و کمک‌تابع‌های درخواست تحت مالکیت Plugin باقی می‌مانند |
    | `plugin-sdk/text-chunking` | قطعه‌بندی متن خروجی و محدوده با حفظ آفست، کمک‌تابع‌های قطعه‌بندی و رندر Markdown، توکن‌سازی تگ HTML با توجه به نقل‌قول، تبدیل جدول Markdown، حذف تگ‌های دستورالعمل و ابزارهای متن ایمن |
    | `plugin-sdk/speech` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهندهٔ گفتار، به‌همراه خروجی‌های دستورالعمل، رجیستری و اعتبارسنجی مخصوص ارائه‌دهنده، سازندهٔ TTS سازگار با OpenAI و کمک‌تابع‌های گفتار |
    | `plugin-sdk/speech-core` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع مشترک ارائه‌دهندهٔ گفتار و خروجی‌های رجیستری، دستورالعمل، نرمال‌سازی و کمک‌تابع‌های گفتار |
    | `plugin-sdk/speech-settings` | سازوکارهای سبک رفع و نرمال‌سازی پیکربندی TTS، بدون رجیستری ارائه‌دهنده یا زمان‌اجرای سنتز |
    | `plugin-sdk/realtime-transcription` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهندهٔ رونویسی بلادرنگ، کمک‌تابع‌های رجیستری و کمک‌تابع مشترک نشست WebSocket |
    | `plugin-sdk/realtime-bootstrap-context` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع راه‌اندازی اولیهٔ نمایهٔ بلادرنگ برای تزریق محدود زمینهٔ `IDENTITY.md`، `USER.md` و `SOUL.md` |
    | `plugin-sdk/realtime-voice` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهندهٔ صدای بلادرنگ، کمک‌تابع‌های رجیستری، دروازه‌های مشترک انرژی صوتی/آغاز گفتار و کمک‌تابع‌های رفتار صدای بلادرنگ، شامل بستر آزمایش نشست مستقل از انتقال و رهگیری فعالیت خروجی |
    | `plugin-sdk/meeting-runtime` | زمان‌اجرای نشست جلسهٔ مرورگر، موتورهای صوتی/انتقال‌های بلادرنگ، `MeetingPlatformAdapter`، کنترل مرورگر/Node، مشاوره با عامل، واگذاری تماس صوتی، بررسی‌های راه‌اندازی و کمک‌تابع‌های فرمان SoX |
    | `plugin-sdk/image-generation` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهندهٔ تولید تصویر، به‌همراه کمک‌تابع‌های دارایی تصویر/URL داده و سازندهٔ ارائه‌دهندهٔ تصویر سازگار با OpenAI |
    | `plugin-sdk/image-generation-core` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع مشترک تولید تصویر و کمک‌تابع‌های جایگزینی هنگام خرابی، احراز هویت و رجیستری |
    | `plugin-sdk/music-generation` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهنده/درخواست/نتیجهٔ تولید موسیقی |
    | `plugin-sdk/video-generation` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع ارائه‌دهنده/درخواست/نتیجهٔ تولید ویدئو |
    | `plugin-sdk/video-generation-core` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع مشترک تولید ویدئو، کمک‌تابع‌های جایگزینی هنگام خرابی، جست‌وجوی ارائه‌دهنده و تجزیهٔ ارجاع مدل |
    | `plugin-sdk/transcripts` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ انواع مشترک ارائه‌دهندهٔ منبع رونوشت، کمک‌تابع‌های رجیستری، کارخانهٔ پل ارائه‌دهندهٔ جلسه، توصیف‌گرهای نشست و فرادادهٔ گفته |
    | `plugin-sdk/webhook-targets` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ رجیستری مقصد Webhook و کمک‌تابع‌های نصب مسیر |
    | `plugin-sdk/web-media` | کمک‌تابع‌های مشترک بارگیری رسانهٔ راه‌دور/محلی |
    | `plugin-sdk/zod` | بازصدور سازگاری منسوخ؛ `zod` را مستقیماً از `zod` وارد کنید |
    | `plugin-sdk/plugin-test-api` | کمک‌تابع حداقلی `createTestPluginApi` و محلی مخزن برای آزمون‌های واحد ثبت مستقیم Plugin، بدون واردکردن پل‌های کمک‌تابع آزمون مخزن |
    | `plugin-sdk/agent-runtime-test-contracts` | فیکسچرهای محلی مخزن برای قرارداد آداپتور بومی زمان‌اجرای عامل، جهت آزمون‌های احراز هویت، تحویل، جایگزینی، قلاب ابزار، هم‌پوشانی پرامپت، طرح‌واره و تصویرسازی رونوشت |
    | `plugin-sdk/channel-test-helpers` | کمک‌تابع‌های آزمون کانال‌محور و محلی مخزن برای قراردادهای عمومی کنش/راه‌اندازی/وضعیت، وارسی‌های دایرکتوری، چرخهٔ حیات راه‌اندازی حساب، عبور پیکربندی ارسال، ماک‌های زمان‌اجرا، مشکلات وضعیت، تحویل خروجی و ثبت قلاب |
    | `plugin-sdk/channel-target-testing` | مجموعهٔ مشترک و محلی مخزن از حالت‌های خطای رفع مقصد برای آزمون‌های کانال |
    | `plugin-sdk/channel-contract-testing` | کمک‌تابع‌های آزمون محدود قرارداد کانال و محلی مخزن، بدون مجموعهٔ گستردهٔ آزمایش |
    | `plugin-sdk/plugin-test-contracts` | کمک‌تابع‌های محلی مخزن برای قراردادهای بستهٔ Plugin، ثبت، مصنوع عمومی، واردکردن مستقیم، API زمان‌اجرا و اثر جانبی واردکردن |
    | `plugin-sdk/plugin-state-test-runtime` | کمک‌تابع‌های آزمون محلی مخزن برای ذخیره‌گاه وضعیت Plugin، صف ورودی و پایگاه دادهٔ وضعیت |
    | `plugin-sdk/provider-test-contracts` | کمک‌تابع‌های محلی مخزن برای قراردادهای زمان‌اجرای ارائه‌دهنده، احراز هویت، کشف، ورود اولیه، کاتالوگ، جادوگر، قابلیت رسانه، خط‌مشی بازپخش، صوت زندهٔ STT بلادرنگ، جست‌وجو/واکشی وب و جریان |
    | `plugin-sdk/provider-http-test-mocks` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ ماک‌های اختیاری HTTP/احراز هویت Vitest و محلی مخزن برای آزمون‌های ارائه‌دهنده‌ای که `plugin-sdk/provider-http` را به‌کار می‌گیرند |
    | `plugin-sdk/reply-payload-testing` | کمک‌تابع‌های محلی مخزن برای پیوست‌کردن فراداده به فیکسچرهای محمولهٔ پاسخ |
    | `plugin-sdk/sqlite-runtime-testing` | کمک‌تابع‌های محلی مخزن برای چرخهٔ حیات SQLite در آزمون‌های شخص‌اول |
    | `plugin-sdk/test-fixtures` | فیکسچرهای محلی مخزن برای ضبط عمومی زمان‌اجرای CLI، زمینهٔ sandbox، نویسندهٔ skill، پیام عامل، رویداد سیستم، بارگذاری مجدد ماژول، مسیر Plugin همراه، متن پایانه، قطعه‌بندی، توکن احراز هویت و حالت‌های نوع‌دار |
    | `plugin-sdk/test-node-mocks` | کمک‌تابع‌های متمرکز و محلی مخزن برای ماک‌کردن امکانات داخلی Node جهت استفاده در کارخانه‌های `vi.mock("node:*")` در Vitest |
  </Accordion>

  <Accordion title="زیرمسیرهای حافظه">
    | زیرمسیر | خروجی‌های کلیدی |
    | --- | --- |
    | `plugin-sdk/memory-core-host-embedding-registry` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های سبک رجیستری ارائه‌دهندهٔ جاسازی حافظه |
    | `plugin-sdk/memory-core-host-engine-foundation` | خروجی‌های موتور پایهٔ میزبان حافظه |
    | `plugin-sdk/memory-core-host-engine-embeddings` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ قراردادهای جاسازی میزبان حافظه، دسترسی به رجیستری، ارائه‌دهندهٔ محلی و کمک‌تابع‌های عمومی دسته‌ای/راه‌دور. `registerMemoryEmbeddingProvider` در این سطح منسوخ است؛ برای ارائه‌دهندگان جدید از API عمومی ارائه‌دهندهٔ جاسازی استفاده کنید. |
    | `plugin-sdk/memory-core-host-engine-qmd` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ خروجی‌های موتور QMD میزبان حافظه |
    | `plugin-sdk/memory-core-host-engine-storage` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ خروجی‌های موتور ذخیره‌سازی میزبان حافظه |
    | `plugin-sdk/memory-core-host-secret` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های محرمانهٔ میزبان حافظه |
    | `plugin-sdk/memory-core-host-status` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های وضعیت میزبان حافظه |
    | `plugin-sdk/memory-core-host-runtime-cli` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های زمان‌اجرای CLI میزبان حافظه |
    | `plugin-sdk/memory-core-host-runtime-core` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های اصلی زمان‌اجرای میزبان حافظه |
    | `plugin-sdk/memory-core-host-runtime-files` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های فایل/زمان‌اجرای میزبان حافظه |
    | `plugin-sdk/memory-host-core` | نمای سازگاری منسوخ برای کمک‌تابع‌های مستقل از فروشندهٔ میزبان حافظه. Pluginهای جدید حافظه از قابلیت‌های تزریق‌شدهٔ حافظه و پرامپت‌های آماده‌شده توسط میزبان استفاده می‌کنند؛ Pluginهای همراه تا زمانی که یک درگاه خواندن متمرکز وجود داشته باشد، برای کشف مصنوع عمومی همچنان از نمای حفظ‌شده استفاده می‌کنند. |
    | `plugin-sdk/memory-host-events` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ نام مستعار مستقل از فروشنده برای کمک‌تابع‌های دفتر رویداد میزبان حافظه |
    | `plugin-sdk/memory-host-markdown` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع‌های مشترک Markdown مدیریت‌شده برای Pluginهای مرتبط با حافظه |
    | `plugin-sdk/memory-host-search` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ نمای زمان‌اجرای Active Memory برای دسترسی به مدیر جست‌وجو |
  </Accordion>

  <Accordion title="زیرمسیرهای رزروشدهٔ کمک‌تابع‌های همراه">
    زیرمسیرهای SDK رزروشدهٔ کمک‌تابع‌های همراه، سطوحی محدود و ویژهٔ مالک برای
    کد Plugin همراه هستند. آن‌ها در فهرست موجودی SDK رهگیری می‌شوند تا ساخت
    بسته‌ها و نام‌گذاری مستعار قطعی بماند، اما APIهای عمومی برای
    نگارش Plugin نیستند. قراردادهای میزبان جدید و قابل‌استفادهٔ مجدد باید از زیرمسیرهای عمومی SDK،
    مانند `plugin-sdk/gateway-runtime` و `plugin-sdk/ssrf-runtime`، استفاده کنند.

    | زیرمسیر | مالک و هدف |
    | --- | --- |
    | `plugin-sdk/codex-mcp-projection` | پس از ژوئیهٔ 2026 فقط محلی و خصوصی؛ کمک‌تابع Plugin همراه Codex برای تصویرسازی پیکربندی سرور MCP کاربر در پیکربندی رشتهٔ app-server در Codex (خروجی رزروشدهٔ بسته) |
    | `plugin-sdk/codex-native-task-runtime` | کمک‌تابع Plugin همراه Codex برای بازتاب زیرعامل‌های بومی app-server در Codex در وضعیت وظیفهٔ OpenClaw (فقط محلی مخزن، نه خروجی بسته) |

  </Accordion>
</AccordionGroup>

## مرتبط

- [نمای کلی SDKِ Plugin](/fa/plugins/sdk-overview)
- [راه‌اندازی SDKِ Plugin](/fa/plugins/sdk-setup)
- [ساخت Pluginها](/fa/plugins/building-plugins)
