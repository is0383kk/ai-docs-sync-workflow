---
read_when:
    - در حال نوشتن آزمون‌هایی برای یک Plugin هستید
    - به ابزارهای آزمایش از SDK افزونه نیاز دارید
    - می‌خواهید تست‌های قرارداد برای Pluginهای همراه را درک کنید
sidebarTitle: Testing
summary: ابزارها و الگوهای آزمایش برای Pluginهای OpenClaw
title: آزمایش Plugin
x-i18n:
    generated_at: "2026-07-27T17:00:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c6c050826dae3cd2c794d50b2dd95e20e6533d838161cce037742ee5fdf7e0e
    source_path: plugins/sdk-testing.md
    workflow: 16
---

مرجع ابزارهای کمکی آزمون، الگوها و اعمال lint برای
Pluginهای OpenClaw.

<Tip>
  **در جست‌وجوی نمونه‌های آزمون هستید؟** راهنماهای عملی شامل نمونه‌های تکمیل‌شده آزمون هستند:
  [آزمون‌های Plugin کانال](/fa/plugins/sdk-channel-plugins#step-6-test) و
  [آزمون‌های Plugin ارائه‌دهنده](/fa/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## ابزارهای کمکی آزمون

این زیرمسیرها نقاط ورود کد منبع محلی مخزن برای آزمون‌های Pluginهای همراه
خود OpenClaw هستند. آن‌ها خروجی‌های `package.json` منتشرشده برای Pluginهای
شخص ثالث نیستند و ممکن است Vitest یا دیگر وابستگی‌های آزمون مختص مخزن را وارد کنند.

```typescript
import {
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/channel-feedback";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";
import { AUTH_PROFILE_RUNTIME_CONTRACT } from "openclaw/plugin-sdk/agent-runtime-test-contracts";
import { createTestPluginApi } from "openclaw/plugin-sdk/plugin-test-api";
import { expectChannelInboundContextContract } from "openclaw/plugin-sdk/channel-contract-testing";
import { createStartAccountContext } from "openclaw/plugin-sdk/channel-test-helpers";
import { describePluginRegistrationContract } from "openclaw/plugin-sdk/plugin-test-contracts";
import { registerSingleProviderPlugin } from "openclaw/plugin-sdk/plugin-test-runtime";
import { describeOpenAIProviderRuntimeContract } from "openclaw/plugin-sdk/provider-test-contracts";
import { getProviderHttpMocks } from "openclaw/plugin-sdk/provider-http-test-mocks";
import { withEnv, withFetchPreconnect, withServer } from "openclaw/plugin-sdk/test-env";
import { isLiveTestEnabled } from "openclaw/plugin-sdk/test-live";
import { createRequestCaptureJsonFetch } from "openclaw/plugin-sdk/test-media-understanding";
import {
  bundledPluginRoot,
  createCliRuntimeCapture,
  typedCases,
} from "openclaw/plugin-sdk/test-fixtures";
import { mockNodeBuiltinModule } from "openclaw/plugin-sdk/test-node-mocks";
```

برای آزمون‌های Pluginهای همراه، از این زیرمسیرهای متمرکز استفاده کنید. barrel پیشین
`openclaw/plugin-sdk/testing` محلی مخزن بود، از بسته‌های عرضه‌شده
کنار گذاشته می‌شد و اکنون حذف شده است. نام مستعار پیشین `openclaw/plugin-sdk/test-utils`
نیز همراه آن حذف شد. `pnpm run lint:plugins:no-extension-test-core-imports`
(`scripts/check-no-extension-test-core-imports.ts`) آزمون‌های افزونه را روی
زیرمسیرهای متمرکز آزمون در بالا نگه می‌دارد.

### خروجی‌های موجود

| خروجی                                               | هدف                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | یک ماک حداقلی از API افزونه برای آزمون‌های واحد ثبت مستقیم بسازید. از `plugin-sdk/plugin-test-api` وارد کنید                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | فیکسچر مشترک قرارداد پروفایل احراز هویت برای آداپتورهای بومی زمان اجرای عامل. از `plugin-sdk/agent-runtime-test-contracts` وارد کنید            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | فیکسچر مشترک قرارداد جلوگیری از تحویل برای آداپتورهای بومی زمان اجرای عامل. از `plugin-sdk/agent-runtime-test-contracts` وارد کنید    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | فیکسچر مشترک قرارداد دسته‌بندی بازگشت جایگزین برای آداپتورهای بومی زمان اجرای عامل. از `plugin-sdk/agent-runtime-test-contracts` وارد کنید |
| `createParameterFreeTool`                            | فیکسچرهای شِمای ابزار پویا را برای آزمون‌های قرارداد زمان اجرای بومی بسازید. از `plugin-sdk/agent-runtime-test-contracts` وارد کنید              |
| `expectChannelInboundContextContract`                | ساختار زمینه ورودی کانال را بررسی کنید. از `plugin-sdk/channel-contract-testing` وارد کنید                                                  |
| `installChannelOutboundPayloadContractSuite`         | موارد قرارداد بار مفید خروجی کانال را نصب کنید. از `plugin-sdk/channel-contract-testing` وارد کنید                                       |
| `createStartAccountContext`                          | زمینه‌های چرخه عمر حساب کانال را بسازید. از `plugin-sdk/channel-test-helpers` وارد کنید                                                  |
| `installChannelActionsContractSuite`                 | موارد قرارداد عمومی کنش پیام کانال را نصب کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                                     |
| `installChannelSetupContractSuite`                   | موارد قرارداد عمومی راه‌اندازی کانال را نصب کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                                              |
| `installChannelStatusContractSuite`                  | موارد قرارداد عمومی وضعیت کانال را نصب کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                                             |
| `expectDirectoryIds`                                 | شناسه‌های فهرست کانال را از یک تابع فهرست‌کردن دایرکتوری بررسی کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                               |
| `assertBundledChannelEntries`                        | بررسی کنید که نقاط ورود کانال‌های همراه، قرارداد عمومی مورد انتظار را ارائه می‌کنند. از `plugin-sdk/channel-test-helpers` وارد کنید                    |
| `formatEnvelopeTimestamp`                            | مُهرهای زمانی پوشش را به‌صورت قطعی قالب‌بندی کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                                                  |
| `expectPairingReplyText`                             | متن پاسخ جفت‌سازی کانال را بررسی و کد آن را استخراج کنید. از `plugin-sdk/channel-test-helpers` وارد کنید                                    |
| `describePluginRegistrationContract`                 | بررسی‌های قرارداد ثبت افزونه را نصب کنید. از `plugin-sdk/plugin-test-contracts` وارد کنید                                              |
| `registerSingleProviderPlugin`                       | یک افزونه ارائه‌دهنده را در آزمون‌های دود بارگذار ثبت کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                         |
| `registerProviderPlugin`                             | همه انواع ارائه‌دهنده را از یک افزونه ثبت کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                                 |
| `registerProviderPlugins`                            | ثبت‌های ارائه‌دهنده را در چند افزونه ثبت کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                     |
| `requireRegisteredProvider`                          | بررسی کنید که یک مجموعه ارائه‌دهنده حاوی یک شناسه است. از `plugin-sdk/plugin-test-runtime` وارد کنید                                           |
| `createRuntimeEnv`                                   | یک محیط ماک‌شده زمان اجرای CLI/افزونه بسازید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                              |
| `createPluginRuntimeMock`                            | یک سطح ماک‌شده زمان اجرای افزونه بسازید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                                      |
| `createPluginSetupWizardStatus`                      | کمک‌تابع‌های وضعیت راه‌اندازی را برای افزونه‌های کانال بسازید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                             |
| `createTestWizardPrompter`                           | یک درخواست‌گر ماک‌شده برای راهنمای راه‌اندازی بسازید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                                       |
| `createRuntimeTaskFlow`                              | وضعیت مجزای TaskFlow زمان اجرا را ایجاد کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                                    |
| `runProviderCatalog`                                 | یک هوک کاتالوگ ارائه‌دهنده را با وابستگی‌های آزمون اجرا کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                     |
| `resolveProviderWizardOptions`                       | انتخاب‌های راهنمای راه‌اندازی ارائه‌دهنده را در آزمون‌های قرارداد حل کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                    |
| `resolveProviderModelPickerEntries`                  | ورودی‌های انتخاب‌گر مدل ارائه‌دهنده را در آزمون‌های قرارداد حل کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                    |
| `buildProviderPluginMethodChoice`                    | شناسه‌های انتخاب راهنمای ارائه‌دهنده را برای بررسی‌ها بسازید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                            |
| `setProviderWizardProvidersResolverForTest`          | ارائه‌دهندگان راهنمای ارائه‌دهنده را برای آزمون‌های مجزا تزریق کنید. از `plugin-sdk/plugin-test-runtime` وارد کنید                                        |
| `describeOpenAIProviderRuntimeContract`              | بررسی‌های قرارداد زمان اجرای خانواده ارائه‌دهنده را نصب کنید. از `plugin-sdk/provider-test-contracts` وارد کنید                                        |
| `expectPassthroughReplayPolicy`                      | بررسی کنید که سیاست‌های بازپخش ارائه‌دهنده از ابزارها و فراداده متعلق به ارائه‌دهنده عبور می‌کنند. از `plugin-sdk/provider-test-contracts` وارد کنید         |
| `runRealtimeSttLiveTest`                             | یک آزمون زنده ارائه‌دهنده بلادرنگ STT را با فیکسچرهای صوتی مشترک اجرا کنید. از `plugin-sdk/provider-test-contracts` وارد کنید                       |
| `normalizeTranscriptForMatch`                        | خروجی رونویسی زنده را پیش از بررسی‌های تقریبی نرمال‌سازی کنید. از `plugin-sdk/provider-test-contracts` وارد کنید                               |
| `expectExplicitVideoGenerationCapabilities`          | بررسی کنید که ارائه‌دهندگان ویدئو قابلیت‌های صریح حالت تولید را اعلام می‌کنند. از `plugin-sdk/provider-test-contracts` وارد کنید                   |
| `expectExplicitMusicGenerationCapabilities`          | بررسی کنید که ارائه‌دهندگان موسیقی قابلیت‌های صریح تولید/ویرایش را اعلام می‌کنند. از `plugin-sdk/provider-test-contracts` وارد کنید                   |
| `mockSuccessfulDashscopeVideoTask`                   | یک پاسخ موفق وظیفه ویدئویی سازگار با DashScope نصب کنید. از `plugin-sdk/provider-test-contracts` وارد کنید                          |
| `getProviderHttpMocks`                               | به ماک‌های اختیاری HTTP/احراز هویت ارائه‌دهنده در Vitest دسترسی پیدا کنید. از `plugin-sdk/provider-http-test-mocks` وارد کنید                                         |
| `installProviderHttpMockCleanup`                     | ماک‌های HTTP/احراز هویت ارائه‌دهنده را پس از هر آزمون بازنشانی کنید. از `plugin-sdk/provider-http-test-mocks` وارد کنید                                        |
| `installCommonResolveTargetErrorCases`               | موارد آزمون مشترک برای مدیریت خطای حل مقصد. از `plugin-sdk/channel-target-testing` وارد کنید                                  |
| `shouldAckReaction`                                  | بررسی کنید که آیا یک کانال باید واکنش تأیید اضافه کند. از `plugin-sdk/channel-feedback` وارد کنید                                            |
| `removeAckReactionAfterReply`                        | واکنش تأیید را پس از تحویل پاسخ حذف کنید. از `plugin-sdk/channel-feedback` وارد کنید                                                      |
| `createTestRegistry`                                 | یک فیکسچر رجیستری افزونه کانال بسازید. از `plugin-sdk/plugin-test-runtime` یا `plugin-sdk/channel-test-helpers` وارد کنید               |
| `createEmptyPluginRegistry`                          | یک فیکسچر رجیستری افزونه خالی بسازید. از `plugin-sdk/plugin-test-runtime` یا `plugin-sdk/channel-test-helpers` وارد کنید                |
| `setActivePluginRegistry`                            | یک فیکسچر رجیستری برای آزمون‌های زمان اجرای افزونه نصب کنید. از `plugin-sdk/plugin-test-runtime` یا `plugin-sdk/channel-test-helpers` وارد کنید   |
| `createRequestCaptureJsonFetch`                      | درخواست‌های واکشی JSON را در آزمون‌های کمک‌تابع رسانه ثبت کنید. از `plugin-sdk/test-media-understanding` وارد کنید                                     |
| `isLiveTestEnabled`                                  | آزمون‌های زنده اختیاری ارائه‌دهنده را کنترل کنید. از `plugin-sdk/test-live` وارد کنید                                                                      |
| `collectProviderApiKeys`                             | اطلاعات اعتبارسنجی را برای آزمون‌های زنده ارائه‌دهنده کشف کنید. از `plugin-sdk/test-live-auth` وارد کنید                                                    |
| `parseProviderModelMap`                              | مقادیر جایگزین مدل آزمون زنده موسیقی/ویدئو را تجزیه کنید. از `plugin-sdk/test-media-generation` وارد کنید                                              |
| `withServer`                                         | آزمون‌ها را روی یک سرور HTTP محلی یک‌بارمصرف اجرا کنید. از `plugin-sdk/test-env` وارد کنید                                                      |
| `createMockIncomingRequest`                          | یک شیء حداقلی درخواست HTTP ورودی بسازید. از `plugin-sdk/test-env` وارد کنید                                                          |
| `withFetchPreconnect`                                | آزمون‌های واکشی را با هوک‌های پیش‌اتصال نصب‌شده اجرا کنید. از `plugin-sdk/test-env` وارد کنید                                                       |
| `withEnv` / `withEnvAsync`                           | متغیرهای محیطی را موقتاً وصله کنید. از `plugin-sdk/test-env` وارد کنید                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | فیکسچرهای مجزای آزمون سامانه فایل را ایجاد کنید. از `plugin-sdk/test-env` وارد کنید                                                              |
| `createMockServerResponse`                           | یک ماک حداقلی پاسخ سرور HTTP ایجاد کنید. از `plugin-sdk/test-env` وارد کنید                                                            |
| `createProviderUsageFetch`                           | فیکسچرهای واکشی میزان استفاده ارائه‌دهنده را بسازید. از `plugin-sdk/test-env` وارد کنید                                                                   |
| `useFrozenTime` / `useRealTime`                      | زمان‌سنج‌ها را برای آزمون‌های حساس به زمان متوقف و بازیابی کنید. از `plugin-sdk/test-env` وارد کنید                                                    |
| `createCliRuntimeCapture`                            | خروجی زمان اجرای CLI را در آزمون‌ها ثبت کنید. از `plugin-sdk/test-fixtures` وارد کنید                                                              |
| `importFreshModule`                                  | برای دور زدن حافظه نهان ماژول، یک ماژول ESM را با توکن پرس‌وجوی تازه وارد کنید. از `plugin-sdk/test-fixtures` وارد کنید                             |
| `bundledPluginRoot` / `bundledPluginFile`            | مسیرهای فیکسچر منبع یا dist افزونه همراه را حل کنید. از `plugin-sdk/test-fixtures` وارد کنید                                              |
| `mockNodeBuiltinModule`                              | ماک‌های محدود Vitest برای قابلیت‌های داخلی Node را نصب کنید. از `plugin-sdk/test-node-mocks` وارد کنید                                                       |
| `createSandboxTestContext`                           | زمینه‌های آزمون سندباکس را بسازید. از `plugin-sdk/test-fixtures` وارد کنید                                                                      |
| `writeSkill`                                         | فیکسچرهای Skills را بنویسید. از `plugin-sdk/test-fixtures` وارد کنید                                                                             |
| `makeAgentAssistantMessage`                          | فیکسچرهای پیام رونویسی عامل را بسازید. از `plugin-sdk/test-fixtures` وارد کنید                                                          |
| `peekSystemEvents` / `resetSystemEventsForTest`      | فیکسچرهای رویداد سیستم را بررسی و بازنشانی کنید. از `plugin-sdk/test-fixtures` وارد کنید                                                          |
| `sanitizeTerminalText`                               | خروجی ترمینال را برای بررسی‌ها پاک‌سازی کنید. از `plugin-sdk/test-fixtures` وارد کنید                                                          |
| `countLines` / `hasBalancedFences`                   | ساختار خروجی قطعه‌بندی را بررسی کنید. از `plugin-sdk/test-fixtures` وارد کنید                                                                     |
| `typedCases`                                         | نوع‌های لفظی را برای آزمون‌های جدول‌محور حفظ کنید. از `plugin-sdk/test-fixtures` وارد کنید                                                    |

مجموعه‌آزمون‌های قرارداد Pluginهای همراه نیز از این زیرمسیرهای آزمایشی SDK برای
ابزارهای کمکی رجیستری، مانیفست، مصنوعات عمومی و فیکسچرهای زمان اجرا که فقط مخصوص آزمون هستند استفاده می‌کنند.
مجموعه‌آزمون‌های مختص هسته که به موجودی همراه OpenClaw وابسته‌اند، در عوض زیر
`src/plugins/contracts` باقی می‌مانند.

### نوع‌ها

زیرمسیرهای متمرکز آزمایش، نوع‌های مفید در فایل‌های آزمون را نیز دوباره صادر می‌کنند:

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
} from "openclaw/plugin-sdk/channel-contract";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
import type { MockFn, PluginRuntime, RuntimeEnv } from "openclaw/plugin-sdk/plugin-test-runtime";
```

## آزمایش تفکیک مقصد

از `installCommonResolveTargetErrorCases` برای افزودن حالت‌های خطای استاندارد به
تفکیک مقصد کانال استفاده کنید:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";

describe("تفکیک مقصد my-channel", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // منطق تفکیک مقصد کانال شما
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // افزودن موارد آزمون مختص کانال
  it("باید مقصدهای @username را تفکیک کند", () => {
    // ...
  });
});
```

## الگوهای آزمایش

### آزمایش قراردادهای ثبت

آزمون‌های واحدی که یک ماک دست‌نویس `api` را به `register(api)` می‌دهند،
دروازه‌های پذیرش بارگذار OpenClaw را آزمایش نمی‌کنند. برای هر سطح ثبتی که Plugin شما به آن وابسته است، دست‌کم یک
آزمون دود مبتنی بر بارگذار اضافه کنید؛ به‌ویژه برای هوک‌ها و قابلیت‌های انحصاری مانند حافظه.

بارگذار واقعی زمانی ثبت Plugin را ناموفق می‌کند که فراداده الزامی وجود نداشته باشد یا
Plugin یک API قابلیت را فراخوانی کند که مالک آن نیست. برای مثال،
`api.registerHook(...)` به نام هوک نیاز دارد و
`api.registerMemoryCapability(...)` مستلزم آن است که مانیفست Plugin یا ورودی
صادرشده، `kind: "memory"` را اعلام کند.

### آزمایش دسترسی به پیکربندی زمان اجرا

ماک مشترک زمان اجرای Plugin را از
`openclaw/plugin-sdk/plugin-test-runtime` ترجیح دهید. ابزارهای کمکی پیکربندی زمان اجرای آن،
APIهای فعلی اسنپ‌شات و تغییر را مدل‌سازی می‌کنند.

### آزمایش واحد یک Plugin کانال

```typescript
import { describe, it, expect, vi } from "vitest";

describe("Plugin my-channel", () => {
  it("باید حساب را از پیکربندی تفکیک کند", () => {
    const cfg = {
      channels: {
        "my-channel": {
          token: "test-token",
          allowFrom: ["user1"],
        },
      },
    };

    const account = myPlugin.setup.resolveAccount(cfg, undefined);
    expect(account.token).toBe("test-token");
  });

  it("باید حساب را بدون مادی‌سازی اسرار بررسی کند", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // هیچ مقدار توکنی افشا نمی‌شود
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### آزمایش واحد یک Plugin ارائه‌دهنده

```typescript
import { describe, it, expect } from "vitest";

describe("Plugin my-provider", () => {
  it("باید مدل‌های پویا را تفکیک کند", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... زمینه
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("باید هنگامی که کلید API در دسترس است کاتالوگ را برگرداند", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... زمینه
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### ماک‌کردن زمان اجرای Plugin

برای کدی که از `createPluginRuntimeStore` استفاده می‌کند، زمان اجرا را در آزمون‌ها ماک کنید:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>({
  pluginId: "test-plugin",
  errorMessage: "زمان اجرای آزمایشی تنظیم نشده است",
});

// در راه‌اندازی آزمون
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... ماک‌های دیگر
  },
  config: {
    current: vi.fn(() => ({}) as const),
    mutateConfigFile: vi.fn(),
    replaceConfigFile: vi.fn(),
  },
  // ... فضاهای نام دیگر
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// پس از آزمون‌ها
store.clearRuntime();
```

### آزمایش با استاب‌های مختص هر نمونه

استاب‌های مختص هر نمونه را به تغییر prototype ترجیح دهید:

```typescript
// ترجیحی: استاب مختص هر نمونه
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// اجتناب شود: تغییر prototype
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## آزمون‌های قرارداد (Pluginهای درون مخزن)

Pluginهای همراه، آزمون‌های قراردادی دارند که مالکیت ثبت را تأیید می‌کنند:

```bash
pnpm test src/plugins/contracts/
```

این آزمون‌ها موارد زیر را بررسی می‌کنند:

- کدام Pluginها کدام ارائه‌دهندگان را ثبت می‌کنند
- کدام Pluginها کدام ارائه‌دهندگان گفتار را ثبت می‌کنند
- درستی ساختار ثبت
- انطباق با قرارداد زمان اجرا

### اجرای آزمون‌های محدودشده

برای یک Plugin مشخص:

```bash
pnpm test <bundled-plugin-root>/my-channel/
```

فقط برای آزمون‌های قرارداد:

```bash
pnpm test src/plugins/contracts/shape.contract.test.ts
pnpm test src/plugins/contracts/auth-choice.contract.test.ts
pnpm test src/plugins/contracts/runtime-seams.contract.test.ts
```

## اعمال قواعد لینت (Pluginهای درون مخزن)

`scripts/run-additional-boundary-checks.mjs` مجموعه‌ای از بررسی‌های مرز واردسازی `lint:plugins:*`
را در CI اجرا می‌کند؛ هرکدام را می‌توان به‌صورت مستقل و محلی نیز اجرا کرد:

| فرمان                                                        | مورد اعمال‌شده                                                                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `pnpm run lint:plugins:no-monolithic-plugin-sdk-entry-imports` | Pluginهای همراه نمی‌توانند barrel ریشه یکپارچه `openclaw/plugin-sdk` را وارد کنند.              |
| `pnpm run lint:plugins:no-extension-src-imports`               | فایل‌های افزونه محیط عملیاتی نمی‌توانند درخت `src/**` مخزن را مستقیماً وارد کنند (`../../src/...`).  |
| `pnpm run lint:plugins:no-extension-test-core-imports`         | فایل‌های آزمون افزونه نمی‌توانند نام‌های مستعار حذف‌شده آزمون SDK یا دیگر ابزارهای کمکی آزمون مختص هسته را وارد کنند. |

Pluginهای خارجی مشمول این قواعد لینت نیستند، اما پیروی از همین
الگوها توصیه می‌شود.

## پیکربندی آزمون

OpenClaw از Vitest 4 با گزارش‌دهی اطلاعاتی پوشش V8 استفاده می‌کند. برای آزمون‌های Plugin:

```bash
# اجرای همه آزمون‌ها
pnpm test

# اجرای آزمون‌های یک Plugin مشخص
pnpm test <bundled-plugin-root>/my-channel/src/channel.test.ts

# اجرا با فیلتر نام آزمون مشخص
pnpm test <bundled-plugin-root>/my-channel/ -t "resolves account"

# اجرا با پوشش
pnpm test:coverage
```

اگر اجرای محلی موجب فشار حافظه می‌شود:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## مرتبط

- [نمای کلی SDK](/fa/plugins/sdk-overview) -- قراردادهای واردسازی
- [Pluginهای کانال SDK](/fa/plugins/sdk-channel-plugins) -- رابط Plugin کانال
- [Pluginهای ارائه‌دهنده SDK](/fa/plugins/sdk-provider-plugins) -- هوک‌های Plugin ارائه‌دهنده
- [ساخت Pluginها](/fa/plugins/building-plugins) -- راهنمای شروع به کار
