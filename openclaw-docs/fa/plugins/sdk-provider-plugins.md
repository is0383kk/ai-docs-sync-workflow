---
read_when:
    - در حال ساخت یک Plugin جدید برای ارائه‌دهنده مدل هستید
    - می‌خواهید یک پروکسی سازگار با OpenAI یا یک LLM سفارشی به OpenClaw اضافه کنید
    - باید احراز هویت ارائه‌دهنده، کاتالوگ‌ها و هوک‌های زمان اجرا را درک کنید
sidebarTitle: Provider plugins
summary: راهنمای گام‌به‌گام ساخت Plugin ارائه‌دهنده مدل برای OpenClaw
title: ساخت Pluginهای ارائه‌دهنده
x-i18n:
    generated_at: "2026-07-27T15:46:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

یک Plugin ارائه‌دهنده بسازید تا یک ارائه‌دهنده مدل (LLM) به OpenClaw اضافه شود: کاتالوگ مدل،
احراز هویت با کلید API و تفکیک پویای مدل.

<Info>
  با Pluginهای OpenClaw تازه آشنا شده‌اید؟ ابتدا [شروع به کار](/fa/plugins/building-plugins)
  را برای ساختار بسته و راه‌اندازی مانیفست بخوانید.
</Info>

<Tip>
  Pluginهای ارائه‌دهنده، مدل‌ها را به چرخه استنتاج عادی OpenClaw اضافه می‌کنند. اگر
  مدل باید از طریق یک دیمن عامل بومی اجرا شود که مالک رشته‌ها، Compaction
  یا رویدادهای ابزار است، به‌جای قراردادن جزئیات پروتکل دیمن در هسته،
  ارائه‌دهنده را با یک [مهار عامل](/fa/plugins/sdk-agent-harness) همراه کنید.
</Tip>

## راهنمای گام‌به‌گام

<Steps>
  <Step title="بسته و مانیفست">
    ### گام 1: بسته و مانیفست

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "ارائه‌دهنده مدل Acme AI",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "کلید API ‏Acme AI",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "کلید API ‏Acme AI"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars` به OpenClaw اجازه می‌دهد اعتبارنامه‌ها را بدون
    بارگذاری زمان‌اجرای Plugin شما تشخیص دهد. وقتی یک گونه ارائه‌دهنده
    باید از احراز هویت شناسه ارائه‌دهنده دیگری دوباره استفاده کند، `providerAuthAliases` را اضافه کنید. `modelSupport`
    اختیاری است و به OpenClaw اجازه می‌دهد پیش از وجود قلاب‌های زمان اجرا، Plugin ارائه‌دهنده شما را از روی
    شناسه‌های کوتاه مدل مانند `acme-large` به‌طور خودکار بارگذاری کند. `openclaw.compat`
    و `openclaw.build` در `package.json` برای انتشار در ClawHub
    الزامی هستند (`openclaw.compat.pluginApi` و `openclaw.build.openclawVersion`
    دو فیلد الزامی‌اند؛ اگر `minGatewayVersion` حذف شود، مقدار آن از
    `openclaw.install.minHostVersion` گرفته می‌شود).

  </Step>

  <Step title="ثبت ارائه‌دهنده">
    یک ارائه‌دهنده متنی حداقلی به `id`، `label`، `auth` و `catalog` نیاز دارد.
    `catalog` قلاب زمان اجرا/پیکربندی تحت مالکیت ارائه‌دهنده است؛ این قلاب می‌تواند APIهای زنده
    فروشنده را فراخوانی کند و ورودی‌های `models.providers` را برگرداند.

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider` سطح جدیدتر کاتالوگ صفحه کنترل
    برای رابط کاربری فهرست/راهنما/انتخاب‌گر است که ردیف‌های `text`، `voice`، `image_generation`،
    `video_generation` و `music_generation` را پوشش می‌دهد. فراخوانی‌های نقطه پایانی
    فروشنده و نگاشت پاسخ را در Plugin نگه دارید؛ OpenClaw مالک شکل مشترک ردیف‌ها،
    برچسب‌های منبع و رندر راهنما است.

    اکنون یک ارائه‌دهنده عملیاتی دارید. کاربران می‌توانند
    `openclaw onboard --acme-ai-api-key <key>` را اجرا کنند و
    `acme-ai/acme-large` را به‌عنوان مدل خود انتخاب کنند.

    ### کشف زنده مدل

    اگر ارائه‌دهنده شما یک API سازگار با OpenAI برای `/models` ارائه می‌کند،
    دستیار تک‌ارائه‌دهنده را برای کشف مشترک فعال کنید:

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true` یک قرارداد عمومی Plugin SDK با رفتارهای زیر است:

    | حوزه | قرارداد |
    | --- | --- |
    | اعتبارنامه‌ها | کشف از اعتبارنامه تفکیک‌شده ارائه‌دهنده در کاتالوگ استفاده می‌کند و وقتی احراز هویت مقداری فراهم کند، `discoveryApiKey` را ترجیح می‌دهد. نشانگرهای ارجاع محرمانه هرگز به‌عنوان توکن ارسال نمی‌شوند. درخواست پیش‌فرض از `Authorization: Bearer <token>` استفاده می‌کند؛ برای طرح احراز هویت دیگری از فروشنده، از `buildRequestHeaders` استفاده کنید. |
    | نقطه پایانی | نشانی اینترنتی پیش‌فرض، `models` نسبت به `baseUrl` مؤثر ارائه‌دهنده است و وقتی `allowExplicitBaseUrl` فعال باشد، بازنویسی اپراتور را نیز در بر می‌گیرد. برای مسیر نسبی دیگری از `endpointPath` استفاده کنید. فقط برای یک نشانی اینترنتی ثابت فروشنده از `endpointUrl: { url, requireBaseUrl }` استفاده کنید؛ مگر آنکه URL پایه مؤثر همچنان با `requireBaseUrl` برابر باشد، کشف انجام نمی‌شود تا اعتبارنامه پراکسی سفارشی برای فروشنده ارسال نشود. |
    | محدودیت‌های شبکه | واکشی‌ها از محافظ SSRF ‏OpenClaw، یک بودجه مهلت 5 ثانیه‌ای برای کل صفحه‌بندی، محدودیت پاسخ 4 MiB برای هر صفحه و محدودیت 50 صفحه استفاده می‌کنند. پیوندهای صفحه‌بندی با مبدأ متفاوت رد می‌شوند؛ اعتبارنامه‌ها پس از تغییر مسیر به مبدأ دیگر حذف می‌شوند. |
    | حافظه نهان | کاتالوگ‌های موفق و غیرخالی بر اساس ارائه‌دهنده، نقطه پایانی و اعتبارنامه تفکیک‌شده به‌مدت 60 ثانیه در حافظه نهان ذخیره می‌شوند. نتایج خالی یا غیرقابل‌استفاده در حافظه نهان ذخیره نمی‌شوند. |
    | پالایش | شناسه‌های زنده منطبق دقیق، فراداده ایستای مورداعتماد خود را حفظ می‌کنند. ردیف‌های جدید با رویکردی محافظه‌کارانه به‌عنوان مدل‌های متن/گفت‌وگو بازنمایی می‌شوند. ردیف‌های غیرفعال، بایگانی‌شده، منسوخ، صراحتاً غیرگفت‌وگویی، تعبیه‌سازی، رتبه‌بندی مجدد، نظارت محتوا، گفتار، فقط‌تصویر و فقط‌ویدئو کنار گذاشته می‌شوند. فقط برای انتخاب ردیف‌ها از یک پوش پاسخ غیراستاندارد از `readRows` استفاده کنید؛ معناشناسی مدل مختص ارائه‌دهنده همچنان باید در یک کاتالوگ سفارشی قرار گیرد. |
    | شکست | کشف زنده جنبه راهنما دارد. شکست‌های احراز هویت، شبکه، مهلت زمانی، صفحه‌بندی، تجزیه، کاتالوگ خالی و پالایش، به‌جای حذف ارائه‌دهنده، بذر ایستای تحت مالکیت ارائه‌دهنده را برمی‌گردانند. |

    برای یک نقطه پایانی فهرست غیر Bearer یا غیراستاندارد، به‌جای
    `true` گزینه‌ها را ارسال کنید:

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    از `endpointUrl` به‌عنوان میزبان جایگزین بدون قیدوشرط استفاده نکنید. بررسی
    `requireBaseUrl` آن، مرز جداسازی اعتبارنامه برای ارائه‌دهندگانی است
    که میزبان فهرست مدل آن‌ها با میزبان استنتاجشان تفاوت دارد.

    اگر ارائه‌دهنده به‌جای بازنمایی محافظه‌کارانه سازگار با OpenAI
    به معناشناسی سفارشی مدل نیاز دارد، آن بازنمایی را در Plugin نگه دارید و برای چرخه عمر
    واکشی مشترک از `openclaw/plugin-sdk/provider-catalog-live-runtime` استفاده کنید.
    این دستیار، واکشی‌های HTTP محافظت‌شده، سرآیندهای احراز هویت ارائه‌دهنده،
    خطاهای ساختاریافته HTTP، ذخیره‌سازی TTL در حافظه نهان و رفتار بازگشت به حالت ایستا را بدون
    قراردادن سیاست ارائه‌دهنده در هسته OpenClaw فراهم می‌کند.

    وقتی API زنده فقط مشخص می‌کند کدام ردیف‌های کاتالوگ ایستای تحت مالکیت
    ارائه‌دهنده در حال حاضر در دسترس هستند، از `buildLiveModelProviderConfig` استفاده کنید:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    هنگامی از `getCachedLiveProviderModelRows` استفاده کنید که API ارائه‌دهنده فرادادهٔ غنی‌تری
    برمی‌گرداند و Plugin باید خودش ردیف‌ها را به تعریف‌های مدل OpenClaw
    نگاشت کند:

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` باید وابسته به احراز هویت باقی بماند و وقتی هیچ اعتبارنامهٔ
    قابل‌استفاده‌ای موجود نیست، `null` را برگرداند. یک `staticRun`
    آفلاین یا جایگزین ایستا نگه دارید تا راه‌اندازی، مستندات، آزمون‌ها و سطوح انتخاب‌گر
    به دسترسی زندهٔ شبکه وابسته نباشند. از TTL متناسب با تازگی فهرست مدل‌ها استفاده کنید،
    از پایش سامانهٔ فایل هنگام درخواست بپرهیزید و فقط زمانی یک
    `readRows` / `readModelId` مختص ارائه‌دهنده ارسال کنید که پاسخ
    بالادستی دارای قالب `{ data: [{ id, object }] }` سازگار با OpenAI نباشد.

    اگر ارائه‌دهندهٔ بالادستی از توکن‌های کنترلی متفاوتی نسبت به OpenClaw استفاده می‌کند،
    به‌جای جایگزین‌کردن مسیر جریان، یک تبدیل متنی دوسویهٔ کوچک اضافه کنید:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input` اعلان سیستمی نهایی و محتوای پیام متنی را پیش از
    انتقال بازنویسی می‌کند. `output` دلتاهای متنی دستیار و متن نهایی را پیش از
    آن‌که OpenClaw نشانگرهای کنترلی خودش را تجزیه کند یا تحویل به کانال انجام شود،
    بازنویسی می‌کند.

    برای ارائه‌دهندگان همراهی که فقط یک ارائه‌دهندهٔ متنی با احراز هویت
    مبتنی بر کلید API و یک زمان‌اجرای مبتنی بر کاتالوگ ثبت می‌کنند، راهکار محدودتر
    `defineSingleProviderPluginEntry(...)` را ترجیح دهید:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API key",
            hint: "API key from your Acme AI dashboard",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Enter your Acme AI API key",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider` مسیر کاتالوگ زنده‌ای است که وقتی OpenClaw بتواند احراز هویت
    واقعی ارائه‌دهنده را برطرف کند، استفاده می‌شود. این مسیر می‌تواند کشف مختص
    ارائه‌دهنده را انجام دهد. از `buildStaticProvider` فقط برای ردیف‌های آفلاینی استفاده
    کنید که نمایش آن‌ها پیش از پیکربندی احراز هویت ایمن است؛ این مسیر نباید به
    اعتبارنامه نیاز داشته باشد یا درخواست شبکه‌ای ایجاد کند.
    نمایش `models list --all` در OpenClaw درحال‌حاضر کاتالوگ‌های ایستا را
    فقط برای Pluginهای ارائه‌دهندهٔ همراه، با پیکربندی خالی، محیط خالی و بدون
    مسیرهای عامل/فضای کاری اجرا می‌کند.

    اگر جریان احراز هویت شما باید هنگام ورود اولیه، `models.providers.*`، نام‌های مستعار
    و مدل پیش‌فرض عامل را نیز اصلاح کند، از راهکارهای ازپیش‌تنظیم‌شدهٔ
    `openclaw/plugin-sdk/provider-onboard` استفاده کنید. محدودترین راهکارها عبارت‌اند از
    `createDefaultModelPresetAppliers(...)`،
    `createDefaultModelsPresetAppliers(...)` و
    `createModelCatalogPresetAppliers(...)`.

    وقتی نقطهٔ پایانی بومی یک ارائه‌دهنده از بلوک‌های مصرف جریانی روی
    انتقال عادی `openai-completions` پشتیبانی می‌کند، به‌جای قراردادن بررسی‌های
    شناسهٔ ارائه‌دهنده به‌صورت ثابت در کد، راهکارهای کاتالوگ مشترک در
    `openclaw/plugin-sdk/provider-catalog-shared` را ترجیح دهید. `supportsNativeStreamingUsageCompat(...)` و
    `applyProviderNativeStreamingUsageCompat(...)` پشتیبانی را از نگاشت قابلیت‌های
    نقطهٔ پایانی تشخیص می‌دهند؛ بنابراین نقطه‌های پایانی بومی به‌سبک Moonshot/DashScope
    حتی وقتی یک Plugin از شناسهٔ سفارشی ارائه‌دهنده استفاده می‌کند نیز همچنان
    به‌صورت انتخابی فعال می‌شوند.

    نمونه‌های کشف زندهٔ بالا APIهای ارائه‌دهنده به‌سبک `/models` را پوشش
    می‌دهند. این کشف را درون `catalog.run`، مشروط به وجود احراز هویت
    قابل‌استفاده، نگه دارید و `staticRun` را برای تولید کاتالوگ آفلاین
    بدون دسترسی به شبکه نگه دارید.

  </Step>

  <Step title="افزودن تفکیک پویای مدل">
    اگر ارائه‌دهندهٔ شما شناسه‌های دلخواه مدل را می‌پذیرد (مانند پراکسی یا مسیریاب)،
    `resolveDynamicModel` را اضافه کنید:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    اگر تفکیک به فراخوانی شبکه نیاز دارد، برای آماده‌سازی اولیهٔ ناهمگام از
    `prepareDynamicModel` استفاده کنید — `resolveDynamicModel` پس از تکمیل آن دوباره
    اجرا می‌شود.

  </Step>

  <Step title="افزودن هوک‌های زمان اجرا (در صورت نیاز)">
    بیشتر ارائه‌دهندگان فقط به `catalog` + `resolveDynamicModel` نیاز دارند.
    هوک‌ها را متناسب با نیازهای ارائه‌دهنده، به‌تدریج اضافه کنید.

    سازنده‌های راهکار مشترک اکنون متداول‌ترین خانواده‌های بازپخش/سازگاری ابزار را
    پوشش می‌دهند؛ بنابراین Pluginها معمولاً نیازی ندارند هر هوک را جداگانه و دستی
    متصل کنند:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    خانواده‌های بازپخش موجود درحال‌حاضر:

    | خانواده | مواردی که متصل می‌کند | نمونه‌های همراه |
    | --- | --- | --- |
    | `openai-compatible` | خط‌مشی بازپخش مشترک به‌سبک OpenAI برای انتقال‌های سازگار با OpenAI، شامل پاک‌سازی شناسهٔ فراخوانی ابزار، اصلاح ترتیب‌های آغازشونده با دستیار و اعتبارسنجی عمومی نوبت Gemini در مواردی که انتقال به آن نیاز دارد | `moonshot`، `ollama`، `xai`، `zai` |
    | `anthropic-by-model` | خط‌مشی بازپخش آگاه از Claude که توسط `modelId` انتخاب می‌شود؛ بنابراین انتقال‌های پیام Anthropic فقط زمانی پاک‌سازی بلوک تفکر مختص Claude را دریافت می‌کنند که مدل تفکیک‌شده واقعاً یک شناسهٔ Claude باشد | `amazon-bedrock` |
    | `native-anthropic-by-model` | همان خط‌مشی Claude بر اساس مدل مانند `anthropic-by-model`، به‌علاوهٔ پاک‌سازی شناسهٔ فراخوانی ابزار و حفظ شناسهٔ بومی استفاده از ابزار Anthropic برای انتقال‌هایی که باید شناسه‌های بومی فروشنده را حفظ کنند | `anthropic-vertex`، `clawrouter` |
    | `google-gemini` | خط‌مشی بازپخش بومی Gemini به‌همراه پاک‌سازی بازپخش راه‌اندازی اولیه. خانوادهٔ مشترک، Gemini CLI با خروجی متنی را روی استدلال برچسب‌گذاری‌شده نگه می‌دارد؛ ارائه‌دهندهٔ مستقیم `google` مقدار `resolveReasoningOutputMode` را با `native` بازنویسی می‌کند، زیرا تفکر Gemini API به‌شکل بخش‌های بومی فکر دریافت می‌شود. | `google`، `google-gemini-cli` |
    | `passthrough-gemini` | پاک‌سازی امضای فکر Gemini برای مدل‌های Gemini که از طریق انتقال‌های پراکسی سازگار با OpenAI اجرا می‌شوند؛ اعتبارسنجی بازپخش بومی Gemini یا بازنویسی‌های راه‌اندازی اولیه را فعال نمی‌کند | `openrouter`، `kilocode`، `opencode`، `opencode-go` |
    | `hybrid-anthropic-openai` | خط‌مشی ترکیبی برای ارائه‌دهندگانی که سطوح مدل پیام Anthropic و سازگار با OpenAI را در یک Plugin ترکیب می‌کنند؛ حذف اختیاری بلوک تفکر مخصوص Claude همچنان به بخش Anthropic محدود می‌ماند | `minimax` |

    خانواده‌های جریان موجود درحال‌حاضر:

    | خانواده | آنچه متصل می‌کند | نمونه‌های همراه |
    | --- | --- | --- |
    | `google-thinking` | نرمال‌سازی محتوای تفکر Gemini در مسیر مشترک استریم | `google`، `google-gemini-cli` |
    | `kilocode-thinking` | پوشش‌دهنده استدلال Kilo در مسیر مشترک استریم پروکسی، به‌طوری‌که `kilo-auto/balanced` و شناسه‌های پشتیبانی‌نشده استدلال پروکسی از تزریق تفکر صرف‌نظر می‌کنند | `kilocode` |
    | `moonshot-thinking` | نگاشت محتوای بومی تفکر دودویی Moonshot از پیکربندی + سطح `/think` | `moonshot` |
    | `minimax-fast-mode` | بازنویسی مدل حالت سریع MiniMax در مسیر مشترک استریم | `minimax`، `minimax-portal` |
    | `openai-responses-defaults` | پوشش‌دهنده‌های مشترک و بومی Responses برای OpenAI/Codex: سرآیندهای انتساب، `/fast`/`serviceTier`، میزان تفصیل متن، جست‌وجوی وب بومی Codex، شکل‌دهی محتوای سازگار با استدلال و مدیریت زمینه Responses | `openai` |
    | `openrouter-thinking` | پوشش‌دهنده استدلال OpenRouter برای مسیرهای پروکسی، با مدیریت متمرکز صرف‌نظرکردن برای مدل پشتیبانی‌نشده/`auto` | `openrouter` |
    | `tool-stream-default-on` | پوشش‌دهنده `tool_stream` با فعال‌بودن پیش‌فرض برای ارائه‌دهندگانی مانند Z.AI که استریم ابزار را می‌خواهند، مگر آنکه صریحاً غیرفعال شده باشد | `zai` |

    <Accordion title="درزهای SDK زیربنای سازندگان خانواده">
      هر سازنده خانواده از کمک‌کننده‌های عمومی سطح پایین‌تری که از همان بسته صادر می‌شوند تشکیل شده است؛ وقتی ارائه‌دهنده‌ای باید از الگوی رایج خارج شود، می‌توان از آن‌ها استفاده کرد:

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`، `buildProviderReplayFamilyHooks(...)` و سازندگان خام بازپخش (`buildOpenAICompatibleReplayPolicy`، `buildAnthropicReplayPolicyForModel`، `buildGoogleGeminiReplayPolicy`، `buildHybridAnthropicOrOpenAIReplayPolicy`). همچنین کمک‌کننده‌های بازپخش Gemini (`sanitizeGoogleGeminiReplayHistory`، `resolveTaggedReasoningOutputMode`) و کمک‌کننده‌های نقطه پایانی/مدل (`resolveProviderEndpoint`، `normalizeProviderId`، `normalizeGooglePreviewModelId`) را صادر می‌کند.
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`، `buildProviderStreamFamilyHooks(...)`، `composeProviderStreamWrappers(...)`، به‌علاوه پوشش‌دهنده‌های مشترک OpenAI/Codex (`createOpenAIAttributionHeadersWrapper`، `createOpenAIFastModeWrapper`، `createOpenAIServiceTierWrapper`، `createOpenAIResponsesContextManagementWrapper`، `createCodexNativeWebSearchWrapper`)، پوشش‌دهنده سازگار با OpenAI برای DeepSeek V4 (`createDeepSeekV4OpenAICompatibleThinkingWrapper`)، پاک‌سازی پیش‌پرکردن تفکر در Anthropic Messages (`createAnthropicThinkingPrefillPayloadWrapper`)، سازگاری فراخوانی ابزار با متن ساده (`createPlainTextToolCallCompatWrapper`) و پوشش‌دهنده‌های مشترک پروکسی/ارائه‌دهنده (`createOpenRouterWrapper`، `createToolStreamWrapper`، `createMinimaxFastModeWrapper`).
      - `openclaw/plugin-sdk/provider-stream-shared` - پوشش‌دهنده‌های سبک محتوای ارسالی و رویداد برای مسیرهای پرترافیک ارائه‌دهنده، شامل `createOpenAICompatibleCompletionsThinkingOffWrapper`، `createPayloadPatchStreamWrapper`، `createPlainTextToolCallCompatWrapper`، `normalizeOpenAICompatibleReasoningPayload(...)` و `setQwenChatTemplateThinking(...)`.
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`، `buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")` و کمک‌کننده‌های زیربنایی شِمای ارائه‌دهنده.

      برای ارائه‌دهندگان خانواده Gemini، حالت خروجی استدلال را با
      انتقال هماهنگ نگه دارید. ارائه‌دهندگان مستقیم Google Gemini API باید از خروجی استدلال
      `native` استفاده کنند تا OpenClaw بخش‌های بومی تفکر را بدون افزودن
      دستورالعمل‌های پرامپت `<think>` / `<final>` مصرف کند. بک‌اندهای فقط‌متنی
      به سبک Gemini CLI که پاسخ نهایی JSON/متن را تجزیه می‌کنند، می‌توانند قرارداد برچسب‌دار مشترک
      `google-gemini` را حفظ کنند.

      برخی کمک‌کننده‌های استریم عمداً محلیِ ارائه‌دهنده باقی می‌مانند. `@openclaw/anthropic-provider`، `wrapAnthropicProviderStream`، `resolveAnthropicBetas`، `resolveAnthropicFastMode`، `resolveAnthropicServiceTier` و سازندگان سطح پایین‌تر پوشش‌دهنده Anthropic را در درز عمومی `api.ts` / `contract-api.ts` خود نگه می‌دارد، زیرا مدیریت بتای OAuth مربوط به Claude و دروازه‌بندی `context1m` را کدگذاری می‌کنند. Plugin مربوط به xAI نیز شکل‌دهی بومی Responses برای xAI را در `wrapStreamFn` خود نگه می‌دارد (نام‌های مستعار `/fast`، مقدار پیش‌فرض `tool_stream`، پاک‌سازی ابزار سخت‌گیرانه پشتیبانی‌نشده و حذف محتوای استدلال مختص xAI).

      همین الگوی ریشه بسته، زیربنای `@openclaw/openai-provider` (سازندگان ارائه‌دهنده، کمک‌کننده‌های مدل پیش‌فرض، سازندگان ارائه‌دهنده بلادرنگ) و `@openclaw/openrouter-provider` (سازنده ارائه‌دهنده به‌همراه کمک‌کننده‌های راه‌اندازی اولیه/پیکربندی) نیز هست.
    </Accordion>

    <Tabs>
      <Tab title="مبادله توکن">
        برای ارائه‌دهندگانی که پیش از هر فراخوانی استنتاج به مبادله توکن نیاز دارند:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="سرآیندهای سفارشی">
        برای ارائه‌دهندگانی که به سرآیندهای درخواست سفارشی یا تغییرات بدنه نیاز دارند:

        ```typescript
        // wrapStreamFn یک StreamFn مشتق‌شده از ctx.streamFn را برمی‌گرداند
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="هویت انتقال بومی">
        برای ارائه‌دهندگانی که به سرآیندهای بومی درخواست/نشست یا فراداده در
        انتقال‌های عمومی HTTP یا WebSocket نیاز دارند:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="مصرف و صورت‌حساب">
        برای ارائه‌دهندگانی که داده‌های مصرف/صورت‌حساب را ارائه می‌کنند:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` سه نتیجه دارد. زمانی
        `{ token, accountId?, subscriptionType?, rateLimitTier? }` را برگردانید که
        ارائه‌دهنده یک اعتبارنامه مصرف/صورت‌حساب دارد (فیلدهای اختیاری، فراداده غیرمحرمانه طرح را
        از نمایه حل‌شده به `fetchUsageSnapshot` منتقل می‌کنند).
        فقط زمانی `{ handled: true }` را برگردانید که ارائه‌دهنده احراز هویت مصرف را قطعاً
        مدیریت کرده، اما هیچ توکن مصرف قابل‌استفاده‌ای ندارد و OpenClaw باید از پس‌گرد عمومی
        کلید API/OAuth صرف‌نظر کند. زمانی `null` یا `undefined` را برگردانید که ارائه‌دهنده
        درخواست را مدیریت نکرده و OpenClaw باید پس‌گرد عمومی را ادامه دهد.

        شناسه ارائه‌دهنده را در `contracts.usageProviders` اعلام کنید. وقتی آن قرارداد مانیفست
        و **هر دو** هوک وجود داشته باشند، OpenClaw ارائه‌دهنده را بدون بارگیری Pluginهای
        نامرتبط ارائه‌دهنده، به‌طور خودکار در جمع‌آوری مصرف می‌گنجاند. هیچ به‌روزرسانی فهرست مجاز هسته لازم نیست.
        `fetchUsageSnapshot` ساختار مشترک و مستقل از ارائه‌دهنده را برمی‌گرداند:

        - `plan`: برچسب اشتراک یا کلید گزارش‌شده از سوی ارائه‌دهنده
        - `windows`: بازه‌های سهمیه بازنشانی‌پذیر به‌صورت درصد مصرف‌شده
        - `billing`: ورودی‌های نوع‌دار `balance`، `spend` یا `budget`؛ `unit` می‌تواند
          یک ارز ISO یا واحد ارائه‌دهنده مانند `credits` باشد
        - `summary`: زمینه فشرده مختص ارائه‌دهنده که در آن
          فیلدهای ساختاریافته جا نمی‌گیرد

        معنای ارز را دقیق نگه دارید. اعتبار یک ارائه‌دهنده USD نیست، مگر اینکه
        قرارداد بالادستی چنین بگوید. Pluginای که فقط
        `fetchUsageSnapshot` را پیاده‌سازی می‌کند، همچنان برای فراخوان‌های صریح/مصنوعی در دسترس است، اما
        به‌طور خودکار کشف نمی‌شود، زیرا OpenClaw نمی‌تواند اعتبارنامه مصرف آن را حل کند.
      </Tab>
    </Tabs>

    <Accordion title="هوک‌های رایج ارائه‌دهنده">
      OpenClaw هوک‌ها را برای Pluginهای مدل/ارائه‌دهنده تقریباً با این ترتیب فراخوانی می‌کند.
      بیشتر ارائه‌دهندگان فقط از 2-3 مورد استفاده می‌کنند. این قرارداد کامل `ProviderPlugin`
      نیست؛ برای فهرست کامل و درحال‌حاضر دقیق هوک‌ها و نکات پس‌گرد، به [جزئیات داخلی: هوک‌های زمان اجرای
      ارائه‌دهنده](/fa/plugins/architecture-internals#provider-runtime-hooks) مراجعه کنید.
      فیلدهای ارائه‌دهنده که فقط برای سازگاری هستند و OpenClaw دیگر آن‌ها را فراخوانی نمی‌کند، مانند
      `ProviderPlugin.capabilities` و `suppressBuiltInModel`، در اینجا
      فهرست نشده‌اند.

      | هوک | زمان استفاده |
      | --- | --- |
      | `catalog` | کاتالوگ مدل یا مقادیر پیش‌فرض URL پایه |
      | `applyConfigDefaults` | مقادیر پیش‌فرض سراسری متعلق به ارائه‌دهنده هنگام مادی‌سازی پیکربندی |
      | `normalizeModelId` | پاک‌سازی نام مستعار شناسه مدل قدیمی/پیش‌نمایش پیش از جست‌وجو |
      | `normalizeTransport` | پاک‌سازی `api` / `baseUrl` خانواده ارائه‌دهنده پیش از مونتاژ عمومی مدل |
      | `normalizeConfig` | نرمال‌سازی پیکربندی `models.providers.<id>` |
      | `applyNativeStreamingUsageCompat` | بازنویسی‌های بومی سازگاری مصرف استریم برای ارائه‌دهندگان پیکربندی |
      | `resolveConfigApiKey` | حل احراز هویت نشانگر محیطی متعلق به ارائه‌دهنده |
      | `resolveSyntheticAuth` | احراز هویت مصنوعی محلی/خودمیزبان یا مبتنی بر پیکربندی |
      | `resolveExternalAuthProfiles` | هم‌پوشانی نمایه‌های احراز هویت خارجی متعلق به ارائه‌دهنده برای اعتبارنامه‌های مدیریت‌شده با CLI/برنامه |
      | `shouldDeferSyntheticProfileAuth` | پایین‌آوردن جای‌نگهدارهای مصنوعی نمایه ذخیره‌شده پشت احراز هویت محیطی/پیکربندی |
      | `resolveDynamicModel` | پذیرش شناسه‌های دلخواه مدل بالادستی |
      | `prepareDynamicModel` | واکشی ناهمگام فراداده پیش از حل |
      | `normalizeResolvedModel` | بازنویسی‌های انتقال پیش از اجراکننده |
      | `normalizeToolSchemas` | پاک‌سازی شِمای ابزار متعلق به ارائه‌دهنده پیش از ثبت |
      | `inspectToolSchemas` | عیب‌یابی شِمای ابزار متعلق به ارائه‌دهنده |
      | `resolveReasoningOutputMode` | قرارداد خروجی استدلال برچسب‌دار در برابر بومی |
      | `prepareExtraParams` | پارامترهای پیش‌فرض درخواست |
      | `createStreamFn` | انتقال کاملاً سفارشی StreamFn |
      | `wrapStreamFn` | پوشش‌دهنده‌های سفارشی سرآیند/بدنه در مسیر عادی استریم |
      | `resolveTransportTurnState` | سرآیندها/فراداده بومی هر نوبت |
      | `resolveWebSocketSessionPolicy` | سرآیندها/دوره انتظار نشست بومی WS |
      | `formatApiKey` | ساختار سفارشی توکن زمان اجرا |
      | `refreshOAuth` | نوسازی سفارشی OAuth |
      | `buildAuthDoctorHint` | راهنمای ترمیم احراز هویت |
      | `matchesContextOverflowError` | تشخیص سرریز متعلق به ارائه‌دهنده |
      | `classifyFailoverReason` | طبقه‌بندی محدودیت نرخ/اضافه‌بار متعلق به ارائه‌دهنده |
      | `isCacheTtlEligible` | دروازه‌بندی TTL حافظه نهان پرامپت |
      | `buildMissingAuthMessage` | راهنمای سفارشی نبود احراز هویت |
      | `augmentModelCatalog` | ردیف‌های مصنوعی سازگاری رو به جلو (منسوخ‌شده؛ `registerModelCatalogProvider` ترجیح داده می‌شود) |
      | `resolveThinkingProfile` | مجموعه گزینه `/think` مختص مدل |
      | `isBinaryThinking` | سازگاری روشن/خاموش تفکر دودویی (منسوخ‌شده؛ `resolveThinkingProfile` ترجیح داده می‌شود) |
      | `supportsXHighThinking` | سازگاری پشتیبانی استدلال `xhigh` (منسوخ‌شده؛ `resolveThinkingProfile` ترجیح داده می‌شود) |
      | `resolveDefaultThinkingLevel` | سازگاری سیاست پیش‌فرض `/think` (منسوخ‌شده؛ `resolveThinkingProfile` ترجیح داده می‌شود) |
      | `isModernModelRef` | تطبیق مدل زنده/آزمون دود |
      | `prepareRuntimeAuth` | مبادله توکن پیش از استنتاج |
      | `resolveUsageAuth` | تجزیه سفارشی اعتبارنامه مصرف |
      | `fetchUsageSnapshot` | نقطه پایانی سفارشی مصرف |
      | `createEmbeddingProvider` | آداپتور تعبیه‌سازی متعلق به ارائه‌دهنده برای حافظه/جست‌وجو |
      | `buildReplayPolicy` | سیاست سفارشی بازپخش/Compaction رونوشت |
      | `sanitizeReplayHistory` | بازنویسی‌های بازپخش مختص ارائه‌دهنده پس از پاک‌سازی عمومی |
      | `validateReplayTurns` | اعتبارسنجی سخت‌گیرانه نوبت بازپخش پیش از اجراکننده تعبیه‌شده |
      | `onModelSelected` | فراخوانی بازگشتی پس از انتخاب (برای مثال، تله‌متری) |

      نکات پس‌گرد زمان اجرا:

      - `normalizeConfig` برای هر شناسه ارائه‌دهنده یک Plugin مالک را تعیین می‌کند (ابتدا ارائه‌دهندگان همراه، سپس Plugin زمان‌اجرای منطبق) و فقط همان قلاب را فراخوانی می‌کند؛ هیچ پیمایشی در سایر ارائه‌دهندگان انجام نمی‌شود. قلاب `normalizeConfig` خود Google است که ورودی‌های پیکربندی `google` / `google-vertex` / `google-antigravity` را نرمال‌سازی می‌کند؛ این یک بازگشت جایگزین جداگانه در هسته نیست.
      - `resolveConfigApiKey` در صورت ارائه‌شدن قلاب ارائه‌دهنده، از آن استفاده می‌کند. Amazon Bedrock تشخیص نشانگرهای محیطی AWS را در Plugin ارائه‌دهنده خود نگه می‌دارد؛ خود احراز هویت زمان اجرا، وقتی با `auth: "aws-sdk"` پیکربندی شده باشد، همچنان از زنجیره پیش‌فرض AWS SDK استفاده می‌کند.
      - `resolveThinkingProfile(ctx)`، `provider` و `modelId` انتخاب‌شده، راهنمای اختیاری ادغام‌شده کاتالوگ `reasoning` و اطلاعات اختیاری ادغام‌شده مدل `compat` را دریافت می‌کند. از `compat` فقط برای انتخاب رابط کاربری/پروفایل تفکر ارائه‌دهنده استفاده کنید.
      - `resolveSystemPromptContribution` به ارائه‌دهنده اجازه می‌دهد راهنمایی آگاه از کش برای اعلان سیستمی یک خانواده مدل تزریق کند. وقتی رفتار به یک خانواده ارائه‌دهنده/مدل تعلق دارد و باید تفکیک کش پایدار/پویا را حفظ کند، آن را به قلاب قدیمی سراسری Plugin یعنی `before_prompt_build` ترجیح دهید.

    </Accordion>

  </Step>

  <Step title="افزودن قابلیت‌های بیشتر (اختیاری)">
    ### گام 5: افزودن قابلیت‌های بیشتر

    یک Plugin ارائه‌دهنده می‌تواند در کنار استنتاج متنی، تعبیه‌سازی، گفتار، رونویسی بلادرنگ،
    صدای بلادرنگ، درک رسانه، تولید تصویر، تولید ویدئو،
    واکشی وب و جست‌وجوی وب را ثبت کند. OpenClaw این مورد را به‌عنوان یک Plugin
    با **قابلیت ترکیبی** دسته‌بندی می‌کند؛ الگوی پیشنهادی برای Pluginهای شرکتی
    (یک Plugin برای هر فروشنده). به
    [جزئیات داخلی: مالکیت قابلیت‌ها](/fa/plugins/architecture#capability-ownership-model) مراجعه کنید.

    هر قابلیت را درون `register(api)` و در کنار فراخوانی فعلی
    `api.registerProvider(...)` ثبت کنید. فقط زبانه‌های موردنیاز را انتخاب کنید:

    <Tabs>
      <Tab title="گفتار (TTS)">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        برای خطاهای HTTP ارائه‌دهنده از `assertOkOrThrowProviderError(...)` استفاده کنید تا
        Pluginها خواندن محدودشده بدنه خطا، تجزیه خطای JSON و
        پسوندهای شناسه درخواست را به‌اشتراک بگذارند.
      </Tab>
      <Tab title="رونویسی بلادرنگ">
        `createRealtimeTranscriptionWebSocketSession(...)` را ترجیح دهید؛ راهکار کمکی مشترک،
        ثبت پراکسی، وقفه تصاعدی اتصال مجدد، تخلیه هنگام بسته‌شدن، دست‌دهی‌های
        آمادگی، صف‌بندی صدا و تشخیص‌های رویداد بسته‌شدن را مدیریت می‌کند. Plugin شما
        فقط رویدادهای بالادستی را نگاشت می‌کند.

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        ارائه‌دهندگان STT دسته‌ای که صدای چندبخشی را با POST ارسال می‌کنند باید از
        `buildAudioTranscriptionFormData(...)` در
        `openclaw/plugin-sdk/provider-http` استفاده کنند. این راهکار کمکی نام فایل‌های
        بارگذاری را نرمال‌سازی می‌کند، از جمله بارگذاری‌های AAC که برای APIهای
        رونویسی سازگار به نام فایلی با سبک M4A نیاز دارند.
      </Tab>
      <Tab title="صدای بلادرنگ">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // Set this only if the provider accepts multiple tool responses for
            // one call, for example an immediate "working" response followed by
            // the final result.
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        `capabilities` را اعلام کنید تا `talk.catalog` بتواند حالت‌ها،
        انتقال‌ها، قالب‌های صوتی و پرچم‌های ویژگی معتبر را در اختیار سرویس‌گیرنده‌های
        مرورگری و بومی Talk قرار دهد. وقتی یک انتقال می‌تواند تشخیص دهد که
        انسان در حال قطع پخش دستیار است و ارائه‌دهنده از کوتاه‌سازی یا پاک‌کردن
        پاسخ صوتی فعال پشتیبانی می‌کند، `handleBargeIn` را پیاده‌سازی کنید.
        `submitToolResult` می‌تواند برای ارسال همگام `void` یا برای
        مرز تکمیل ناهمگامی که پل ارائه‌دهنده می‌تواند ارائه کند، یک
        `Promise<void>` برگرداند. نشست‌های رله Gateway پیش از
        تأیید نتیجه نهایی یا پاک‌کردن اجرای پیوندخورده منتظر آن promise می‌مانند؛
        در صورت شکست ارسال، آن را رد کنید.
        وقتی ارائه‌دهنده نمی‌تواند `options.suppressResponse` را رعایت کند،
        `supportsToolResultSuppression: false` را تنظیم کنید. در این صورت OpenClaw از سرکوب نتایج
        داخلی مشورت اجباری و لغو خودداری می‌کند و به‌جای آغاز بی‌سروصدای
        پاسخ، درخواست‌های مستقیم نتیجه سرکوب‌شده را رد می‌کند.
        مصرف‌کنندگان `createRealtimeVoiceBridgeSession` نیز می‌توانند از
        `onToolCall` یک promise برگردانند؛ پرتاب‌های همگام و ردشدن‌ها به
        فراخوان بازگشتی `onError` نشست هدایت می‌شوند.
        `handlesInputAudioBargeIn` را فقط زمانی تنظیم کنید که VAD ارائه‌دهنده با
        فراخوانی `onClearAudio("barge-in")` وقوع وقفه را تأیید کند. ارائه‌دهندگانی که
        این پرچم را حذف می‌کنند از تشخیص جایگزین محلی صدای ورودی OpenClaw استفاده می‌کنند.
      </Tab>
      <Tab title="درک رسانه">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "A photo of..." }),
          transcribeAudio: async (req) => ({ text: "Transcript..." }),
        });
        ```

        ارائه‌دهندگان رسانه محلی یا خودمیزبان که عمداً به
        اعتبارنامه نیاز ندارند، می‌توانند `resolveAuth` را ارائه دهند و
        `kind: "none"` را برگردانند. OpenClaw همچنان دروازه عادی احراز هویت را
        برای ارائه‌دهندگانی که صراحتاً اعلام مشارکت نمی‌کنند حفظ می‌کند.
        ارائه‌دهندگان فعلی می‌توانند به خواندن `req.apiKey` ادامه دهند؛
        ارائه‌دهندگان جدید باید `req.auth` را ترجیح دهند.

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio plugin no-auth",
          }),
          transcribeAudio: async (req) => ({ text: "Transcript..." }),
        });
        ```
      </Tab>
      <Tab title="تعبیه‌سازی‌ها">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        همان شناسه را در `contracts.embeddingProviders` اعلام کنید. این قرارداد عمومی
        تعبیه‌سازی برای تولید بردار قابل‌استفاده مجدد، از جمله جست‌وجوی حافظه است.
        `registerMemoryEmbeddingProvider(...)` سازگاری منسوخ‌شده‌ای برای
        آداپتورهای فعلی ویژه حافظه است.
      </Tab>
      <Tab title="تولید تصویر و ویدئو">
        قابلیت‌های تصویر و ویدئو از ساختاری **آگاه از حالت** استفاده می‌کنند.
        ارائه‌دهندگان تصویر بلوک‌های قابلیت الزامی `generate` و `edit`
        را اعلام می‌کنند؛ ارائه‌دهندگان ویدئو `generate`، `imageToVideo` و
        `videoToVideo` را اعلام می‌کنند. فیلدهای تجمیعی تخت مانند `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` برای اعلام درست پشتیبانی از
        حالت تبدیل یا حالت‌های غیرفعال کافی نیستند. تولید موسیقی نیز از همان
        الگوی `generate` / `edit` پیروی می‌کند.

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "تصاویر Acme",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "ویدیوی Acme",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` برای هر دو نوع ارائه‌دهنده الزامی است؛ `edit` و بلوک‌های
        تبدیل ویدیو (`imageToVideo`، `videoToVideo`) همیشه به یک پرچم
        صریح `enabled` نیاز دارند.

        هنگامی از `catalogByModel` استفاده کنید که حالت‌ها یا قابلیت‌های ایستای یک مدل فهرست‌شده
        با پیش‌فرض‌های ارائه‌دهنده متفاوت باشند. این فراداده، بدون
        فراخوانی کد ارائه‌دهنده، `video_generate action=list` و کاتالوگ‌های مدل را دقیق نگه می‌دارد.
        جست‌وجو و اعمال قابلیت‌ها هنگام درخواست همچنان
        در `resolveModelCapabilities` و `generateVideo` انجام می‌شود؛ در صورت امکان،
        برای هر دو مسیر از ثابت قابلیت یکسان استفاده کنید.
      </Tab>
      <Tab title="واکشی و جست‌وجوی وب">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "واکشی Acme",
          hint: "صفحه‌ها را از طریق بک‌اند رندر Acme واکشی کنید.",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "یک صفحه را از طریق واکشی Acme دریافت کنید.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "جست‌وجوی Acme",
          hint: "وب را از طریق بک‌اند جست‌وجوی Acme جست‌وجو کنید.",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "وب را از طریق جست‌وجوی Acme جست‌وجو کنید.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        هر دو نوع ارائه‌دهنده ساختار یکسانی برای اتصال اطلاعات اعتبارنامه دارند:
        `hint`، `envVars`، `placeholder`، `signupUrl`، `credentialPath`،
        `getCredentialValue`، `setCredentialValue` و `createTool` همگی
        الزامی هستند.
      </Tab>
    </Tabs>

  </Step>

  <Step title="آزمایش">
    ### گام 6: آزمایش

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // شیء پیکربندی ارائه‌دهنده را از index.ts یا یک فایل اختصاصی صادر کنید
    import { acmeProvider } from "./provider.js";

    describe("ارائه‌دهنده acme-ai", () => {
      it("مدل‌های پویا را تفکیک می‌کند", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("هنگامی که کلید موجود است کاتالوگ را برمی‌گرداند", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("هنگامی که کلیدی وجود ندارد کاتالوگ null را برمی‌گرداند", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## انتشار در ClawHub

Pluginهای ارائه‌دهنده به همان شیوه هر Plugin کد خارجی دیگری منتشر می‌شوند:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>` فرمان متفاوتی برای انتشار یک پوشه مهارت است،
نه یک بسته Plugin؛ در اینجا از آن استفاده نکنید.

## ساختار فایل

```
<bundled-plugin-root>/acme-ai/
├── package.json              # فراداده openclaw.providers
├── openclaw.plugin.json      # مانیفست دارای فراداده احراز هویت ارائه‌دهنده
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # آزمایش‌ها
    └── usage.ts              # نقطه پایانی مصرف (اختیاری)
```

## مرجع ترتیب کاتالوگ

`catalog.order` زمان ادغام کاتالوگ شما را نسبت به ارائه‌دهندگان
داخلی کنترل می‌کند:

| ترتیب     | زمان          | مورد استفاده                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | گذر نخست    | ارائه‌دهندگان ساده مبتنی بر کلید API                         |
| `profile` | پس از ساده‌ها  | ارائه‌دهندگان وابسته به پروفایل‌های احراز هویت                |
| `paired`  | پس از پروفایل | ترکیب چند ورودی مرتبط             |
| `late`    | گذر نهایی     | بازنویسی ارائه‌دهندگان موجود (در صورت تداخل اولویت دارد) |

## گام‌های بعدی

- [Pluginهای کانال](/fa/plugins/sdk-channel-plugins) - اگر Plugin شما یک کانال نیز ارائه می‌دهد
- [زمان اجرای SDK](/fa/plugins/sdk-runtime) - کمک‌کننده‌های `api.runtime` (تبدیل متن به گفتار، جست‌وجو، عامل فرعی)
- [نمای کلی SDK](/fa/plugins/sdk-overview) - مرجع کامل واردسازی زیرمسیرها
- [جزئیات داخلی Plugin](/fa/plugins/architecture-internals#provider-runtime-hooks) - جزئیات هوک و نمونه‌های همراه

## مرتبط

- [راه‌اندازی SDK مربوط به Plugin](/fa/plugins/sdk-setup)
- [ساخت Pluginها](/fa/plugins/building-plugins)
- [ساخت Pluginهای کانال](/fa/plugins/sdk-channel-plugins)
