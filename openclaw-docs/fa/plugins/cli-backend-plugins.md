---
read_when:
    - شما در حال ساخت یک Plugin بک‌اند محلی هوش مصنوعی برای CLI هستید
    - می‌خواهید یک بک‌اند برای ارجاع‌های مدل مانند acme-cli/model ثبت کنید
    - باید یک CLI شخص ثالث را به اجراکنندهٔ جایگزین متنی OpenClaw نگاشت کنید
sidebarTitle: CLI backend plugins
summary: یک Plugin بسازید که یک بک‌اند محلی CLI هوش مصنوعی را ثبت کند
title: ساخت Pluginهای بک‌اند CLI
x-i18n:
    generated_at: "2026-07-27T15:51:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1923b0829b46a309e4b5a6cbbbfd3dcb76a1e14fe4106310d7a9fb37bca41d70
    source_path: plugins/cli-backend-plugins.md
    workflow: 16
---

Pluginهای بک‌اند CLI به OpenClaw امکان می‌دهند یک CLI هوش مصنوعی محلی را به‌عنوان بک‌اند
استنتاج متن فراخوانی کند. بک‌اند در ارجاع‌های مدل به‌صورت پیشوند ارائه‌دهنده ظاهر می‌شود:

```text
acme-cli/acme-large
```

زمانی از بک‌اند CLI استفاده کنید که یکپارچه‌سازی بالادستی از قبل به‌شکل یک فرمان محلی
ارائه شده باشد، CLI وضعیت ورود محلی را مدیریت کند، یا هنگامی که ارائه‌دهندگان API
در دسترس نیستند به یک گزینه پشتیبان نیاز باشد.

<Info>
  اگر سرویس بالادستی یک API عادی مدل HTTP ارائه می‌دهد، به‌جای آن یک
  [Plugin ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) بنویسید. اگر زمان‌اجرای بالادستی
  نشست‌های کامل عامل، رویدادهای ابزار، Compaction یا وضعیت وظایف پس‌زمینه را
  مدیریت می‌کند، از یک [چارچوب عامل](/fa/plugins/sdk-agent-harness) استفاده کنید.
</Info>

## موارد تحت مالکیت Plugin

یک Plugin بک‌اند CLI سه قرارداد دارد:

| قرارداد              | فایل                   | هدف                                                       |
| -------------------- | ---------------------- | --------------------------------------------------------- |
| ورودی بسته           | `package.json`         | OpenClaw را به ماژول زمان‌اجرای Plugin هدایت می‌کند       |
| مالکیت مانیفست       | `openclaw.plugin.json` | شناسه بک‌اند را پیش از بارگذاری زمان‌اجرا اعلام می‌کند    |
| ثبت زمان‌اجرا        | `index.ts`             | `api.registerCliBackend(...)` را با پیش‌فرض‌های فرمان فراخوانی می‌کند |

مانیفست، فراداده اکتشاف است: CLI را اجرا یا رفتار زمان‌اجرا را ثبت نمی‌کند.
رفتار زمان‌اجرا زمانی آغاز می‌شود که ورودی Plugin، `api.registerCliBackend(...)` را
فراخوانی کند.

## Plugin حداقلی بک‌اند

<Steps>
  <Step title="ایجاد فراداده بسته">
    ```json package.json
    {
      "name": "@acme/openclaw-acme-cli",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      },
      "dependencies": {
        "openclaw": "^2026.3.24"
      },
      "devDependencies": {
        "typescript": "^5.9.0"
      }
    }
    ```

    بسته‌های منتشرشده باید فایل‌های JavaScript ساخته‌شده زمان‌اجرا را ارائه کنند. اگر ورودی
    منبع شما `./src/index.ts` است، `openclaw.runtimeExtensions` را با اشاره به فایل متناظر
    JavaScript ساخته‌شده اضافه کنید. [نقاط ورود](/fa/plugins/sdk-entrypoints) را ببینید.

  </Step>

  <Step title="اعلام مالکیت بک‌اند">
    ```json openclaw.plugin.json
    {
      "id": "acme-cli",
      "name": "Acme CLI",
      "description": "Run Acme's local AI CLI through OpenClaw",
      "cliBackends": ["acme-cli"],
      "setup": {
        "cliBackends": ["acme-cli"],
        "requiresRuntime": false
      },
      "activation": {
        "onStartup": false
      },
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```

    `cliBackends` فهرست مالکیت زمان‌اجرا است؛ این فهرست به OpenClaw امکان می‌دهد
    وقتی انتخاب مدل یا `agentRuntime.id` به `acme-cli` اشاره می‌کند، Plugin را
    به‌طور خودکار بارگذاری کند.

    `setup.cliBackends` سطح راه‌اندازی مبتنی بر توصیف‌گر است. زمانی آن را اضافه کنید که
    اکتشاف مدل، فرایند آغاز به کار یا وضعیت باید بدون بارگذاری زمان‌اجرای Plugin
    بک‌اند را تشخیص دهد. تنها زمانی از `requiresRuntime: false` استفاده کنید که
    همین توصیف‌گرهای ایستا برای راه‌اندازی کافی باشند.

  </Step>

  <Step title="ثبت بک‌اند">
    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      CLI_FRESH_WATCHDOG_DEFAULTS,
      CLI_RESUME_WATCHDOG_DEFAULTS,
      type CliBackendPlugin,
    } from "openclaw/plugin-sdk/cli-backend";

    function buildAcmeCliBackend(): CliBackendPlugin {
      return {
        id: "acme-cli",
        liveTest: {
          defaultModelRef: "acme-cli/acme-large",
          defaultImageProbe: false,
          defaultMcpProbe: false,
          docker: {
            npmPackage: "@acme/acme-cli",
            binaryName: "acme",
          },
        },
        config: {
          command: "acme",
          args: ["chat", "--output-format", "stream-json", "--prompt", "{prompt}"],
          resumeArgs: [
            "chat",
            "--resume",
            "{sessionId}",
            "--output-format",
            "stream-json",
            "--prompt",
            "{prompt}",
          ],
          output: "jsonl",
          resumeOutput: "jsonl",
          jsonlDialect: "gemini-stream-json",
          input: "arg",
          modelArg: "--model",
          modelAliases: {
            large: "acme-large-2026",
            fast: "acme-fast-2026",
          },
          sessionArgs: ["--session", "{sessionId}"],
          sessionMode: "existing",
          sessionIdFields: ["session_id", "conversation_id"],
          systemPromptFileArg: "--system-file",
          systemPromptWhen: "first",
          imageArg: "--image",
          imageMode: "repeat",
          imagePathScope: "workspace",
          reliability: {
            watchdog: {
              fresh: { ...CLI_FRESH_WATCHDOG_DEFAULTS },
              resume: { ...CLI_RESUME_WATCHDOG_DEFAULTS },
            },
          },
          serialize: true,
        },
      };
    }

    export default definePluginEntry({
      id: "acme-cli",
      name: "Acme CLI",
      description: "Run Acme's local AI CLI through OpenClaw",
      register(api) {
        api.registerCliBackend(buildAcmeCliBackend());
      },
    });
    ```

    شناسه بک‌اند باید با ورودی `cliBackends` مانیفست مطابقت داشته باشد. آداپتور
    ثبت‌شده، کد مرجع Plugin است؛ پیکربندی OpenClaw بک‌اند را انتخاب می‌کند،
    اما قرارداد فرمان آن را بازنویسی نمی‌کند.

  </Step>
</Steps>

## شکل پیکربندی

`CliBackendConfig` نحوه راه‌اندازی و تجزیه CLI توسط OpenClaw را توصیف می‌کند. مثال
کامل بالا عمداً همان فیلدهای فرمان، ازسرگیری، JSONL، نام مستعار مدل، نشست، تصویر
و پایشگر آداپتور همراه `google-gemini-cli` را به‌کار می‌گیرد:

| فیلد                                                       | کاربرد                                                                            |
| --------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `command`                                                 | نام فایل اجرایی یا مسیر مطلق فرمان                                                |
| `args`                                                    | آرگومان‌های پایه برای اجراهای تازه                                                |
| `resumeArgs`                                              | آرگومان‌های جایگزین برای نشست‌های ازسرگرفته‌شده؛ از `{sessionId}` پشتیبانی می‌کند |
| `output` / `resumeOutput`                                 | تجزیه‌گر: `json`، `jsonl` یا `text`                                              |
| `jsonlDialect`                                            | گویش رویداد JSONL: `claude-stream-json` یا `gemini-stream-json`                  |
| `liveSession`                                             | حالت پردازش ماندگار CLI ‏(`claude-stdio`)                                    |
| `input`                                                   | انتقال پرامپت: `arg` یا `stdin`                                              |
| `maxPromptArgChars`                                       | حداکثر طول پرامپت در حالت `arg` پیش از بازگشت به ورودی استاندارد              |
| `env` / `clearEnv`                                        | متغیرهای محیطی اضافی برای تزریق، یا نام‌هایی که باید پیش از راه‌اندازی حذف شوند  |
| `modelArg`                                                | پرچم مورداستفاده پیش از شناسه مدل                                                |
| `modelAliases`                                            | نگاشت شناسه‌های مدل OpenClaw به شناسه‌های بومی CLI                               |
| `sessionArgs`                                             | نحوه ارسال شناسه نشست با استفاده از `{sessionId}`                                |
| `sessionMode`                                             | `always`، `existing` یا `none`                                                  |
| `sessionIdFields`                                         | فیلدهای JSON که OpenClaw از خروجی CLI می‌خواند                                   |
| `systemPromptArg` / `systemPromptFileArg`                 | انتقال پرامپت سیستمی                                                             |
| `systemPromptFileConfigArg` / `systemPromptFileConfigKey` | انتقال بازنویسی پیکربندی برای فایل پرامپت سیستمی (برای نمونه `-c`)       |
| `systemPromptMode`                                        | `append` یا `replace`                                                           |
| `systemPromptWhen`                                        | `first`، `always` یا `never`                                                   |
| `imageArg` / `imageMode`                                  | پرچم مسیر تصویر و نحوه ارسال چند تصویر (`repeat` یا `list`)             |
| `imagePathScope`                                          | محل نگهداری فایل‌های تصویر مرحله‌بندی‌شده پیش از تحویل: `temp` یا `workspace` |
| `serialize`                                               | اجراهای متعلق به یک بک‌اند را مرتب نگه می‌دارد                                   |
| `reseedFromRawTranscriptWhenUncompacted`                  | فعال‌سازی اختیاری بازنشانی محدود رونوشت خام پیش از Compaction برای بازنشانی امن نشست‌ها |
| `reliability.watchdog`                                    | تنظیم مهلت نبود خروجی، به‌طور جداگانه برای اجراهای تازه و ازسرگرفته‌شده          |

کوچک‌ترین پیکربندی ایستایی را ترجیح دهید که با CLI مطابقت دارد. فراخوان‌های برگشتی
Plugin را تنها برای رفتاری اضافه کنید که واقعاً به بک‌اند تعلق دارد.

## هوک‌های پیشرفته بک‌اند

`CliBackendPlugin` می‌تواند موارد زیر را نیز تعریف کند:

| هوک                                | کاربرد                                                                      |
| ---------------------------------- | --------------------------------------------------------------------------- |
| `normalizeConfig(config, context)` | آداپتور ایستای ثبت‌شده را با زمینه زمان‌اجرا نرمال‌سازی می‌کند              |
| `resolveExecutionArgs(ctx)`        | پرچم‌های محدود به درخواست مانند میزان تفکر یا جداسازی پرسش جانبی را اضافه می‌کند |
| `prepareExecution(ctx)`            | پل‌های موقت احراز هویت، پیکربندی یا محیط را پیش از راه‌اندازی ایجاد می‌کند |
| `transformSystemPrompt(ctx)`       | تبدیل نهایی مخصوص CLI را روی پرامپت سیستمی اعمال می‌کند                     |
| `textTransforms`                   | جایگزینی‌های دوسویه پرامپت/خروجی                                           |
| `defaultAuthProfileId`             | یک نمایه احراز هویت مشخص OpenClaw را ترجیح می‌دهد                           |
| `authEpochMode`                    | نحوه بی‌اعتبار شدن نشست‌های ذخیره‌شده CLI در اثر تغییرات احراز هویت را تعیین می‌کند |
| `nativeToolMode`                   | اعلام می‌کند ابزارهای بومی وجود ندارند، همیشه فعال‌اند یا میزبان می‌تواند آن‌ها را انتخاب کند |
| `toolAvailabilityEnforcement`      | اعلام می‌کند محدودیت‌های دقیق ابزار در آرگومان‌ها یا مرحله‌بندی اجرا اعمال می‌شوند |
| `sideQuestionToolMode`             | ابزارهای بومی غیرفعال را برای پرسش‌های جانبی `/btw` اعلام می‌کند          |
| `bundleMcp` / `bundleMcpMode`      | پل ابزار MCP حلقه‌بازگشت OpenClaw را به‌صورت اختیاری فعال می‌کند            |
| `ownsNativeCompaction`             | بک‌اند Compaction خود را مدیریت می‌کند — OpenClaw آن را واگذار می‌کند       |
| `subscriptionAuthDispatch`         | اجراهای تعبیه‌شده فعال‌شده با اطلاعات ورود اشتراک از طریق این بک‌اند اجرا می‌شوند |
| `runtimeArtifact`                  | یک راه‌انداز اسکریپت را به درخت کامل بسته همراه آن محدود می‌کند             |

این هوک‌ها را تحت مالکیت ارائه‌دهنده نگه دارید. وقتی یک هوک بک‌اند می‌تواند
رفتار را بیان کند، شاخه‌های مخصوص CLI را به هسته اضافه نکنید.

`prepareExecution(ctx)` مقدار `ctx.contextTokenBudget`، یعنی محدودیت مؤثر توکن
انتخاب‌شده برای اجرا را دریافت می‌کند. بک‌اندهایی که مالک Compaction بومی هستند، می‌توانند این
بودجه را به قرارداد راه‌اندازی ویژهٔ CLI خود نگاشت کنند.

`runtimeArtifact` تحت مالکیت Plugin است. این مورد
فقط زمانی بررسی می‌شود که یک نوبت استنتاج زنده، اختیار تأییدشدهٔ راه‌اندازی را ایجاد یا دوباره اعتبارسنجی کند؛
اجراهای عادی CLI به آن نیاز ندارند. بک‌اندی بدون این اعلان نمی‌تواند
اختیار تأییدشدهٔ راه‌اندازی CLI ایجاد کند. اعلان `bundled-package-tree`
مالک دقیق `package.json` را نام می‌برد و ایجاب می‌کند نقطهٔ ورود بسته همان
فرمان باشد. OpenClaw از درخت کامل و کران‌دار بستهٔ نصب‌شده، شامل
وابستگی‌های تو‌در‌تو، هش می‌گیرد و در برابر پیوندهای نمادین تغییرمسیر‌دهنده،
راه‌اندازهای خارج از بستهٔ اعلان‌شده، اعلان‌های الزامی وابستگی خارجی،
درخت‌های بیش‌ازحد بزرگ و اسکریپت‌های ناشناخته به‌صورت بسته شکست می‌خورد. این مورد را فقط زمانی اعلان کنید که آن
درخت شامل پیاده‌سازی کامل استنتاج باشد؛ یکپارچه‌سازی‌های اختیاری ابزار،
گراف پیاده‌سازی خارجی را ایمن نمی‌کنند.

اگر همان بک‌اند یک فایل اجرایی بومی خودبسنده نیز ارائه می‌کند، نام‌های پایهٔ
متعارف آن را در `nativeExecutableNames` فهرست کنید. سایر فرمان‌های بومی
تأییدنشده باقی می‌مانند.

`ctx.executionMode` برای نوبت‌های عادی `"agent"` و برای
فراخوانی‌های موقتی `/btw` برابر با `"side-question"` است. زمانی از آن استفاده کنید که CLI برای
BTW به پرچم‌های یک‌بارهٔ متفاوتی نیاز دارد، مانند غیرفعال‌کردن ابزارهای بومی،
ماندگاری نشست یا رفتار ازسرگیری. اگر یک بک‌اند معمولاً `nativeToolMode: "always-on"` دارد اما
argv پرسش جانبی آن، آن ابزارها را به‌طور قابل‌اعتماد غیرفعال می‌کند،
`sideQuestionToolMode: "disabled"` را نیز تنظیم کنید؛ در غیر این صورت، هنگامی که BTW
به اجرای CLI بدون ابزار نیاز دارد، OpenClaw به‌صورت بسته شکست می‌خورد.

`nativeToolMode: "selectable"` را فقط زمانی تنظیم کنید که بک‌اند بتواند همهٔ
ابزارهای بومی بک‌اند را برای یک اجرای منفرد غیرفعال کند. اجراهای محدودشده یک قرارداد
متعارف دریافت می‌کنند: `ctx.toolAvailability.native` فهرست دقیق ابزارهای بومی بک‌اند و
`ctx.toolAvailability.openClaw` فهرست دقیق نام ابزارهای OpenClaw است.
میزبان به‌طور مستقل پیکربندی و مجوز تولیدشدهٔ MCP را به همان
فهرست OpenClaw محدود می‌کند؛ Pluginها نباید آن را در هسته ترجمه کنند یا پیشوندهای انتقالی بیفزایند.

نحوهٔ اعمال این قرارداد توسط بک‌اند را اعلان کنید:

- `toolAvailabilityEnforcement: "execution-args"` به
  `resolveExecutionArgs` نیاز دارد. هوک باید پرچم‌های ابزار متعارض را جایگزین کند، سطوح
  سفارشی‌سازی‌ای را که می‌توانند خارج از ابزارهای انتخاب‌شده اجرا شوند غیرفعال کند و
  argv اعمال‌کننده را هم برای اجراهای تازه و هم ازسرگرفته‌شده بازگرداند.
- `toolAvailabilityEnforcement: "prepare-execution"` به
  `prepareExecution` نیاز دارد. هوک باید یک خط‌مشی دقیق مختص هر اجرا را آماده کند و
  `toolAvailabilityEnforced: true` را بازگرداند؛ نبود تأیید دریافت باعث شکست بسته می‌شود و
  OpenClaw منابع آماده‌شده را پیش از راه‌اندازی پاک‌سازی می‌کند.

محدودیت‌های زمان اجرا مانند `toolsAllow` مربوط به Cron، پیش از ساخته‌شدن
این قرارداد توسط OpenClaw عادی‌سازی و بر اساس گروه گسترش داده می‌شوند. ابزارهای بومی غیرفعال می‌شوند و
بک‌اندی بدون مسیر اعمال کامل و اعلان‌شده، پیش از اجرا شکست می‌خورد.

Pluginهایی که بر اساس `v2026.7.2-beta.1` تا `v2026.7.2-beta.3` ساخته شده‌اند، ممکن است همچنان
تصویر منسوخ‌شدهٔ نام انتقالی `ctx.toolAvailability.mcp` را بخوانند و
وقتی یک بک‌اند قابل‌انتخاب `resolveExecutionArgs` را پیاده‌سازی می‌کند، ممکن است
`toolAvailabilityEnforcement` را حذف کنند. OpenClaw آن مسیر بتای منتشرشده را از
فرادادهٔ الزامی `openclaw.build.openclawVersion` بستهٔ Plugin تشخیص می‌دهد و
آن را در سراسر خط `2026.8.x` حفظ می‌کند. Pluginهای جدید و به‌روزشده باید از نام‌های متعارف
`ctx.toolAvailability.openClaw` استفاده کنند و
`toolAvailabilityEnforcement: "execution-args"` را صریحاً اعلان کنند؛ مسیر
سازگاری بتا قرار است پس از آن بازه حذف شود.

### `ownsNativeCompaction`: انصراف از Compaction در OpenClaw

اگر بک‌اند شما عاملی را اجرا می‌کند که رونوشت **خودش** را فشرده می‌کند،
`ownsNativeCompaction: true` را تنظیم کنید تا خلاصه‌ساز حفاظتی OpenClaw هرگز برای
نشست‌های آن اجرا نشود؛ چرخهٔ عمر Compaction در CLI بدون انجام کاری بازمی‌گردد و
نوبت ادامه می‌یابد. `claude-cli` این مورد را اعلان می‌کند، زیرا Claude Code
به‌صورت داخلی و بدون نقطهٔ پایانی مهار، Compaction را انجام می‌دهد. نشست‌های مهار بومی مانند Codex
در عوض همچنان به نقطهٔ پایانی Compaction مهار خود هدایت می‌شوند.

**فقط زمانی آن را اعلان کنید که همهٔ شرایط زیر برقرار باشند**؛ در غیر این صورت، یک نشست معوق
بیش‌ازبودجه می‌تواند بیش‌ازبودجه یا منقضی باقی بماند (OpenClaw دیگر
آن را نجات نمی‌دهد):

- بک‌اند با نزدیک‌شدن به پنجرهٔ خود، رونوشتش را به‌طور قابل‌اعتماد فشرده یا محدود
  می‌کند؛
- نشستی قابل‌ازسرگیری را ماندگار می‌کند تا وضعیت فشرده‌شده بین نوبت‌ها حفظ شود
  (برای مثال `--resume` / `--session-id`)؛
- این نشست از نوع نشست Compaction با مهار بومی نیست؛ نشست‌های منطبق با `agentHarnessId`
  در عوض به نقطهٔ پایانی مهار هدایت می‌شوند.

## پل ابزار MCP

بک‌اندهای CLI به‌طور پیش‌فرض ابزارهای OpenClaw را دریافت نمی‌کنند. اگر CLI بتواند
پیکربندی MCP را مصرف کند، صریحاً فعالش کنید:

```typescript
return {
  id: "acme-cli",
  bundleMcp: true,
  bundleMcpMode: "codex-config-overrides",
  config: {
    command: "acme",
    args: ["chat", "--json"],
    output: "json",
  },
};
```

حالت‌های پشتیبانی‌شدهٔ پل:

| حالت                     | کاربرد                                                              |
| ------------------------ | ---------------------------------------------------------------- |
| `claude-config-file`     | CLIهایی که فایل پیکربندی MCP را می‌پذیرند                              |
| `codex-config-overrides` | CLIهایی که بازنویسی‌های پیکربندی را در argv می‌پذیرند                        |
| `gemini-system-settings` | CLIهایی که تنظیمات MCP را از دایرکتوری تنظیمات سیستم خود می‌خوانند |

پل را فقط زمانی فعال کنید که CLI واقعاً بتواند آن را مصرف کند. اگر CLI
لایهٔ ابزار داخلی خودش را دارد که نمی‌توان آن را غیرفعال کرد، `nativeToolMode:
"always-on"` را تنظیم کنید تا وقتی فراخواننده‌ای نبود ابزارهای بومی را الزامی می‌کند، OpenClaw بتواند به‌صورت بسته شکست بخورد. اگر می‌تواند همهٔ ابزارهای بومی را در هر اجرا غیرفعال کند، از `"selectable"` همراه با
قرارداد `resolveExecutionArgs` بالا استفاده کنید.

## انتخاب بک‌اند

کاربران یک بک‌اند مستقل را از طریق پیشوند ارجاع مدل آن انتخاب می‌کنند. بک‌اندی که
یک `modelProvider` متعارف اعلان می‌کند، می‌تواند در عوض از طریق
`agentRuntime.id` مدل آن ارائه‌دهنده انتخاب شود. سازوکارهای آداپتور در Plugin باقی می‌مانند:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.6-sol",
        fallbacks: ["acme-cli/large"],
      },
    },
  },
}
```

اعتبارنامه‌ها را در پروفایل‌های احراز هویت OpenClaw یا پیکربندی تحت مالکیت Plugin قرار دهید. اطمینان حاصل کنید
فرمان ثبت‌شده در `PATH` سرویس Gateway قرار دارد؛ استقرارهایی که به
مسیر یا argv متفاوتی نیاز دارند، باید ثبت Plugin را تغییر دهند یا در یک پوشش قرار دهند.

## تأیید

برای Pluginهای همراه، یک آزمون متمرکز پیرامون سازنده و ثبت
راه‌اندازی اضافه کنید، سپس مسیر آزمون هدفمند Plugin را اجرا کنید:

```bash
pnpm test extensions/acme-cli
```

برای Pluginهای محلی یا نصب‌شده، کشف و یک اجرای واقعی مدل را تأیید کنید:

```bash
openclaw plugins inspect acme-cli --runtime --json
openclaw agent --message "دقیقاً پاسخ بده: backend ok" --model acme-cli/acme-large
```

اگر بک‌اند از تصاویر یا MCP پشتیبانی می‌کند، یک آزمون دود زنده اضافه کنید که آن
مسیرها را با CLI واقعی اثبات کند. برای رفتار پرامپت، تصویر،
MCP یا ازسرگیری نشست، به بازرسی ایستا تکیه نکنید.

## چک‌لیست

<Check>`package.json` دارای `openclaw.extensions` و ورودی‌های زمان اجرای ساخته‌شده برای بسته‌های منتشرشده است</Check>
<Check>`openclaw.plugin.json`، `cliBackends` و `activation.onStartup` موردنظر را اعلان می‌کند</Check>
<Check>وقتی راه‌اندازی/کشف مدل باید بک‌اند را در حالت سرد ببیند، `setup.cliBackends` موجود است</Check>
<Check>`api.registerCliBackend(...)` از همان شناسهٔ بک‌اند در مانیفست استفاده می‌کند</Check>
<Check>پیشوند مدل بک‌اند یا `agentRuntime.id` محدود به مدل، ثبت را انتخاب می‌کند</Check>
<Check>تنظیمات نشست، پرامپت سیستم، تصویر و تجزیه‌گر خروجی با قرارداد واقعی CLI مطابقت دارند</Check>
<Check>آزمون‌های هدفمند و دست‌کم یک آزمون دود زندهٔ CLI مسیر بک‌اند را اثبات می‌کنند</Check>

## مرتبط

- [بک‌اندهای CLI](/fa/gateway/cli-backends) - انتخاب و رفتار زمان اجرا
- [ساخت Pluginها](/fa/plugins/building-plugins) - مبانی بسته و مانیفست
- [نمای کلی SDK مربوط به Plugin](/fa/plugins/sdk-overview) - مرجع API ثبت
- [مانیفست Plugin](/fa/plugins/manifest) - `cliBackends` و توصیفگرهای راه‌اندازی
- [مهار عامل](/fa/plugins/sdk-agent-harness) - زمان‌های اجرای کامل عامل خارجی
