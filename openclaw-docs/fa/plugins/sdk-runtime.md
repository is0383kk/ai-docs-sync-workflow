---
read_when:
    - لازم است helperهای هسته را از یک Plugin فراخوانی کنید (TTS، STT، تولید تصویر، جست‌وجوی وب، Gateway، زیرعامل، نودها)
    - می‌خواهید بدانید `api.runtime` چه چیزهایی را ارائه می‌دهد
    - شما از کد Plugin به ابزارهای کمکی پیکربندی، عامل یا رسانه دسترسی دارید
sidebarTitle: Runtime helpers
summary: api.runtime -- کمک‌تابع‌های زمان اجرا که به Pluginها تزریق می‌شوند
title: ابزارهای کمکی زمان اجرای Plugin
x-i18n:
    generated_at: "2026-07-27T15:33:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff1d901de8ec70011eeaafbab7b3cc30709fc95894c7ba4f4346c026de682cd0
    source_path: plugins/sdk-runtime.md
    workflow: 16
---

مرجع شیء `api.runtime` که هنگام ثبت در هر Plugin تزریق می‌شود. به‌جای واردکردن مستقیم اجزای داخلی میزبان، از این ابزارهای کمکی استفاده کنید.

<CardGroup cols={2}>
  <Card title="Pluginهای کانال" href="/fa/plugins/sdk-channel-plugins">
    راهنمای گام‌به‌گامی که کاربرد این ابزارهای کمکی را در زمینهٔ Pluginهای کانال نشان می‌دهد.
  </Card>
  <Card title="Pluginهای ارائه‌دهنده" href="/fa/plugins/sdk-provider-plugins">
    راهنمای گام‌به‌گامی که کاربرد این ابزارهای کمکی را در زمینهٔ Pluginهای ارائه‌دهنده نشان می‌دهد.
  </Card>
</CardGroup>

```typescript
register(api) {
  const runtime = api.runtime;
}
```

`api.runtime.version` نسخهٔ فعلی محصول OpenClaw است که از حل‌کنندهٔ نسخهٔ مشترک دریافت می‌شود تا Pluginها همان مقداری را ببینند که CLI گزارش می‌کند.

## بارگذاری و نوشتن پیکربندی

پیکربندی‌ای را ترجیح دهید که از قبل به مسیر فراخوانی فعال ارسال شده است؛ برای مثال، `api.config` هنگام ثبت یا آرگومان `cfg` در callbackهای کانال/ارائه‌دهنده. با این روش، به‌جای تجزیهٔ مجدد پیکربندی در مسیرهای پرتکرار، یک snapshot فرایند در سراسر عملیات جریان می‌یابد.

از `api.runtime.config.current()` فقط زمانی استفاده کنید که یک کنترل‌کنندهٔ طولانی‌عمر به snapshot فعلی فرایند نیاز دارد و هیچ پیکربندی‌ای به آن تابع ارسال نشده است. مقدار بازگشتی فقط‌خواندنی است؛ پیش از ویرایش، آن را clone کنید یا از یک ابزار کمکی جهش استفاده کنید.

کارخانه‌های ابزار، `ctx.runtimeConfig` را به‌همراه `ctx.getRuntimeConfig()` دریافت می‌کنند. اگر پیکربندی ممکن است پس از ایجاد تعریف ابزار تغییر کند، getter را در callback `execute` ابزار طولانی‌عمر به‌کار ببرید.

تغییرات را با `api.runtime.config.mutateConfigFile(...)` یا `api.runtime.config.replaceConfigFile(...)` ماندگار کنید. هر نوشتن باید یک سیاست صریح `afterWrite` انتخاب کند:

- `afterWrite: { mode: "auto" }` به برنامه‌ریز بارگذاری مجدد Gateway اجازهٔ تصمیم‌گیری می‌دهد.
- `afterWrite: { mode: "restart", reason: "..." }` هنگامی که نویسنده می‌داند بارگذاری مجدد آنی ناامن است، راه‌اندازی مجدد پاک را اجباری می‌کند.
- `afterWrite: { mode: "none", reason: "..." }` فقط زمانی بارگذاری مجدد/راه‌اندازی مجدد خودکار را متوقف می‌کند که فراخواننده مسئول اقدام بعدی باشد.

ابزارهای کمکی جهش، `afterWrite` را به‌همراه خلاصهٔ نوع‌دار `followUp` برمی‌گردانند تا فراخوانندگان بتوانند ثبت یا آزمایش کنند که آیا درخواست راه‌اندازی مجدد داده‌اند. زمان انجام واقعی آن راه‌اندازی مجدد همچنان در اختیار Gateway است.

برای دسترسی و نوشتن پیکربندی زمان اجرا، از `current()`، یک `cfg` ارسال‌شده، `mutateConfigFile(...)` یا
`replaceConfigFile(...)` استفاده کنید.

برای importهای مستقیم SDK، زیرمسیرهای متمرکز پیکربندی را به barrel سازگاری گستردهٔ `openclaw/plugin-sdk/config-runtime` ترجیح دهید: `config-contracts` برای نوع‌ها، `runtime-config-snapshot` برای snapshotهای فعلی فرایند و `config-mutation` برای نوشتن‌ها. مقادیر محدود به ورودی را از `api.pluginConfig` بخوانید؛ از context ابزار ارائه‌شده فقط برای snapshot پیکربندی سراسری زمان اجرای آن استفاده کنید و ادغام مختص Plugin را در همان مرز نگه دارید. آزمون‌های Pluginهای همراه باید به‌جای mockکردن barrel سازگاری گسترده، مستقیماً این زیرمسیرهای متمرکز را mock کنند.

کد داخلی زمان اجرای OpenClaw نیز همین رویکرد را دنبال می‌کند: پیکربندی را یک‌بار در مرز CLI، Gateway یا فرایند بارگذاری کنید و سپس همان مقدار را در سراسر مسیر عبور دهید. نوشتن‌های جهش موفق، snapshot زمان اجرای فرایند را تازه و بازبینی داخلی آن را جلو می‌برند؛ cacheهای طولانی‌عمر باید به‌جای serializeکردن محلی پیکربندی، بر اساس کلید cache متعلق به زمان اجرا کلیدگذاری شوند. ماژول‌های طولانی‌عمر زمان اجرا یک اسکنر با عدم‌تحمل مطلق برای فراخوانی‌های محیطی `loadConfig()` دارند؛ از یک `cfg` ارسال‌شده، `context.getRuntimeConfig()` درخواست یا `getRuntimeConfig()` در مرز صریح فرایند استفاده کنید.

مسیرهای اجرای ارائه‌دهنده و کانال باید از snapshot فعال پیکربندی زمان اجرا استفاده کنند، نه snapshot فایلی که برای بازخوانی یا ویرایش پیکربندی بازگردانده شده است. snapshotهای فایل، مقادیر منبع مانند نشانگرهای SecretRef را برای رابط کاربری و نوشتن‌ها حفظ می‌کنند؛ callbackهای ارائه‌دهنده به نمای حل‌شدهٔ زمان اجرا نیاز دارند. اگر ممکن است یک ابزار کمکی با snapshot فعال منبع یا snapshot فعال زمان اجرا فراخوانی شود، پیش از خواندن اطلاعات احراز هویت، مسیر را از `selectApplicableRuntimeConfig()` عبور دهید.

## ابزارهای زمان اجرای قابل‌استفادهٔ مجدد

برای پیام‌های ورودی ایجادشده توسط ربات، از واقعیت‌های ورودی `botLoopProtection` استفاده کنید. هسته، محافظ مشترک پنجرهٔ لغزان درون‌حافظه‌ای را پیش از ثبت نشست و ارسال اعمال می‌کند، بدون اینکه سیاست را به یک کانال وابسته کند. این محافظ کلیدهای `(scopeId, conversationId, participant pair)` را ردیابی می‌کند، هر دو جهت یک جفت را با هم می‌شمارد، پس از عبور از بودجهٔ پنجره دورهٔ انتظار اعمال می‌کند و ورودی‌های غیرفعال را در فرصت‌های مناسب هرس می‌کند.

Pluginهای کانالی که این رفتار را در اختیار اپراتورها قرار می‌دهند، باید شکل مشترک `channels.defaults.botLoopProtection` را برای بودجه‌های پایه ترجیح دهند و سپس بازنویسی‌های مختص کانال/ارائه‌دهنده را روی آن اعمال کنند. پیکربندی مشترک از ثانیه استفاده می‌کند، زیرا برای کاربر قابل‌مشاهده است:

```typescript
type ChannelBotLoopProtectionConfig = {
  enabled?: boolean;
  maxEventsPerWindow?: number;
  windowSeconds?: number;
  cooldownSeconds?: number;
};
```

واقعیت‌های نرمال‌شدهٔ جفت ربات را همراه با نوبت حل‌شده ارسال کنید. هسته، پیش‌فرض‌ها، تبدیل واحد و معنای `enabled` را حل می‌کند:

```typescript
return {
  channel: "example",
  routeSessionKey,
  storePath,
  ctxPayload,
  recordInboundSession,
  runDispatch,
  botLoopProtection: {
    scopeId: "account-1",
    conversationId: "channel-1",
    senderId: "bot-a",
    receiverId: "bot-b",
    config: channelConfig.botLoopProtection,
    defaultsConfig: runtimeConfig.channels?.defaults?.botLoopProtection,
    defaultEnabled: allowBotsMode !== "off",
  },
};
```

از `openclaw/plugin-sdk/pair-loop-guard-runtime` مستقیماً فقط برای حلقه‌های رویداد سفارشی
دونفره‌ای استفاده کنید که از اجراکنندهٔ مشترک پاسخ ورودی عبور نمی‌کنند.

## فضای نام‌های زمان اجرا

<AccordionGroup>
  <Accordion title="api.runtime.agent">
    هویت عامل، پوشه‌ها و مدیریت نشست.

    ```typescript
    // پوشهٔ کاری عامل را حل می‌کند (agentId الزامی است)
    const agentDir = api.runtime.agent.resolveAgentDir(cfg, agentId);

    // فضای کاری عامل را حل می‌کند
    const workspaceDir = api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId);

    // هویت عامل را دریافت می‌کند
    const identity = api.runtime.agent.resolveAgentIdentity(cfg);

    // سطح پیش‌فرض تفکر را دریافت می‌کند
    const thinking = api.runtime.agent.resolveThinkingDefault({
      cfg,
      provider,
      model,
    });

    // سطح تفکر ارائه‌شده توسط کاربر را در برابر نمایهٔ فعال ارائه‌دهنده اعتبارسنجی می‌کند
    const policy = api.runtime.agent.resolveThinkingPolicy({ provider, model });
    const level = api.runtime.agent.normalizeThinkingLevel("extra high");
    if (level && policy.levels.some((entry) => entry.id === level)) {
      // سطح را به یک اجرای تعبیه‌شده ارسال می‌کند
    }

    // مهلت زمانی عامل را دریافت می‌کند
    const timeoutMs = api.runtime.agent.resolveAgentTimeoutMs(cfg);

    // از وجود فضای کاری اطمینان حاصل می‌کند
    await api.runtime.agent.ensureAgentWorkspace(cfg);

    // یک نوبت عامل تعبیه‌شده را اجرا می‌کند
    const result = await api.runtime.agent.runEmbeddedAgent({
      sessionId: "my-plugin:task-1",
      runId: crypto.randomUUID(),
      workspaceDir: api.runtime.agent.resolveAgentWorkspaceDir(cfg, agentId),
      prompt: "آخرین تغییرات را خلاصه کن",
      timeoutMs: api.runtime.agent.resolveAgentTimeoutMs(cfg),
    });
    ```

    `runEmbeddedAgent(...)` ابزار کمکی خنثی برای آغاز یک نوبت عادی عامل OpenClaw از کد Plugin است. این ابزار از همان حل ارائه‌دهنده/مدل و انتخاب چارچوب عامل استفاده می‌کند که پاسخ‌های آغازشده از کانال به‌کار می‌برند.

    `runEmbeddedPiAgent(...)` به‌عنوان نام مستعار سازگاری منسوخ برای Pluginهای موجود باقی می‌ماند. کد جدید باید از `runEmbeddedAgent(...)` استفاده کند.

    `resolveCliBackendDispatchEligibility({ provider, model, agentId, authProfileId, config, agentDir, workspaceDir })` تصمیم ارسال به backend مبتنی بر CLI اجراکنندهٔ تعبیه‌شده را با فراخوانندگانی به‌اشتراک می‌گذارد که اجرای تعبیه‌شده را در `cliBackendDispatch: "subscription-auth"` فعال می‌کنند؛ این تصمیم شامل مسیر، قابلیت اعلام‌شدهٔ `subscriptionAuthDispatch` توسط backend و حالت ذخیره‌شدهٔ اطلاعات احراز هویت است و `authProfileId` صراحتاً تثبیت‌شده را رعایت می‌کند. اگر اجرا از طریق backend مبتنی بر CLI انجام شود، `{ provider }` و اگر در مسیر عبور مستقیم باقی بماند، `undefined` را برمی‌گرداند تا فراخوانندگان بتوانند مهلت‌های زمانی را برای اجرایی که واقعاً انجام خواهد شد بودجه‌بندی کنند.

    `resolveThinkingPolicy(...)` سطوح تفکر پشتیبانی‌شده و پیش‌فرض اختیاری ارائه‌دهنده/مدل را برمی‌گرداند. Pluginهای ارائه‌دهنده از طریق hookهای تفکر خود مالک نمایهٔ مختص مدل هستند؛ بنابراین Pluginهای ابزار باید به‌جای import یا تکرار فهرست‌های ارائه‌دهنده، این ابزار کمکی زمان اجرا را فراخوانی کنند.

    `normalizeThinkingLevel(...)` متن کاربر مانند `on`، `x-high` یا `extra high` را پیش از بررسی در برابر سیاست حل‌شده، به سطح ذخیره‌شدهٔ معیار تبدیل می‌کند.

    **ابزارهای کمکی مخزن نشست** زیر `api.runtime.agent.session` قرار دارند:

    ```typescript
    const entry = api.runtime.agent.session.getSessionEntry({ agentId, sessionKey });
    for (const { sessionKey, entry } of api.runtime.agent.session.listSessionEntries({ agentId })) {
      // بدون وابستگی به شکل قدیمی sessions.json، ردیف‌های نشست را پیمایش می‌کند.
    }
    await api.runtime.agent.session.patchSessionEntry({
      agentId,
      sessionKey,
      update: (entry) => ({ thinkingLevel: "high" }),
    });

    const created = await api.runtime.agent.session.createSessionEntry({
      cfg,
      key: "agent:main:my-plugin:task-1",
      initialEntry: {
        agentHarnessId: "my-harness",
        modelSelectionLocked: true,
        pluginExtensions: { "my-plugin": { phase: "initializing" } },
      },
      afterCreate: async () => ({
        pluginExtensions: { "my-plugin": { phase: "ready" } },
      }),
    });

    const storePath = api.runtime.agent.session.resolveStorePath(cfg.session?.store, { agentId });
    await api.runtime.agent.session.runWithWorkAdmission(
      { storePath, sessionKey },
      async (signal) => {
        // نشست را ایجاد یا به‌روزرسانی می‌کند، سپس signal را به اجرای عامل پذیرفته‌شده ارسال می‌کند.
      },
    );
    ```

    برای گردش‌کارهای نشست، `getSessionEntry(...)`، `listSessionEntries(...)`، `patchSessionEntry(...)` یا `upsertSessionEntry(...)` را ترجیح دهید. این ابزارهای کمکی نشست‌ها را بر اساس هویت عامل/نشست نشانی‌دهی می‌کنند تا Pluginها به شکل ذخیره‌سازی قدیمی `sessions.json` وابسته نباشند. از `preserveActivity: true` برای patchهای صرفاً فراداده‌ای استفاده کنید که نباید فعالیت نشست را تازه کنند و از `replaceEntry: true` فقط زمانی استفاده کنید که callback یک ورودی کامل برمی‌گرداند و فیلدهای حذف‌شده باید حذف‌شده باقی بمانند. مسیرهای Doctor و مهاجرت می‌توانند `fallbackEntry`، `skipMaintenance` و `requireWriteSuccess` را برای یک ترمیم اتمی مخزن معیار ترکیب کنند.

    `createSessionEntry(...)` یک ردیف نشست معیار و رونوشت جدید ایجاد می‌کند. سطح قابل‌اعتماد `initialEntry` آن عمداً محدود است: یک `agentHarnessId` غیرخالی، `modelSelectionLocked: true` اختیاری و `pluginExtensions` اختیاری. زمان اجرای تزریق‌شده از طریق `registerAgentHarness(...)` فقط شناسه‌های چارچوب متعلق به Plugin فراخواننده را می‌پذیرد؛ این یک ناوردای مالکیت است، نه sandbox میان Pluginهای درون‌فرایندی. این ابزار ردیف موجود را رد می‌کند؛ `label` و `spawnedCwd` به‌جای patchهای ورودی قابل‌اعتماد، فیلدهای ایجاد جداگانه هستند.

    ایجاد، حصار جهش چرخهٔ عمر نشست را از طریق `afterCreate` نگه می‌دارد؛ بنابراین کار جدید تا پایان مقداردهی اولیهٔ متعلق به Plugin منتظر می‌ماند و وجود کار پذیرفته‌شدهٔ قبلی باعث شکست ایجاد می‌شود. callback یک clone از وضعیت ایجادشده دریافت می‌کند. اگر patch برگرداند، آن patch فقط می‌تواند شامل `pluginExtensions` باشد و مقدار آن، فیلد نهایی و کامل `pluginExtensions` است. شکست callback یا ماندگارسازی نهایی، ردیف و رونوشت جدیدِ بدون تغییر را rollback می‌کند؛ rollback محافظت‌شده، ردیفی را که هم‌زمان تغییر کرده یا تصاحب شده است حفظ می‌کند. `recoverMatchingInitialEntry: true` فقط برای تلاش دوباره جهت مقداردهی اولیهٔ قطع‌شده است، آن هم زمانی که فیلدهای قابل‌اعتماد ماندگارشده دقیقاً مطابقت دارند؛ بازیابی نیز مستلزم آن است که `afterCreate` یک patch نهایی برگرداند.

    هنگامی که یک Plugin کار روی نشست ماندگارشده را آغاز می‌کند، از `runWithWorkAdmission(...)` استفاده کنید. callback نشست‌های بایگانی‌شده یا جایگزین‌شدهٔ هم‌زمان را رد می‌کند، جهش‌های بایگانی/بازنشانی/حذف را تا پایان هماهنگ نگه می‌دارد و یک `AbortSignal` دریافت می‌کند که باید به اجرای عامل منتقل شود. یک چارچوب می‌تواند نمایندگان اجرای قابل‌اعتماد را صراحتاً از طریق فیلد ثبت آزمایشی `delegatedExecutionPluginIds` خود نام ببرد. نمایندگان فقط می‌توانند یک نشست دقیق، موجود و قفل‌شده روی مدل را بپذیرند و اجرا کنند؛ همهٔ جهش‌های نشست همچنان به مالک چارچوب محدود می‌مانند. به [Pluginهای چارچوب عامل](/fa/plugins/sdk-agent-harness#delegated-execution) مراجعه کنید.

    Plugin‌های نگه‌داری و تعمیر می‌توانند از `deleteSessionEntry(...)` برای یک ورودی نشست با دامنه مشخص، از `cleanupSessionLifecycleArtifacts(...)` برای نشست‌های موقت تحت مالکیت چرخهٔ عمر، و از `resolveSessionStoreBackupPaths(...)` پیش از تغییر یک مخزن استفاده کنند. هنگامی که حذف نباید با به‌روزرسانی هم‌زمان نشست دچار وضعیت رقابتی شود، `expectedSessionId` و `expectedUpdatedAt` را ارسال کنید؛ هنگامی که تصویر لحظه‌ای قبلی فاقد شناسهٔ نشست بود، از `expectedSessionId: null` استفاده کنید. این توابع کمکی، سطوح محدود تعمیر/چرخهٔ عمر هستند، نه یک API عمومی برای حذف از مخزن.

    `resolveStorePath(...)` و `updateSessionStoreEntry(...)` مجموعهٔ توابع کمکی نشست را تکمیل می‌کنند: `resolveStorePath` مسیر مخزن نشست را برای دامنه‌ای معین تعیین می‌کند و `updateSessionStoreEntry({ storePath, sessionKey, update })` هنگامی که فراخواننده از قبل مسیر مخزن را می‌داند، یک ورودی را مستقیماً بر اساس مسیر مخزن اصلاح می‌کند.

    `loadTranscriptEventsSync(...)` برای مسیرهای همگام doctor و تعمیر که نمی‌توانند از زمان‌اجرای ناهمگام رونوشت استفاده کنند، در دسترس است. این تابع رکوردهای خام `SessionStoreTranscriptEvent` را برمی‌گرداند. کد عادی زمان‌اجرای Plugin باید `openclaw/plugin-sdk/session-transcript-runtime` را ترجیح دهد.

    `formatSqliteSessionFileMarker(...)`، `parseSqliteSessionFileMarker(...)` و `sqliteSessionFileMarkerMatchesSession(...)` توابع کمکی گذار برای کدی هستند که همچنان فیلدی قدیمی با نام `sessionFile` دریافت می‌کند. یک نشانگر SQLite تجزیه‌شده، مقصد زندهٔ رونوشت SQLite را مشخص می‌کند؛ این نشانگر مسیر سیستم فایل نیست. APIهای جدید باید به‌جای رشته‌های نشانگر، هویت نشست نوع‌دار را حمل کنند.

    برای خواندن و نوشتن رونوشت، `openclaw/plugin-sdk/session-transcript-runtime` را وارد کنید و از `resolveSessionTranscriptIdentity(...)`، `resolveSessionTranscriptTarget(...)`، `readSessionTranscriptEvents(...)`، `readSessionTranscriptRawDelta(...)`، `readSessionTranscriptVisibleMessageDelta(...)`، `readVisibleSessionTranscriptMessageEntries(...)`، `appendSessionTranscriptMessageByIdentity(...)`، `publishSessionTranscriptUpdateByIdentity(...)` یا `withSessionTranscriptWriteLock(...)` همراه با `{ agentId, sessionKey, sessionId }` استفاده کنید. این APIها به Plugin‌ها امکان می‌دهند یک رونوشت را شناسایی کنند، رویدادهای خام یا ورودی‌های پیام قابل‌مشاهده و ایمن نسبت به شاخه را بخوانند، پیام‌ها را بیفزایند، به‌روزرسانی‌ها را منتشر کنند و عملیات مرتبط را زیر همان قفل نوشتن رونوشت اجرا کنند، بدون آنکه به مسیرهای فایل رونوشت فعال وابسته باشند. `readVisibleSessionTranscriptMessageEntries(...)` فرادادهٔ خواندن مرتب‌شده را برمی‌گرداند؛ فیلد `seq` آن مکان‌نمای قابل‌ازسرگیری نیست.

    `appendSessionTranscriptMessageByIdentity(...)` عملیاتی سطح‌پایین برای افزودن پیامی است که از قبل به شکل استاندارد درآمده است. Plugin‌ها نباید ردیف‌های کاربر دارای رسانه را با `MediaPath`، `MediaPaths`، `MediaUrl`، `MediaUrls`، `MediaType` یا `MediaTypes` در سطح بالا بسازند. ورودی کانال باید واقعیت‌های مرتب‌شده را از طریق `MsgContext.media` ارسال کند و مالکیت ماندگار‌سازی نوبت کاربر را به میزبان بسپارد. پیام کاربر ماندگار‌شده‌ای که میزبان آماده کرده است، واقعیت‌های مرتب‌شدهٔ استاندارد را زیر `message.__openclaw.media` حمل می‌کند؛ API عمومی افزودن، آرایه‌های موازی قدیمی را استنباط یا ترمیم نمی‌کند.

    `readSessionTranscriptRawDelta(...)` یک نتیجهٔ محدود از نوع `page`، `reset` یا `missing` برمی‌گرداند. `page.cursor` مات را به فراخوانی بعدی ارسال کنید. افزودن‌های صرف، مکان‌نما را حفظ می‌کنند، درحالی‌که جایگزینی رونوشت، `reset` را همراه با یک مکان‌نمای راه‌اندازی اولیهٔ جدید برمی‌گرداند. صفحه‌ها به‌طور پیش‌فرض شامل 1,000 رویداد و 1,000,000 بایت سریال‌شده هستند؛ فراخوانندگان می‌توانند حداکثر 10,000 رویداد و 64 MiB درخواست کنند. هنگامی که فقط رویداد بعدی از `maxBytes` فراتر می‌رود، صفحه خالی است و `requiredBytes` را گزارش می‌کند؛ اگر حد بایت موردنیاز از 64 MiB بیشتر نیست، با حدی دست‌کم برابر با آن دوباره تلاش کنید. رویدادهای منفرد بزرگ‌تر به API خواندن کامل نیاز دارند. مکان‌نما فقط موقعیت را مشخص می‌کند و هرگز دسترسی به نشست دیگری اعطا نمی‌کند.

    `readSessionTranscriptVisibleMessageDelta(...)` همان ساختار محدود راه‌اندازی اولیه و ازسرگیری را روی نمای پیام فعالِ تحت مالکیت میزبان فراهم می‌کند. این تابع پیام‌ها را از قدیمی‌ترین به جدیدترین برمی‌گرداند تا موتورهای زمینه بتوانند تاریخچهٔ اولیه را تخلیه و مکان‌نمای مات را به‌عنوان نشان حد پیشرفت خود ماندگار کنند. مکان‌نما را بدون تغییر ذخیره و بازگردانید؛ این یک راهنمای ادامه است، نه اعتبارنامهٔ مجوزدهی. افزودن‌های خطی پس از آخرین پیام بازگردانده‌شده از سر گرفته می‌شوند. جایگزینی رونوشت، مکان‌نمایی که لنگر آن از شاخهٔ فعال خارج شده یا درون آن جابه‌جا شده است، مکان‌نماهای نادرست و مکان‌نماهای بین‌نشستی، `reset` را همراه با یک مکان‌نمای راه‌اندازی اولیهٔ تازه برمی‌گردانند. مقادیر پیش‌فرض و سقف‌های شمارش و بایت با API دلتای خام یکسان‌اند. هنگامی که نمای فعال پس از تغییر شاخه در حال بازسازی است، نتیجه `unavailable` با دلیل `projection_rebuilding` است؛ به‌جای بازگشت به فایل رونوشت فعال، بعداً دوباره تلاش کنید.

    توابع کمکی قدیمی مربوط به کل مخزن و فایل رونوشت فعال دیگر از SDK مربوط به Plugin صادر نمی‌شوند. برای فرادادهٔ نشست از توابع کمکی ورودی با دامنه مشخص و برای عملیات رونوشت فعال از توابع کمکی هویت رونوشت استفاده کنید. گردش‌کارهای بایگانی/پشتیبانی که به مصنوعات فایلی نیاز دارند، باید به‌جای APIهای زمان‌اجرای نشست فعال از سطوح اختصاصی بایگانی خود استفاده کنند.

  </Accordion>
  <Accordion title="api.runtime.agent.defaults">
    ثابت‌های مدل و ارائه‌دهندهٔ پیش‌فرض:

    ```typescript
    const model = api.runtime.agent.defaults.model; // برای نمونه "gpt-5.6-sol"
    const provider = api.runtime.agent.defaults.provider; // برای نمونه "openai"
    ```

  </Accordion>

  <Accordion title="api.runtime.llm">
    یک تکمیل متن تحت مالکیت میزبان را بدون واردکردن اجزای داخلی ارائه‌دهنده یا
    تکرار آماده‌سازی مدل/احراز هویت/نشانی پایهٔ OpenClaw اجرا کنید.

    ```typescript
    const result = await api.runtime.llm.complete({
      messages: [{ role: "user", content: "این رونوشت را خلاصه کن." }],
      purpose: "my-plugin.summary",
      maxTokens: 512,
      temperature: 0.2,
      reasoning: "high",
    });
    ```

    هماهنگ‌سازی ارائه‌دهنده همچنین می‌تواند پیش از ارسال یک درخواست HTTP،
    چرخهٔ عمر سرویس محلی پیکربندی‌شده را در اختیار بگیرد:

    ```typescript
    const lease = await api.runtime.llm.acquireLocalService(
      {
        providerId,
        baseUrl,
        headers,
      },
      signal,
    );
    try {
      // درخواست ارائه‌دهنده را ارسال و به‌طور کامل مصرف کنید.
    } finally {
      await lease?.release();
    }
    ```

    `acquireLocalService(...)` یک قرارداد پایدار و عمومی SDK برای سرویس ارائه‌دهنده
    است. میزبان پیکربندی فرایند را از
    `models.providers.<providerId>.localService` تعیین می‌کند؛ فراخوانندگان نمی‌توانند
    فرمان، آرگومان‌ها، محیط یا سیاست چرخهٔ عمر را تأمین کنند. ایجاد فرایند،
    آمادگی، عیب‌یابی و سیاست توقف هنگام بیکاری، داخلیِ میزبان باقی می‌مانند.

    شناسهٔ دقیق ارائه‌دهندهٔ پیکربندی‌شده و نشانی پایهٔ تعیین‌شدهٔ درخواست را ارسال کنید. نام‌های مستعار
    را با شناسهٔ آداپتور جایگزین نکنید: نام‌های مستعار جداگانه می‌توانند به میزبان‌های
    GPU محلی جداگانه اشاره کنند. میزبان نقاط پایانی‌ای را که با نشانی پایهٔ
    ارائه‌دهندهٔ پیکربندی‌شده مطابقت ندارند رد می‌کند، به‌جز نرمال‌سازی `/v1` که آداپتورهای Ollama و LM
    Studio استفاده می‌کنند. میزبان مالک سریال‌سازی راه‌اندازی، کاوش‌های آمادگی،
    اجاره‌های درخواست، مدیریت لغو و خاموش‌سازی هنگام بیکاری است.

    این تابع کمکی از همان مسیر آماده‌سازی تکمیل سادهٔ زمان‌اجرای
    داخلی OpenClaw و تصویر لحظه‌ای پیکربندی زمان‌اجرای تحت مالکیت میزبان استفاده می‌کند. موتورهای زمینه
    قابلیت `llm.complete` وابسته به نشست دریافت می‌کنند، بنابراین فراخوانی‌های مدل از عامل
    نشست فعال استفاده می‌کنند و بی‌سروصدا به عامل پیش‌فرض بازنمی‌گردند.
    نتیجه، انتساب ارائه‌دهنده/مدل/عامل را به‌همراه مصرف نرمال‌شدهٔ توکن،
    حافظهٔ نهان و هزینهٔ تخمینی، در صورت دسترس‌بودن، دربر می‌گیرد.

    `reasoning` را تنظیم کنید تا برای مدل انتخاب‌شده میزان تلاش استدلال درخواست شود.
    میزبان سطوح استاندارد تفکر (`off`، `minimal`، `low`،
    `medium`، `high`، `xhigh`، `adaptive`، `max` و `ultra`) را پیش از ارسال
    تکمیل، برای ارائه‌دهنده و مدل انتخاب‌شده نرمال می‌کند. `adaptive`
    به `medium` تبدیل می‌شود؛ `max` و `ultra` در صورت پشتیبانی به `max` و در غیر این صورت به `xhigh` تبدیل می‌شوند.

    <Warning>
    بازنویسی مدل نیازمند پذیرش صریح اپراتور از طریق `plugins.entries.<id>.llm.allowModelOverride: true` در پیکربندی است. برای محدودکردن Plugin‌های مورداعتماد به مقصدهای استاندارد مشخص `provider/model`، از `plugins.entries.<id>.llm.allowedModels` استفاده کنید. تکمیل‌های بین‌عاملی به `plugins.entries.<id>.llm.allowAgentIdOverride: true` نیاز دارند.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.gateway">
    روش دیگری از Gateway را درون فرایند فراخوانی کنید، درحالی‌که هویت زمان‌اجرای مورداعتماد
    Plugin فعلی حفظ می‌شود. این قابلیت برای Plugin‌های داخلی یا رسمی مورداعتماد در نظر گرفته شده است که قابلیت‌های
    Gateway تحت مالکیت Plugin را بدون بازکردن اتصال WebSocket بازگشتی ترکیب می‌کنند.

    ```typescript
    if (await api.runtime.gateway.isAvailable()) {
      const result = await api.runtime.gateway.request<{ callId: string }>(
        "voicecall.start",
        { to: "+15550001234", mode: "conversation" },
        { timeoutMs: 60_000 },
      );
    }
    ```

    درخواست‌ها از دامنهٔ `operator.write` استفاده می‌کنند و دامنهٔ مدیر را اعطا نمی‌کنند. فراخوانی‌های Plugin‌های خارجی
    دلخواه رد می‌شوند. متدهای ناموفق یک `GatewayClientRequestError` پرتاب می‌کنند و
    `details` ساخت‌یافته، فرادادهٔ تلاش مجدد و کد خطای Gateway را برای جریان‌های بازیابی حفظ می‌کنند. پیش از انتخاب این مسیر از ابزارهایی که می‌توانند در فرایندهای عامل مستقل نیز اجرا شوند، از `isAvailable()`
    استفاده کنید.

  </Accordion>
  <Accordion title="api.runtime.subagent">
    اجراهای عامل فرعی پس‌زمینه را راه‌اندازی و مدیریت کنید.

    ```typescript
    // اجرای عامل فرعی را آغاز کنید
    const { runId } = await api.runtime.subagent.run({
      sessionKey: "agent:main:subagent:search-helper",
      message: "این پرس‌وجو را به جست‌وجوهای تکمیلی متمرکز گسترش دهید.",
      toolsAlsoAllow: ["my_plugin_progress"],
      provider: "openai", // بازنویسی اختیاری
      model: "gpt-5.6-sol", // بازنویسی اختیاری
      deliver: false,
    });

    // منتظر تکمیل بمانید
    const result = await api.runtime.subagent.waitForRun({ runId, timeoutMs: 30000 });

    // پیام‌های نشست را بخوانید
    const { messages } = await api.runtime.subagent.getSessionMessages({
      sessionKey: "agent:main:subagent:search-helper",
      limit: 10,
    });

    // یک نشست را حذف کنید
    await api.runtime.subagent.deleteSession({
      sessionKey: "agent:main:subagent:search-helper",
    });
    ```

    <Warning>
    بازنویسی مدل (`provider`/`model`) نیازمند پذیرش صریح اپراتور از طریق `plugins.entries.<id>.subagent.allowModelOverride: true` در پیکربندی است. Plugin‌های نامطمئن همچنان می‌توانند عامل‌های فرعی را اجرا کنند، اما درخواست‌های بازنویسی رد می‌شوند.
    </Warning>

    `toolsAlsoAllow` ابزارهای دقیق و دارای مالکیت یکتا را که Plugin فراخواننده ثبت کرده است، به سطح ابزار عادی کارگر می‌افزاید. زمان‌اجرا ابزارهای هسته و نام‌های مشترک با Plugin دیگر را رد می‌کند. نمایه‌ها و سیاست‌های ابزار اپراتور، از جمله فهرست‌های مجاز و منع‌های صریح، همچنان اعمال می‌شوند.

    `deleteSession(...)` می‌تواند نشست‌هایی را که همان Plugin از طریق `api.runtime.subagent.run(...)` ایجاد کرده است حذف کند. حذف نشست‌های دلخواه کاربر یا اپراتور همچنان به درخواست Gateway با دامنهٔ مدیر نیاز دارد.

  </Accordion>
  <Accordion title="api.runtime.sandbox">
    اختیار مؤثر فضای کاری sandbox را برای یک نشست عامل بررسی کنید.

    ```typescript
    const authority = api.runtime.sandbox.resolveWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
    });

    const liveAuthority = await api.runtime.sandbox.prepareWorkspaceAuthority({
      config: cfg,
      agentId,
      sessionKey,
      workspaceDir,
      confinedToolNames: ["my_plugin_safe_tool"],
    });
    ```

    نتیجه گزارش می‌کند که آیا این نشست در sandbox قرار دارد، آیا فضای کاری آن
    دردسترس نیست، فقط‌خواندنی است یا نوشتنی، و در صورتی که سیاست مؤثر Docker، ابزار، نشست، مرورگر یا دسترسی ارتقایافته بتواند
    از آن فضای کاری خارج شود، یک `confinementError` اختیاری ارائه می‌دهد. از این قابلیت برای تصمیم‌های واگذاری تحت مالکیت میزبان استفاده کنید که
    نباید به کارگر اختیاری بیش از فراخوانندهٔ آن اعطا کنند. این یک تابع کمکی گواهی‌دهی
    است، نه جایگزینی برای بررسی مجوز خود فراخواننده.

    `prepareWorkspaceAuthority(...)` همان بررسی سیاست را انجام می‌دهد و همچنین
    sandbox مربوط به Docker را برای `workspaceDir` آماده می‌کند. این تابع کانتینر فعالی را
    که هش پیکربندی زندهٔ آن با mountها یا سیاست درخواستی مطابقت ندارد، رد می‌کند. فقط
    نام‌های دقیق ابزارهایی را ارسال کنید که Plugin فراخواننده پیاده‌سازی‌های ثبت‌شدهٔ آن‌ها را
    محدود می‌کند؛ پیشوندهای wildcard مالکیت ابزار را اثبات نمی‌کنند.

  </Accordion>
  <Accordion title="api.runtime.nodes">
    Nodeهای متصل را فهرست کنید و یک فرمان میزبان Node را از کد Plugin بارگذاری‌شده توسط Gateway یا از فرمان‌های CLI مربوط به Plugin فراخوانی کنید. هنگامی از این قابلیت استفاده کنید که یک Plugin مالک کار محلی روی دستگاهی جفت‌شده است، برای نمونه یک پل مرورگر یا صدا روی Mac دیگر.

    ```typescript
    const { nodes } = await api.runtime.nodes.list({ connected: true });

    const result = await api.runtime.nodes.invoke({
      nodeId: "mac-studio",
      command: "my-plugin.command",
      params: { action: "start" },
      timeoutMs: 30000,
    });
    ```

    `nodes.list(...)` شامل توصیف‌گرهای اعلام‌شدهٔ `nodePluginTools` برای هر Node متصل است، هنگامی که آن Node ابزارهای مبتنی بر Plugin یا MCP را در اختیار عامل قرار می‌دهد. این توصیف‌گرها وضعیت زندهٔ اتصال هستند: Gateway هنگام قطع اتصال Node آن‌ها را حذف می‌کند و یک Node می‌تواند پس از تغییر موجودی محلی Plugin/MCP آن‌ها را با `node.pluginTools.update` جایگزین کند.

    درون Gateway، این زمان اجرا درون‌فرایندی است. در فرمان‌های CLI مربوط به Plugin، این زمان اجرا Gateway پیکربندی‌شده را از طریق RPC فراخوانی می‌کند؛ بنابراین فرمان‌هایی مانند `openclaw googlemeet recover-tab` می‌توانند Nodeهای جفت‌شده را از ترمینال بررسی کنند. فرمان‌های Node همچنان از جفت‌سازی عادی Node در Gateway، فهرست‌های مجاز فرمان، سیاست‌های فراخوانی Node در Plugin و مدیریت محلی فرمان در Node عبور می‌کنند.

    Pluginهایی که ابزارهای عاملِ میزبانی‌شده روی Node ارائه می‌کنند، می‌توانند برای فرمان‌های غیرخطرناکی که باید به‌طور پیش‌فرض در فهرست مجاز قرار گیرند، `agentTool.defaultPlatforms` را تنظیم کنند. هنگامی که اپراتورها باید با `gateway.nodes.commands.allow` صریحاً آن را فعال کنند، این گزینه را حذف کنید. فرمان‌های خطرناک میزبان Node باید با `api.registerNodeInvokePolicy(...)` یک سیاست فراخوانی Node ثبت کنند؛ این سیاست پس از بررسی فهرست مجاز فرمان و پیش از ارسال فرمان به Node در Gateway اجرا می‌شود، بنابراین فراخوانی‌های مستقیم `node.invoke`، ابزارهای Plugin میزبانی‌شده روی Node و ابزارهای سطح‌بالاتر Plugin همگی از مسیر اجرایی یکسانی استفاده می‌کنند.

    <Warning>
    فیلد اختیاری `scopes` برای فراخوانی، حوزه‌های دسترسی اپراتور Gateway را درخواست می‌کند. OpenClaw آن را فقط برای Pluginهای همراه و نصب‌های مورد اعتماد Pluginهای رسمی اعمال می‌کند؛ درخواست‌های سایر Pluginها سطح دسترسی فراخوانی را افزایش نمی‌دهند. فقط زمانی از آن استفاده کنید که یک Plugin مورد اعتماد باید فرمانی از Node را با حوزهٔ دسترسی سخت‌گیرانه‌تری در Gateway، مانند `operator.admin`، فراخوانی کند.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.tasks">
    وضعیت جریان وظیفه و اجرای وظیفه را به یک کلید نشست موجود OpenClaw یا زمینهٔ ابزار مورد اعتماد متصل کنید.

    - `api.runtime.tasks.managedFlows` قابلیت تغییر دارد: جریان‌های وظیفه را ایجاد، پیش‌برد و لغو می‌کند.
    - `api.runtime.tasks.flows` و `api.runtime.tasks.runs` نماهای DTO فقط‌خواندنی برای فهرست‌کردن و جست‌وجوی وضعیت هستند؛ هر دو `bindSession(...)` / `fromToolContext(...)` به‌همراه `get`، `list`، `findLatest` و `resolve` را ارائه می‌کنند.

    جریان وظیفه وضعیت پایدار گردش‌کار چندمرحله‌ای را پیگیری می‌کند. این یک زمان‌بند نیست:
    برای بیدارسازی‌های آینده از Cron یا `api.session.workflow.scheduleSessionTurn(...)` استفاده کنید،
    سپس هنگامی که آن کار به وضعیت جریان، وظایف فرزند، انتظار یا لغو نیاز دارد،
    از `managedFlows` در نوبت زمان‌بندی‌شده استفاده کنید.

    ```typescript
    const taskFlow = api.runtime.tasks.managedFlows.fromToolContext(ctx);

    const created = taskFlow.createManaged({
      controllerId: "my-plugin/review-batch",
      goal: "بازبینی Pull requestهای جدید",
    });

    const child = taskFlow.runTask({
      flowId: created.flowId,
      runtime: "acp",
      childSessionKey: "agent:main:subagent:reviewer",
      task: "بازبینی PR شمارهٔ 123",
      status: "running",
      startedAt: Date.now(),
    });

    const waiting = taskFlow.setWaiting({
      flowId: created.flowId,
      expectedRevision: created.revision,
      currentStep: "await-human-reply",
      waitJson: { kind: "reply", channel: "telegram" },
    });
    ```

    هنگامی که از لایهٔ اتصال خود یک کلید نشست مورد اعتماد OpenClaw دارید، از `bindSession({ sessionKey, requesterOrigin })` استفاده کنید. اتصال را از ورودی خام کاربر انجام ندهید.

  </Accordion>
  <Accordion title="api.runtime.tts">
    تبدیل متن به گفتار.

    ```typescript
    // TTS استاندارد
    const clip = await api.runtime.tts.textToSpeech({
      text: "سلام از OpenClaw",
      cfg: api.config,
    });

    // TTS بهینه‌شده برای تلفن
    const telephonyClip = await api.runtime.tts.textToSpeechTelephony({
      text: "سلام از OpenClaw",
      cfg: api.config,
    });

    // فهرست‌کردن صداهای موجود
    const voices = await api.runtime.tts.listVoices({
      provider: "elevenlabs",
      cfg: api.config,
    });
    ```

    از پیکربندی اصلی `tts` و انتخاب ارائه‌دهنده استفاده می‌کند. بافر صوتی PCM به‌همراه نرخ نمونه‌برداری را برمی‌گرداند. `textToSpeechStream` نیز برای تبدیل جریانی در دسترس است.

  </Accordion>
  <Accordion title="api.runtime.mediaUnderstanding">
    تحلیل تصویر، صدا و ویدئو.

    ```typescript
    // توصیف یک تصویر
    const image = await api.runtime.mediaUnderstanding.describeImageFile({
      filePath: "/tmp/inbound-photo.jpg",
      cfg: api.config,
      agentDir: "/tmp/agent",
    });

    // رونویسی صدا
    const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
      filePath: "/tmp/inbound-audio.ogg",
      cfg: api.config,
      mime: "audio/ogg", // اختیاری، برای زمانی که MIME قابل تشخیص نیست
    });

    // توصیف یک ویدئو
    const video = await api.runtime.mediaUnderstanding.describeVideoFile({
      filePath: "/tmp/inbound-video.mp4",
      cfg: api.config,
    });

    // تحلیل عمومی فایل
    const result = await api.runtime.mediaUnderstanding.runFile({
      filePath: "/tmp/inbound-file.pdf",
      cfg: api.config,
    });

    // استخراج ساخت‌یافتهٔ تصویر از طریق یک ارائه‌دهنده/مدل مشخص.
    // دست‌کم یک تصویر وارد کنید؛ ورودی‌های متنی زمینهٔ تکمیلی هستند.
    const evidence = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
      provider: "codex",
      model: "gpt-5.6-sol",
      input: [
        {
          type: "image",
          buffer: receiptImageBuffer,
          fileName: "receipt.png",
          mime: "image/png",
        },
        { type: "text", text: "مبلغ کل چاپ‌شده را بر یادداشت‌های دست‌نویس ترجیح بده." },
      ],
      instructions: "فروشنده، مبلغ کل و برچسب‌های قابل جست‌وجو را استخراج کن.",
      schemaName: "receipt.evidence",
      jsonSchema: {
        type: "object",
        properties: {
          vendor: { type: "string" },
          total: { type: "number" },
          tags: { type: "array", items: { type: "string" } },
        },
        required: ["vendor", "total"],
      },
      cfg: api.config,
    });
    ```

    هنگامی که هیچ خروجی تولید نشود (برای نمونه، ورودی نادیده گرفته شده باشد)، `{ text: undefined }` را برمی‌گرداند.

    `describeImageFileWithModel(...)` یک تصویر ازپیش‌شناخته‌شده را از طریق ارائه‌دهنده/مدلی مشخص توصیف می‌کند و از تفکیک پیش‌فرض مدل فعال که `describeImageFile(...)` استفاده می‌کند، عبور نمی‌کند.

  </Accordion>
  <Accordion title="api.runtime.imageGeneration">
    تولید تصویر.

    ```typescript
    const result = await api.runtime.imageGeneration.generate({
      prompt: "رباتی در حال نقاشی یک غروب",
      cfg: api.config,
    });

    const providers = api.runtime.imageGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.videoGeneration">
    تولید ویدئو، با ساختاری مشابه تولید تصویر.

    ```typescript
    const result = await api.runtime.videoGeneration.generate({
      prompt: "نمای پهپادی در حال پرواز بر فراز خط ساحلی هنگام طلوع خورشید",
      cfg: api.config,
    });

    const providers = api.runtime.videoGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.musicGeneration">
    تولید موسیقی، با ساختاری مشابه تولید تصویر.

    ```typescript
    const result = await api.runtime.musicGeneration.generate({
      prompt: "یک قطعهٔ شاد لو-فای برای جلسهٔ کدنویسی",
      cfg: api.config,
    });

    const providers = api.runtime.musicGeneration.listProviders({ cfg: api.config });
    ```

  </Accordion>
  <Accordion title="api.runtime.webSearch">
    جست‌وجوی وب.

    ```typescript
    const providers = api.runtime.webSearch.listProviders({ config: api.config });

    const result = await api.runtime.webSearch.search({
      config: api.config,
      args: { query: "SDK مربوط به Plugin در OpenClaw", count: 5 },
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.media">
    ابزارهای سطح‌پایین رسانه.

    ```typescript
    const webMedia = await api.runtime.media.loadWebMedia(url);
    const mime = await api.runtime.media.detectMime(buffer);
    const kind = api.runtime.media.mediaKindFromMime("image/jpeg"); // "image"
    const isVoice = api.runtime.media.isVoiceCompatibleAudio(filePath);
    const metadata = await api.runtime.media.getImageMetadata(filePath);
    const resized = await api.runtime.media.resizeToJpeg(buffer, { maxWidth: 800 });
    const terminalQr = await api.runtime.media.renderQrTerminal("https://openclaw.ai");
    const pngQr = await api.runtime.media.renderQrPngBase64("https://openclaw.ai", {
      scale: 6, // 1-12
      marginModules: 4, // 0-16
    });
    const pngQrDataUrl = await api.runtime.media.renderQrPngDataUrl("https://openclaw.ai");
    const tmpRoot = resolvePreferredOpenClawTmpDir();
    const pngQrFile = await api.runtime.media.writeQrPngTempFile("https://openclaw.ai", {
      tmpRoot,
      dirPrefix: "my-plugin-qr-",
      fileName: "qr.png",
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.config">
    نمای لحظه‌ای پیکربندی زمان اجرا و نوشتن تراکنشی پیکربندی. پیکربندی‌ای را ترجیح دهید
    که از قبل به مسیر فراخوانی فعال ارسال شده است؛ فقط زمانی از
    `current()` استفاده کنید که مدیریت‌کننده مستقیماً به نمای لحظه‌ای فرایند نیاز دارد.

    ```typescript
    const cfg = api.runtime.config.current();
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    `mutateConfigFile(...)` و `replaceConfigFile(...)` یک مقدار `followUp`
    را برمی‌گردانند، برای نمونه `{ mode: "restart", requiresRestart: true, reason }`،
    که قصد نویسنده را بدون گرفتن کنترل راه‌اندازی مجدد از
    Gateway ثبت می‌کند.

  </Accordion>
  <Accordion title="api.runtime.system">
    ابزارهای سطح سیستم.

    ```typescript
    await api.runtime.system.enqueueSystemEvent(event);
    api.runtime.system.requestHeartbeat({
      source: "other",
      intent: "event",
      reason: "plugin-event",
    });
    api.runtime.system.requestHeartbeatNow({ reason: "plugin-event" }); // نام مستعار سازگاری منسوخ‌شده.
    const heartbeatResult = await api.runtime.system.runHeartbeatOnce({
      reason: "plugin-triggered-check",
    });
    const output = await api.runtime.system.runCommandWithTimeout(cmd, args, opts);
    const hint = api.runtime.system.formatNativeDependencyHint(pkg);
    ```

    `runHeartbeatOnce(...)` یک چرخهٔ Heartbeat را بلافاصله و با عبور از زمان‌سنج عادی ادغام اجرا می‌کند. برای اجبار ارسال به آخرین کانال فعال به‌جای سرکوب پیش‌فرض `target: "none"`، مقدار `{ heartbeat: { target: "last" } }` را ارسال کنید.

    `runCommandWithTimeout(...)` مقادیر ضبط‌شدهٔ `stdout` و `stderr`، تعدادهای اختیاری
    کوتاه‌سازی، `code`، `signal`، `killed`، `termination` و
    `noOutputTimedOut` را برمی‌گرداند. نتایج مهلت زمانی و مهلت زمانیِ بدون خروجی، هنگامی که فرایند فرزند کد خروج غیرصفر ارائه نمی‌کند، `code: 124`
    را گزارش می‌دهند. خروج با سیگنال که ناشی از مهلت زمانی نیست
    همچنان می‌تواند `code: null` را برگرداند؛ بنابراین برای تشخیص دلایل مهلت زمانی از `termination` و
    `noOutputTimedOut` استفاده کنید.

  </Accordion>
  <Accordion title="api.runtime.events">
    اشتراک رویدادها.

    ```typescript
    api.runtime.events.onAgentEvent((event) => {
      /* ... */
    });
    api.runtime.events.onSessionTranscriptUpdate((update) => {
      /* ... */
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.logging">
    ثبت گزارش.

    ```typescript
    const verbose = api.runtime.logging.shouldLogVerbose();
    const childLogger = api.runtime.logging.getChildLogger({ plugin: "my-plugin" }, { level: "debug" });
    ```

  </Accordion>
  <Accordion title="api.runtime.modelAuth">
    تفکیک احراز هویت مدل و ارائه‌دهنده.

    ```typescript
    const auth = await api.runtime.modelAuth.getApiKeyForModel({ model, cfg });

    // احراز هویت آمادهٔ درخواست، شامل تبادل‌های زمان اجرای ارائه‌دهنده (برای مثال، نوسازی OAuth)
    const runtimeAuth = await api.runtime.modelAuth.getRuntimeAuthForModel({ model, cfg });

    const providerAuth = await api.runtime.modelAuth.resolveApiKeyForProvider({
      provider: "openai",
      cfg,
    });
    ```

  </Accordion>
  <Accordion title="api.runtime.state">
    تفکیک پوشهٔ وضعیت و ذخیره‌سازی کلیددار مبتنی بر SQLite.

    ```typescript
    const stateDir = api.runtime.state.resolveStateDir(process.env);
    const store = api.runtime.state.openKeyedStore<MyRecord>({
      namespace: "my-feature",
      maxEntries: 200,
      defaultTtlMs: 15 * 60_000,
    });

    await store.register("key-1", { value: "hello" });
    const claimed = await store.registerIfAbsent("dedupe-key", { value: "first" });
    const value = await store.lookup("key-1");
    await store.deleteIf?.("key-1", (current) => current.value === "hello");
    await store.consume("key-1");
    await store.clear();

    const blobs = api.runtime.state.openBlobStore<MyBlobMetadata>({
      namespace: "rendered-artifacts",
      maxEntries: 100,
      maxBytesPerEntry: 4 * 1024 * 1024,
      maxBytesPerNamespace: 64 * 1024 * 1024,
      defaultTtlMs: 15 * 60_000,
    });
    await blobs.register(
      "artifact-1",
      new TextEncoder().encode("binary or text payload"),
      { contentType: "text/plain" },
    );
    const blob = await blobs.lookup("artifact-1");

    await api.runtime.state.withLease(
      {
        namespace: "my-feature",
        key: "writer",
        database: { scope: "agent", agentId },
        leaseMs: 5 * 60_000,
        waitMs: 30_000,
      },
      async ({ signal, assertOwned }) => {
        await runExternalWriter({ signal });
        assertOwned();
      },
    );
    ```

    ذخیره‌گاه‌های کلیددار پس از راه‌اندازی مجدد باقی می‌مانند و بر اساس شناسهٔ Plugin مقید به زمان اجرا از یکدیگر جدا می‌شوند. برای ادعاهای اتمی حذف تکرار از `registerIfAbsent(...)` استفاده کنید: اگر کلید وجود نداشته یا منقضی شده و ثبت شود، `true` را برمی‌گرداند؛ یا اگر از قبل مقداری فعال وجود داشته باشد، بدون بازنویسی مقدار، زمان ایجاد یا TTL آن، `false` را برمی‌گرداند. هنگامی که پاک‌سازی باید فقط مقدار مشاهده‌شدهٔ قبلی را حذف کند، از `deleteIf(...)` استفاده کنید؛ گزارهٔ همگام و حذف آن در یک تراکنش SQLite اجرا می‌شوند. محدودیت‌ها: `maxEntries` برای هر فضای نام، 50,000 ردیف فعال برای هر Plugin، مقادیر JSON کمتر از 64KB و انقضای اختیاری TTL. به‌طور پیش‌فرض، نوشتن در هر یک از محدودیت‌های ردیف، قدیمی‌ترین ردیف‌های فعال را از فضای نامی که در آن نوشته می‌شود حذف می‌کند؛ فضاهای نام هم‌سطح برای آن نوشتن تخلیه نمی‌شوند و اگر فضای نام نتواند به‌اندازهٔ کافی ردیف آزاد کند، نوشتن همچنان شکست می‌خورد. برای رکوردهای مالکیت پایدار که هرگز نباید تخلیه شوند، `overflowPolicy: "reject-new"` را تنظیم کنید: کلیدهای جدید در هر یک از محدودیت‌ها شکست می‌خورند، درحالی‌که کلیدهای موجود همچنان قابل به‌روزرسانی می‌مانند.

    `openSyncKeyedStore<T>(...)` همان ساختار ذخیره‌گاه را با متدهای همگام برمی‌گرداند (`register`، `registerIfAbsent`، `deleteIf`، `lookup`، `consume` و `clear` همگی به‌جای promise، مقادیر را مستقیماً برمی‌گردانند) تا فراخوان‌هایی که نمی‌توانند منتظر بمانند از آن استفاده کنند.

    `openBlobStore<TMetadata>(...)` بارهای دودویی محدودشده را بدون base64 یا فایل‌های جانبی در SQLite مشترک ذخیره می‌کند. این مورد به محدودیت‌های بایت برای هر ورودی و هر فضای نام، و نیز محدودیت ردیف نیاز دارد؛ آرایه‌های بایت را در مرز API کپی می‌کند؛ و فراداده‌ها را بدون بارگیری هر BLOB فهرست می‌کند. `register(...)` یک upsert صریح است، از جمله برای کلیدهای منقضی‌شده. `registerIfAbsent(...)` ایجاد ایمن در برابر برخورد را فراهم می‌کند: یک کلید منقضی‌شده تا زمانی که مالک آن با `deleteExpiredKey(key)` یا `deleteExpired()` ادعایش کند، اشغال‌شده باقی می‌ماند و فرادادهٔ لازم برای حذف مصنوعات نام‌گذاری‌شدهٔ مرتبط پس از commit در SQLite را حفظ می‌کند. هر ردیفی که TTL داشته باشد موقتی است و حتی پیش از انقضا از پشتیبان‌گیری/بازیابی کنار گذاشته می‌شود؛ برای وضعیت پایدار و قابل‌بازیابی، TTL را حذف کنید. فیوزهای میزبان هر BLOB را به 100 MiB، هر Plugin را به 512 MiB از BLOBهای ذخیره‌شدهٔ فیزیکی، و هر Plugin را به 50,000 ردیف ذخیره‌شدهٔ فیزیکی محدود می‌کنند؛ این تعداد شامل ردیف‌های منقضی‌شده‌ای نیز می‌شود که در انتظار پاک‌سازی توسط مالک هستند. هنگامی که جایگزینی یا تخلیه نباید تجسم‌های خارجی را بی‌سروصدا بدون مالک باقی بگذارد، از `registerIfAbsent(...)` همراه با `overflowPolicy: "reject-new"` استفاده کنید.

    `openChannelIngressQueue<TPayload>(...)` یک صف ورودی پایدار با دامنهٔ محدود به Plugin فراخوان باز می‌کند تا رویدادهای ورودی نیازمند پردازش حداقل یک‌باره در راه‌اندازی‌های مجدد را بافر کند. هنگامی که بازیابی ادعای کهنه از `shouldRecover` استفاده می‌کند، اگر بارهای ادعاشدهٔ خراب باید قرنطینه شوند، `shouldRecoverCorrupt` را نیز ارائه کنید: هویت ادعای مستقل از بار آن به Plugin امکان می‌دهد پیش از آن‌که صف ردیف را به سنگ‌قبر تبدیل کند، سیاست فعال مالک و مسیر را حفظ کند.

    `withLease(...)` کار مشارکتی Plugin را میان فرایندهای OpenClaw سریال‌سازی می‌کند. برای یک مالک سراسری `database: { scope: "shared" }` یا برای مالکیت مستقل هر عامل `{ scope: "agent", agentId }` را انتخاب کنید. `AbortSignal` فراخوان بازگشتی را به هر عملیات شکست‌پذیر ارسال کنید. `assertOwned()` پیش از آغاز یک گام مهم دیگر، یک نقطهٔ وارسی لحظه‌ای است؛ میزبان نیز پس از فراخوان بازگشتی مالکیت را تأیید می‌کند. از دست رفتن اجاره یا لغو از سوی فراخوان، سیگنال را لغو می‌کند. انتظار برای اکتساب و Heartbeatها خارج از تراکنش‌های کوتاه و همگام SQLite رخ می‌دهند؛ Pluginها هرگز مسیرها یا دستگیره‌های پایگاه‌داده را دریافت نمی‌کنند. این لغو مشارکتی است، نه توکن حصارگذاری یا مجوزی برای نوشتن خارجی بدون حصار.

    `openChannelIngressDrain(...)` کارگر اصلی مستقل از کانال را روی آن صف باز می‌کند (یا وقتی صفی ارائه نشده باشد، یک صف می‌سازد). تخلیه، مالک بازیابی ادعاهای کهنه، سریال‌سازی ادعا برای هر مسیر، تکمیل هنگام پذیرش یا تکمیل هنگام بازگشت ارسال، تعیین وضعیت تلاش مجدد/نامهٔ مرده، جایگزینی اختیاری پیش از پذیرش، و مهلت توقف ادعا←پذیرش است. مالکیت ادعا را با `turnAdoptionLifecycle` (از طریق `bindIngressLifecycleToReplyOptions` از `plugin-sdk/channel-outbound`) به تولید پاسخ متصل کنید. Pluginهای کانال، صف‌گذاری سمت پذیرش، استخراج مسیر، طبقه‌بندی غیرقابل‌تلاش‌مجدد و هرگونه سیاست مجوز جایگزینی را نگه می‌دارند.

    <Warning>
    `openBlobStore`، `openKeyedStore`، `openSyncKeyedStore`، `withLease`، `openChannelIngressQueue` و `openChannelIngressDrain` در این نسخه فقط برای Pluginهای همراه و نصب‌های رسمی و مورداعتماد Plugin در دسترس هستند.
    </Warning>

  </Accordion>
  <Accordion title="api.runtime.channel">
    ابزارهای کمکی زمان اجرای مختص کانال (هنگامی که یک Plugin کانال بارگذاری شده باشد در دسترس‌اند). گروه‌بندی بر اساس موضوع:

    | گروه | هدف |
    | --- | --- |
    | `text` | قطعه‌بندی (`chunkText`، `chunkMarkdownText`، `resolveChunkMode`)؛ تشخیص فرمان کنترلی؛ تبدیل جدول Markdown. |
    | `reply` | ارسال پاسخ بلوکی بافری، قالب‌بندی پاکت، تفکیک پیکربندی مؤثر پیام‌ها/تأخیر انسانی. |
    | `routing` | `buildAgentSessionKey`، `resolveAgentRoute`. |
    | `pairing` | `buildPairingReply`، خواندن/حذف فهرست مجاز، upsert درخواست‌های جفت‌سازی و ورودی‌های تأیید استخراج‌شده از درخواست. |
    | `media` | بارگیری/ذخیرهٔ رسانهٔ راه‌دور (پایین را ببینید). |
    | `activity` | ثبت/خواندن آخرین فعالیت کانال. |
    | `session` | فرادادهٔ نشست از رویدادهای ورودی، به‌روزرسانی‌های آخرین مسیر. |
    | `mentions` | ابزارهای کمکی سیاست اشاره (پایین را ببینید). |
    | `reactions` | دستگیره‌های واکنش تأیید برای نشانگرهای پردازش در حال انجام. |
    | `groups` | تفکیک سیاست گروه و الزام اشاره. |
    | `debounce` | حذف نوسان پیام ورودی. |
    | `commands` | مجوزدهی فرمان و دروازه‌بانی فرمان متنی. |
    | `outbound` | بارگذاری آداپتور خروجی یک کانال. |
    | `inbound` | ساخت زمینهٔ رویداد ورودی و اجرای هستهٔ مشترک رویداد ورودی/پاسخ. |
    | `threadBindings` | تنظیم مهلت بی‌کاری/حداکثر سن برای رشته‌های نشست مقید. |
    | `runtimeContexts` | ثبت، خواندن و پایش زمینهٔ محلی فرایند برای هر کانال/حساب/قابلیت. |

    `api.runtime.channel.media` سطح ترجیحی برای بارگیری و ذخیره‌سازی رسانهٔ کانال است:

    ```typescript
    const saved = await api.runtime.channel.media.saveRemoteMedia({
      url,
      subdir: "inbound",
      maxBytes,
      filePathHint: fileName,
    });
    ```

    هنگامی که یک URL راه‌دور باید به رسانهٔ OpenClaw تبدیل شود، از `saveRemoteMedia(...)` استفاده کنید. هنگامی که Plugin از قبل یک `Response` را با احراز هویت، تغییر مسیر یا مدیریت فهرست مجاز تحت مالکیت Plugin دریافت کرده است، از `saveResponseMedia(...)` استفاده کنید. فقط هنگامی از `readRemoteMediaBuffer(...)` استفاده کنید که Plugin برای بازرسی، تبدیل، رمزگشایی یا بارگذاری مجدد به بایت‌های خام نیاز دارد. `fetchRemoteMedia(...)` همچنان یک نام مستعار سازگاری منسوخ برای `readRemoteMediaBuffer(...)` است.

    `api.runtime.channel.mentions` سطح مشترک سیاست اشارهٔ ورودی برای Pluginهای کانال همراهی است که از تزریق زمان اجرا استفاده می‌کنند:

    ```typescript
    const mentionMatch = api.runtime.channel.mentions.matchesMentionWithExplicit(text, {
      mentionRegexes,
      mentionPatterns,
    });

    const decision = api.runtime.channel.mentions.resolveInboundMentionDecision({
      facts: {
        canDetectMention: true,
        wasMentioned: mentionMatch.matched,
        implicitMentionKinds: api.runtime.channel.mentions.implicitMentionKindWhen(
          "reply_to_bot",
          isReplyToBot,
        ),
      },
      policy: {
        isGroup,
        requireMention,
        allowTextCommands,
        hasControlCommand,
        commandAuthorized,
      },
    });
    ```

    ابزارهای کمکی اشارهٔ موجود:

    - `buildMentionRegexes`
    - `matchesMentionPatterns`
    - `matchesMentionWithExplicit`
    - `implicitMentionKindWhen`
    - `resolveInboundMentionDecision`

    برای تصمیم‌های اشاره از مسیر نرمال‌شدهٔ `{ facts, policy }` استفاده کنید.

    چندین فیلد در `reply`، `session` و `inbound` دارای یادداشت‌های `@deprecated` برای هر فیلد هستند که به هستهٔ کنونی نوبت کانال یا آداپتورهای خروجی کانال اشاره می‌کنند؛ پیش از ساخت کد جدید بر مبنای آن، JSDoc درون‌خطی ابزار کمکی مشخص را بررسی کنید.

  </Accordion>
</AccordionGroup>

## ذخیره‌سازی ارجاع‌های زمان اجرا

برای ذخیرهٔ ارجاع زمان اجرا جهت استفاده خارج از فراخوان بازگشتی `register`، از `createPluginRuntimeStore` استفاده کنید:

<Steps>
  <Step title="ایجاد ذخیره‌گاه">
    ```typescript
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

    const store = createPluginRuntimeStore<PluginRuntime>({
      pluginId: "my-plugin",
      errorMessage: "my-plugin runtime not initialized",
    });
    ```

  </Step>
  <Step title="اتصال به نقطهٔ ورود">
    ```typescript
    export default defineChannelPluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Example",
      plugin: myPlugin,
      setRuntime: store.setRuntime,
    });
    ```
  </Step>
  <Step title="دسترسی از فایل‌های دیگر">
    ```typescript
    export function getRuntime() {
      return store.getRuntime(); // اگر مقداردهی اولیه نشده باشد، خطا می‌اندازد
    }

    export function tryGetRuntime() {
      return store.tryGetRuntime(); // اگر مقداردهی اولیه نشده باشد، null برمی‌گرداند
    }
    ```

  </Step>
</Steps>

<Note>
برای هویت ذخیره‌گاه زمان اجرا، `pluginId` را ترجیح دهید. فرم سطح پایین‌تر `key` برای موارد نامتداولی است که یک Plugin عمداً به بیش از یک جایگاه زمان اجرا نیاز دارد.
</Note>

## سایر فیلدهای سطح بالای `api`

فراتر از `api.runtime`، شیء API موارد زیر را نیز فراهم می‌کند:

<ParamField path="api.id" type="string">
  شناسه Plugin.
</ParamField>
<ParamField path="api.name" type="string">
  نام نمایشی Plugin.
</ParamField>
<ParamField path="api.config" type="OpenClawConfig">
  تصویر لحظه‌ای پیکربندی فعلی (در صورت وجود، تصویر لحظه‌ای فعال زمان اجرا در حافظه).
</ParamField>
<ParamField path="api.pluginConfig" type="Record<string, unknown>">
  پیکربندی مختص Plugin از `plugins.entries.<id>.config`.
</ParamField>
<ParamField path="api.logger" type="PluginLogger">
  ثبت‌کننده گزارش با دامنه محدود (`debug`، `info`، `warn`، `error`).
</ParamField>
<ParamField path="api.registrationMode" type="PluginRegistrationMode">
  حالت بارگذاری فعلی: `"full"` (فعال‌سازی زنده)، `"discovery"` / `"tool-discovery"` (کشف قابلیت فقط‌خواندنی)، `"setup-only"` (ورودی راه‌اندازی سبک‌وزن)، `"setup-runtime"` (جریان راه‌اندازی که به ورودی کانال زمان اجرا نیز نیاز دارد)، یا `"cli-metadata"` (گردآوری فراداده فرمان CLI).
</ParamField>
<ParamField path="api.resolvePath(input)" type="(string) => string">
  یک مسیر را نسبت به ریشه Plugin تفکیک کنید.
</ParamField>

## مرتبط

- [جزئیات داخلی Plugin](/fa/plugins/architecture) — مدل قابلیت و رجیستری
- [نقاط ورود SDK](/fa/plugins/sdk-entrypoints) — گزینه‌های `definePluginEntry`
- [نمای کلی SDK](/fa/plugins/sdk-overview) — مرجع زیرمسیر
