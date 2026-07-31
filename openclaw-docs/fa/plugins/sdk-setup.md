---
read_when:
    - در حال افزودن یک راه‌انداز پیکربندی به یک Plugin هستید
    - باید تفاوت `setup-entry.ts` و `index.ts` را درک کنید
    - در حال تعریف طرح‌واره‌های پیکربندی Plugin یا فرادادهٔ openclaw در package.json هستید
sidebarTitle: Setup and config
summary: راهنماهای گام‌به‌گام راه‌اندازی، setup-entry.ts، طرح‌واره‌های پیکربندی و فراداده‌های package.json
title: راه‌اندازی و پیکربندی Plugin
x-i18n:
    generated_at: "2026-07-27T15:58:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

مرجع بسته‌بندی Plugin (فرادادهٔ `package.json`)، مانیفست‌ها (`openclaw.plugin.json`)، ورودی‌های راه‌اندازی و طرح‌واره‌های پیکربندی.

<Tip>
**به‌دنبال یک راهنمای گام‌به‌گام هستید؟** راهنماهای عملی، بسته‌بندی را در بستر مناسب پوشش می‌دهند: [Pluginهای کانال](/plugins/sdk-channel-plugins#step-1-package-and-manifest) و [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## فرادادهٔ بسته

`package.json` شما به یک فیلد `openclaw` نیاز دارد که به سامانهٔ Plugin اعلام کند Plugin شما چه چیزی ارائه می‌دهد:

<Tabs>
  <Tab title="Plugin کانال">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "کانال من",
          "blurb": "توضیح کوتاهی دربارهٔ کانال."
        }
      }
    }
    ```
  </Tab>
  <Tab title="Plugin ارائه‌دهنده / خط مبنای ClawHub">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
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
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
انتشار خارجی در ClawHub به `compat` و `build` نیاز دارد. قطعه‌کدهای معیار انتشار در `docs/snippets/plugin-publish/` قرار دارند.
</Note>

### فیلدهای `openclaw`

<ParamField path="extensions" type="string[]">
  فایل‌های نقطهٔ ورود (نسبت به ریشهٔ بسته). ورودی‌های منبع معتبر برای توسعه در فضای کاری و checkout گیت.
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  همتاهای JavaScript ساخته‌شده برای `extensions` که هنگام بارگیری یک بستهٔ نصب‌شدهٔ npm توسط OpenClaw ترجیح داده می‌شوند. برای ترتیب تفکیک منبع/نسخهٔ ساخته‌شده، به [نقاط ورود SDK](/fa/plugins/sdk-entrypoints) مراجعه کنید.
</ParamField>
<ParamField path="setupEntry" type="string">
  ورودی سبک‌وزنِ مختص راه‌اندازی (اختیاری).
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  همتای JavaScript ساخته‌شده برای `setupEntry`. لازم است `setupEntry` نیز تنظیم شده باشد.
</ParamField>
<ParamField path="plugin" type="object">
  هویت جایگزین Plugin در `{ id, label }` که وقتی Plugin فاقد فرادادهٔ کانال/ارائه‌دهنده برای استخراج شناسه یا برچسب است، استفاده می‌شود.
</ParamField>
<ParamField path="channel" type="object">
  فرادادهٔ کاتالوگ کانال برای سطوح راه‌اندازی، انتخاب‌گر، شروع سریع و وضعیت.
</ParamField>
<ParamField path="install" type="object">
  راهنمایی‌های نصب: `npmSpec`، `localPath`، `defaultChoice`، `minHostVersion`، `expectedIntegrity`، `allowInvalidConfigRecovery`، `requiredPlatformPackages`.
</ParamField>
<ParamField path="startup" type="object">
  پرچم‌های رفتار هنگام راه‌اندازی.
</ParamField>
<ParamField path="compat" type="object">
  محدودهٔ نسخهٔ `pluginApi` که این Plugin پشتیبانی می‌کند. برای انتشار خارجی در ClawHub الزامی است.
</ParamField>

<Note>
شناسه‌های ارائه‌دهنده (`providers: string[]`) فرادادهٔ مانیفست هستند، نه فرادادهٔ بسته. آن‌ها را در `openclaw.plugin.json` اعلام کنید، نه اینجا — به [مانیفست Plugin](/fa/plugins/manifest) مراجعه کنید.
</Note>

### `openclaw.channel`

`openclaw.channel` فرادادهٔ کم‌هزینهٔ بسته برای کشف کانال و سطوح راه‌اندازی، پیش از بارگیری زمان اجرا است.

### فیلدهای راه‌اندازی متعلق به کانال

Pluginهای کانال باید فیلدهای راه‌اندازی را یک‌بار در کد زمان اجرا با `defineChannelSetupContract(...)` تعریف کنند و تصویر سریال‌پذیر متناظر را زیر `openclaw.channel.setup.fields` منتشر کنند. تعریف زمان اجرا، نوع ورودی محلی Plugin را استنتاج می‌کند، مقادیر هدایت‌شده و غیرتعاملی را تجزیه می‌کند و کلیدهای مختص کانال را از نوع‌های هسته دور نگه می‌دارد. فرادادهٔ بسته به `openclaw channels add <channel-id> --help` و `openclaw channels add --channel <channel-id> --help` اجازه می‌دهد بدون بارگیری Plugin، فقط گزینه‌های کانال انتخاب‌شده را کشف کنند.

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "نقطهٔ پایانی سرویس" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "مالک انتقال" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "نقطهٔ پایانی سرویس" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "مالک انتقال" }
          }
        ]
      }
    }
  }
}
```

انواع فیلد پشتیبانی‌شده عبارت‌اند از `string`، `boolean`، `integer`، `string-list` و `choice`. برای اطلاعات ورود از `sensitive: true` استفاده کنید. کلید هر فیلد باید با نام ویژگی camelCase پرچم بلند CLI آن، از جمله هر شکل منفی‌شده، برابر باشد؛ مانند `apiToken` برای `--api-token`. وقتی هر دو شکل مثبت و `--no-*` لازم باشند، فیلدهای بولی می‌توانند `cli.negatedFlags` را اضافه کنند. `channel`، `account` و `name` مربوط به نمایش حساب، همچنان پوشش کنترلی مشترک باقی می‌مانند.

آداپتور منتشرشدهٔ `setup`/`ChannelSetupInput` برای Pluginهای خارجی موجود همچنان در دسترس است. Pluginهای جدید باید `setupContract` را ارائه کنند؛ OpenClaw هرگاه هر دو موجود باشند، همیشه آن را ترجیح می‌دهد.

| فیلد                                  | نوع       | مفهوم آن                                                                 |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                                   | `string`   | شناسهٔ معیار کانال.                                                         |
| `label`                                | `string`   | برچسب اصلی کانال.                                                        |
| `selectionLabel`                       | `string`   | برچسب انتخاب‌گر/راه‌اندازی، هنگامی که باید با `label` متفاوت باشد.                        |
| `detailLabel`                          | `string`   | برچسب جزئیات ثانویه برای کاتالوگ‌های غنی‌تر کانال و سطوح وضعیت.       |
| `docsPath`                             | `string`   | مسیر مستندات برای پیوندهای راه‌اندازی و انتخاب.                                      |
| `docsLabel`                            | `string`   | برچسب جایگزین برای پیوندهای مستندات، هنگامی که باید با شناسهٔ کانال متفاوت باشد. |
| `blurb`                                | `string`   | توضیح کوتاه آغاز به کار/کاتالوگ.                                         |
| `order`                                | `number`   | ترتیب مرتب‌سازی در کاتالوگ‌های کانال.                                               |
| `aliases`                              | `string[]` | نام‌های مستعار جست‌وجوی اضافی برای انتخاب کانال.                                   |
| `preferOver`                           | `string[]` | شناسه‌های Plugin/کانال با اولویت پایین‌تر که این کانال باید بر آن‌ها مقدم باشد.                |
| `systemImage`                          | `string`   | نام اختیاری آیکون/تصویر سیستمی برای کاتالوگ‌های رابط کاربری کانال.                      |
| `selectionDocsPrefix`                  | `string`   | متن پیشوند پیش از پیوندهای مستندات در سطوح انتخاب.                          |
| `selectionDocsOmitLabel`               | `boolean`  | نمایش مستقیم مسیر مستندات به‌جای پیوند برچسب‌دار مستندات در متن انتخاب. |
| `selectionExtras`                      | `string[]` | رشته‌های کوتاه اضافی که به متن انتخاب افزوده می‌شوند.                               |
| `markdownCapable`                      | `boolean`  | کانال را برای تصمیم‌های قالب‌بندی خروجی، دارای قابلیت Markdown علامت‌گذاری می‌کند.      |
| `exposure`                             | `object`   | کنترل‌های نمایانی کانال برای راه‌اندازی، فهرست‌های پیکربندی‌شده و سطوح مستندات.   |
| `quickstartAllowFrom`                  | `boolean`  | این کانال را در جریان استاندارد راه‌اندازی `allowFrom` شروع سریع وارد می‌کند.         |
| `forceAccountBinding`                  | `boolean`  | حتی وقتی فقط یک حساب وجود دارد، اتصال صریح حساب را الزامی می‌کند.           |
| `preferSessionLookupForAnnounceTarget` | `boolean`  | هنگام تفکیک مقصدهای اعلان برای این کانال، جست‌وجوی نشست را ترجیح می‌دهد.       |
| `setup`                                | `object`   | فیلدهای راه‌اندازی سریال‌پذیر متعلق به کانال که برای کشف تنبل گزینه‌های CLI استفاده می‌شوند.   |

مثال:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "کانال من",
      "selectionLabel": "کانال من (خودمیزبان)",
      "detailLabel": "ربات کانال من",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "یکپارچه‌سازی گفت‌وگوی خودمیزبان مبتنی بر Webhook.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "راهنما:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` از موارد زیر پشتیبانی می‌کند:

- `configured`: کانال را در سطوح فهرست‌سازی پیکربندی‌شده/سبک وضعیت قرار می‌دهد
- `setup`: کانال را در انتخاب‌گرهای تعاملی راه‌اندازی/پیکربندی قرار می‌دهد
- `docs`: کانال را در سطوح مستندات/ناوبری به‌عنوان عمومی علامت‌گذاری می‌کند

### `openclaw.install`

`openclaw.install` فرادادهٔ بسته است، نه فرادادهٔ مانیفست.

| فیلد                        | نوع                                | معنای آن                                                                     |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | مشخصهٔ مرجع ClawHub برای جریان‌های نصب/به‌روزرسانی و نصب هنگام نیاز در راه‌اندازی اولیه. |
| `npmSpec`                    | `string`                            | مشخصهٔ مرجع npm برای جریان‌های جایگزین نصب/به‌روزرسانی.                             |
| `localPath`                  | `string`                            | مسیر توسعهٔ محلی یا نصب همراه محصول.                                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | منبع نصب ترجیحی هنگامی که چند منبع در دسترس است.                     |
| `minHostVersion`             | `string`                            | حداقل نسخهٔ پشتیبانی‌شدهٔ OpenClaw، `>=x.y.z` یا `>=x.y.z-prerelease`.            |
| `expectedIntegrity`          | `string`                            | رشتهٔ صحت مورد انتظار توزیع npm، معمولاً `sha512-...`، برای نصب‌های سنجاق‌شده.    |
| `allowInvalidConfigRecovery` | `boolean`                           | به جریان‌های نصب مجدد Plugin همراه محصول امکان می‌دهد از خطاهای مشخص پیکربندی منسوخ بازیابی شوند.  |
| `requiredPlatformPackages`   | `string[]`                          | نام‌های مستعار npm ویژهٔ پلتفرم که هنگام نصب npm الزامی و اعتبارسنجی می‌شوند.               |

<AccordionGroup>
  <Accordion title="رفتار راه‌اندازی اولیه">
    راه‌اندازی اولیهٔ تعاملی برای بخش‌های نصب هنگام نیاز از `openclaw.install` استفاده می‌کند: اگر Plugin پیش از بارگذاری زمان اجرا گزینه‌های احراز هویت ارائه‌دهنده یا فرادادهٔ راه‌اندازی/کاتالوگ کانال را ارائه کند، راه‌اندازی اولیه می‌تواند برای نصب از ClawHub، npm یا مسیر محلی درخواست دهد، Plugin را نصب یا فعال کند و سپس جریان انتخاب‌شده را ادامه دهد. گزینه‌های ClawHub از `clawhubSpec` استفاده می‌کنند و در صورت وجود ترجیح داده می‌شوند؛ گزینه‌های npm به فرادادهٔ کاتالوگ قابل‌اعتماد با `npmSpec` رجیستری نیاز دارند (نسخه‌های دقیق و `expectedIntegrity` سنجاق‌های اختیاری هستند و در صورت تنظیم، هنگام نصب/به‌روزرسانی اعمال می‌شوند). «چه چیزی نمایش داده شود» را در `openclaw.plugin.json` و «چگونه نصب شود» را در `package.json` نگه دارید.
  </Accordion>
  <Accordion title="اعمال minHostVersion">
    اگر `minHostVersion` تنظیم شده باشد، هم نصب و هم بارگذاری رجیستری مانیفستِ غیرهمراه آن را اعمال می‌کنند. میزبان‌های قدیمی‌تر از Pluginهای خارجی صرف‌نظر می‌کنند؛ رشته‌های نسخهٔ نامعتبر رد می‌شوند. فرض می‌شود Pluginهای منبع همراه محصول با نسخهٔ مخزن کاری میزبان هم‌نسخه‌اند.
  </Accordion>
  <Accordion title="نصب‌های سنجاق‌شدهٔ npm">
    برای نصب‌های سنجاق‌شدهٔ npm، نسخهٔ دقیق را در `npmSpec` نگه دارید و صحت مورد انتظار مصنوع را اضافه کنید:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="دامنهٔ allowInvalidConfigRecovery">
    `allowInvalidConfigRecovery` یک راه عبور عمومی برای پیکربندی‌های خراب نیست. این قابلیت فقط برای بازیابی محدود Pluginهای همراه محصول است و به نصب مجدد/راه‌اندازی اجازه می‌دهد بقایای شناخته‌شدهٔ ارتقا، مانند مسیر گم‌شدهٔ یک Plugin همراه محصول یا ورودی منسوخ `channels.<id>` برای همان Plugin را ترمیم کند. اگر پیکربندی به دلایل نامرتبط خراب باشد، نصب همچنان با حالت بسته شکست می‌خورد و به اپراتور اعلام می‌کند `openclaw doctor --fix` را اجرا کند.
  </Accordion>
</AccordionGroup>

### بارگذاری کاملِ به‌تعویق‌افتاده

Pluginهای کانال می‌توانند با پیکربندی زیر بارگذاری به‌تعویق‌افتاده را فعال کنند:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

هنگام فعال بودن، OpenClaw در مرحلهٔ راه‌اندازی پیش از گوش‌دادن فقط `setupEntry` را بارگذاری می‌کند، حتی برای کانال‌هایی که از قبل پیکربندی شده‌اند. ورودی کامل پس از آغاز گوش‌دادن Gateway بارگذاری می‌شود.

<Warning>
بارگذاری به‌تعویق‌افتاده را فقط زمانی فعال کنید که `setupEntry` شما همهٔ موارد موردنیاز Gateway پیش از آغاز گوش‌دادن را ثبت کند (ثبت کانال، مسیرهای HTTP، متدهای Gateway). اگر ورودی کامل مالک قابلیت‌های ضروری راه‌اندازی است، رفتار پیش‌فرض را حفظ کنید.
</Warning>

اگر ورودی راه‌اندازی/کامل شما متدهای RPC مربوط به Gateway را ثبت می‌کند، آن‌ها را زیر یک پیشوند ویژهٔ Plugin نگه دارید. فضاهای نام مدیریتی رزروشدهٔ هسته (`config.*`، `exec.approvals.*`، `wizard.*`، `update.*`) در مالکیت هسته باقی می‌مانند و همیشه به `operator.admin` نرمال‌سازی می‌شوند.

## مانیفست Plugin

هر Plugin بومی باید یک `openclaw.plugin.json` در ریشهٔ بسته ارائه کند. OpenClaw از آن برای اعتبارسنجی پیکربندی بدون اجرای کد Plugin استفاده می‌کند.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds My Plugin capabilities to OpenClaw",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook verification secret"
      }
    }
  }
}
```

برای Pluginهای کانال، `channels` را اضافه کنید (و Pluginهای ارائه‌دهنده `providers` را اضافه می‌کنند):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

حتی Pluginهای بدون پیکربندی نیز باید یک طرح‌واره ارائه کنند. طرح‌وارهٔ خالی معتبر است:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

برای مرجع کامل طرح‌واره، [مانیفست Plugin](/fa/plugins/manifest) را ببینید.

## انتشار در ClawHub

Skills و بسته‌های Plugin از فرمان‌های انتشار جداگانهٔ ClawHub استفاده می‌کنند. برای بسته‌های Plugin، از فرمان ویژهٔ بسته استفاده کنید:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>` فرمانی متفاوت برای انتشار پوشهٔ یک Skill است، نه بستهٔ Plugin. [انتشار در ClawHub](/fa/clawhub/publishing) را ببینید.
</Note>

## ورودی راه‌اندازی

`setup-entry.ts` جایگزینی سبک‌وزن برای `index.ts` است که OpenClaw هنگامی بارگذاری می‌کند که فقط به بخش‌های راه‌اندازی نیاز دارد (راه‌اندازی اولیه، ترمیم پیکربندی، بررسی کانال غیرفعال):

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

این کار از بارگذاری کد سنگین زمان اجرا (کتابخانه‌های رمزنگاری، ثبت‌های CLI، سرویس‌های پس‌زمینه) هنگام جریان‌های راه‌اندازی جلوگیری می‌کند.

کانال‌های فضای کاری همراه محصول که خروجی‌های امن برای راه‌اندازی را در ماژول‌های جانبی نگه می‌دارند، می‌توانند به‌جای `defineSetupPluginEntry(...)` از `defineBundledChannelSetupEntry(...)` در `openclaw/plugin-sdk/channel-entry-contract` استفاده کنند. آن قرارداد همراه محصول همچنین از خروجی اختیاری `runtime` پشتیبانی می‌کند تا سیم‌کشی زمان اجرا هنگام راه‌اندازی سبک‌وزن و صریح باقی بماند.

<AccordionGroup>
  <Accordion title="زمانی که OpenClaw به‌جای ورودی کامل از setupEntry استفاده می‌کند">
    - کانال غیرفعال است، اما به بخش‌های راه‌اندازی/راه‌اندازی اولیه نیاز دارد.
    - کانال فعال است، اما پیکربندی نشده است.
    - بارگذاری به‌تعویق‌افتاده فعال است (`deferConfiguredChannelFullLoadUntilAfterListen`).

  </Accordion>
  <Accordion title="مواردی که setupEntry باید ثبت کند">
    - شیء Plugin کانال (از طریق `defineSetupPluginEntry`).
    - هر مسیر HTTP که پیش از گوش‌دادن Gateway موردنیاز است.
    - هر متد Gateway که هنگام راه‌اندازی موردنیاز است.

    آن متدهای Gateway مربوط به راه‌اندازی همچنان باید از فضاهای نام مدیریتی رزروشدهٔ هسته، مانند `config.*` یا `update.*`، دوری کنند.

  </Accordion>
  <Accordion title="مواردی که setupEntry نباید شامل شود">
    - ثبت‌های CLI.
    - سرویس‌های پس‌زمینه.
    - درون‌ریزی‌های سنگین زمان اجرا (رمزنگاری، SDKها).
    - متدهای Gateway که فقط پس از راه‌اندازی موردنیازند.

  </Accordion>
</AccordionGroup>

### درون‌ریزی‌های محدود کمک‌ابزار راه‌اندازی

برای مسیرهای داغ و مختص راه‌اندازی، هنگامی که فقط به بخشی از سطح راه‌اندازی نیاز دارید، درگاه‌های محدود کمک‌ابزار راه‌اندازی را به چتر گسترده‌تر `plugin-sdk/setup` ترجیح دهید:

| مسیر درون‌ریزی                | کاربرد                                                                                | خروجی‌های کلیدی                                                                                                                                                                                                                                                                                                           |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | کمک‌ابزارهای زمان اجرای هنگام راه‌اندازی که در `setupEntry` / راه‌اندازی به‌تعویق‌افتادهٔ کانال در دسترس می‌مانند | `createSetupTranslator`، `createPatchedAccountSetupAdapter`، `createEnvPatchedAccountSetupAdapter`، `createSetupInputPresenceValidator`، `noteChannelLookupFailure`، `noteChannelLookupSummary`، `promptResolvedAllowFrom`، `splitSetupEntries`، `createAllowlistSetupWizardProxy`، `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | کمک‌ابزارهای CLI/بایگانی/مستندات برای راه‌اندازی/نصب                                                    | `formatCliCommand`، `detectBinary`، `extractArchive`، `resolveBrewExecutable`، `formatDocsLink`، `CONFIG_DIR`                                                                                                                                                                                                         |

هنگامی که مجموعه‌ابزار مشترک کامل راه‌اندازی، از جمله کمک‌ابزارهای وصلهٔ پیکربندی مانند `moveSingleAccountChannelSectionToDefaultAccount(...)`، را می‌خواهید از درگاه گسترده‌تر `plugin-sdk/setup` استفاده کنید.

برای متن ثابت جادوگر راه‌اندازی از `createSetupTranslator(...)` استفاده کنید. این بخش به‌ترتیب از نخستین مقدار غیرخالی میان `OPENCLAW_LOCALE`، `LC_ALL`، `LC_MESSAGES` و `LANG` استفاده می‌کند و سپس به انگلیسی برمی‌گردد. برای بازنویسی صریح انگلیسی، `OPENCLAW_LOCALE=en` را تنظیم کنید. متن راه‌اندازی ویژهٔ Plugin را در کد متعلق به Plugin نگه دارید و فقط برای برچسب‌های مشترک راه‌اندازی، متن وضعیت و متن راه‌اندازی Pluginهای رسمی همراه محصول از کلیدهای کاتالوگ مشترک استفاده کنید.

آداپتورهای وصلهٔ راه‌اندازی هنگام درون‌ریزی برای مسیر داغ ایمن باقی می‌مانند. جست‌وجوی سطح قرارداد ارتقای حساب منفرد همراه محصول آن‌ها تنبل است؛ بنابراین درون‌ریزی `plugin-sdk/setup-runtime` پیش از استفادهٔ واقعی از آداپتور، کشف سطح قرارداد همراه محصول را مشتاقانه بارگذاری نمی‌کند.

### فیلدهای ورودی راه‌اندازی متعلق به کانال

`ChannelSetupInput` یک پوشش عمومی مشترک میان فراخواننده‌های راه‌اندازی و Pluginهای
کانال است. فیلدهای دارای نوع دائمی آن عبارت‌اند از `name`، `token`، `tokenFile`،
`useEnv`، `allowFrom` و `defaultTo`. کلیدهای اضافی متعلق به Plugin همچنان می‌توانند
در شیء ورودی زمان اجرا وجود داشته باشند، اما نوع مشترک هیچ امضای
شاخصی را اعلام نمی‌کند. هر Plugin باید فیلدهای راه‌اندازی خود را اعلام و محدود کند یا
آن‌ها را با یک طرح‌وارهٔ متعلق به Plugin در مرز آداپتور اعتبارسنجی کند:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

فیلدهای مختص کانال که پیش‌تر مستقیماً روی
`ChannelSetupInput` تعریف شده بودند، برای سازگاری با کد منبع خارجی موقتاً دارای نوع باقی می‌مانند.
آن‌ها منسوخ شده‌اند. پیمایش رجیستری در 2026-07-22 روی 426
Plugin کانال منتشرشده خارج از درخت، 21 فیلد بدون خواننده را حذف کرد و 22 فیلد دارای
خواننده‌های شناخته‌شده را نگه داشت. هر فیلد نگه‌داشته‌شده به‌محض اینکه هیچ Plugin منتشرشده‌ای آن را نخواند، حذف می‌شود؛
هیچ مرز نسخه‌ای لازم نیست. Pluginهای جدید و همراه نباید به این
لایه متکی باشند؛ فیلدهای تحت مالکیت خود را به‌صورت محلی تعریف کنید.

### ارتقای تک‌حساب تحت مالکیت کانال

هنگامی که یک کانال از پیکربندی سطح‌بالای تک‌حساب به `channels.<id>.accounts.*` ارتقا می‌یابد، رفتار مشترک پیش‌فرض، مقادیر ارتقایافته در محدوده حساب را به `accounts.default` منتقل می‌کند.

هر Plugin کانال می‌تواند این ارتقا را از طریق آداپتور راه‌اندازی خود گسترش دهد یا محدودتر کند:

- `singleAccountKeysToMove`: کلیدهای سطح‌بالای اضافی که باید به حساب ارتقایافته منتقل شوند
- `namedAccountPromotionKeys`: وقتی حساب‌های نام‌گذاری‌شده از قبل وجود دارند، فقط این کلیدها به حساب ارتقایافته منتقل می‌شوند؛ کلیدهای مشترک سیاست/تحویل در ریشه کانال باقی می‌مانند
- `resolveSingleAccountPromotionTarget(...)`: انتخاب کنید کدام حساب موجود مقادیر ارتقایافته را دریافت کند

وجود `singleAccountKeysToMove` نشان می‌دهد قرارداد ارتقا کامل است. حتی وقتی این فیلد یک آرایه خالی است، آن را تعریف کنید تا از ارتقای کلیدهای قدیمی انصراف دهید. آداپتورهایی که این فیلد را حذف می‌کنند، برای Pluginهای ازپیش‌منتشرشده یک لایه ارتقای پیش از تعریف با پشتوانه خواننده را حفظ می‌کنند. پیمایش رجیستری در 2026-07-22، تعداد 23 کلید بدون وابسته منتشرشده را حذف کرد و شش کلید رایج به‌همراه کلید صرفاً مخصوص راه‌اندازی `rooms` را نگه داشت. هر کلید نگه‌داشته‌شده به‌محض اینکه خواننده‌های منتشرشده آن به تعریف‌ها مهاجرت کنند، حذف می‌شود؛ هیچ مرز نسخه‌ای لازم نیست.

هنگامی که doctor باید این تعریف‌ها را از آرتیفکت سبک راه‌اندازی همراه بارگذاری کند، `openclaw.setupFeatures.configPromotion: true` را در مانیفست بسته Plugin تعریف کنید. سطح Plugin صرفاً مخصوص راه‌اندازی و Plugin کامل کانال باید تعریف‌های یکسانی ارائه دهند.

هنگام فراخوانی `moveSingleAccountChannelSectionToDefaultAccount(...)` با یک Plugin ازپیش‌حل‌شده، آداپتور راه‌اندازی آن را به‌عنوان `setupSurface` ارسال کنید. سطوح راه‌اندازی ارائه‌شده توسط فراخواننده بر جست‌وجوی بارگذاری‌شده و همراه اولویت دارند؛ در نتیجه Pluginهای محدوده‌بندی‌شده یا صرفاً مخصوص راه‌اندازی از ثبت سراسری مستقل می‌مانند.

<Note>
Matrix نمونه همراه فعلی است. اگر دقیقاً یک حساب نام‌گذاری‌شده Matrix از قبل وجود داشته باشد، یا اگر `defaultAccount` به یک کلید غیرمتعارف موجود مانند `Ops` اشاره کند، ارتقا به‌جای ایجاد ورودی جدید `accounts.default` همان حساب را حفظ می‌کند.
</Note>

## شِمای پیکربندی

پیکربندی Plugin در برابر JSON Schema موجود در مانیفست اعتبارسنجی می‌شود. کاربران Pluginها را از این طریق پیکربندی می‌کنند:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Plugin شما هنگام ثبت، این پیکربندی را به‌صورت `api.pluginConfig` دریافت می‌کند.

برای پیکربندی مختص کانال، در عوض از بخش پیکربندی کانال استفاده کنید:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### ساخت شِماهای پیکربندی کانال

برای تبدیل یک شِمای Zod به پوشش `ChannelConfigSchema` که آرتیفکت‌های پیکربندی تحت مالکیت Plugin از آن استفاده می‌کنند، از `buildChannelConfigSchema` استفاده کنید:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

اگر قرارداد را از قبل به‌صورت JSON Schema یا TypeBox می‌نویسید، از تابع کمکی مستقیم استفاده کنید تا OpenClaw بتواند در مسیرهای فراداده از تبدیل Zod به JSON Schema صرف‌نظر کند:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

برای Pluginهای شخص ثالث، قرارداد مسیر سرد همچنان مانیفست Plugin است: JSON Schema تولیدشده را در `openclaw.plugin.json#channelConfigs` بازتاب دهید تا سطوح شِمای پیکربندی، راه‌اندازی و رابط کاربری بتوانند بدون بارگذاری کد زمان اجرا، `channels.<id>` را بررسی کنند.

## جادوگرهای راه‌اندازی

Pluginهای کانال می‌توانند جادوگرهای راه‌اندازی تعاملی برای `openclaw onboard` ارائه دهند. جادوگر یک شیء `ChannelSetupWizard` روی `ChannelPlugin` است:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard` همچنین از `textInputs`، `dmPolicy`، `allowFrom`، `groupAccess`، `prepare`، `finalize` و موارد دیگر پشتیبانی می‌کند. برای مشاهده یک نمونه کامل همراه، به `src/setup-core.ts` در Plugin مربوط به Discord مراجعه کنید.

<AccordionGroup>
  <Accordion title="پرسش‌های مشترک allowFrom">
    برای پرسش‌های فهرست مجاز DM که فقط به جریان استاندارد `note -> prompt -> parse -> merge -> patch` نیاز دارند، توابع کمکی راه‌اندازی مشترک از `openclaw/plugin-sdk/setup`، یعنی `createPromptParsedAllowFromForAccount(...)` و `createTopLevelChannelParsedAllowFromPrompt(...)`، را ترجیح دهید.
  </Accordion>
  <Accordion title="وضعیت استاندارد راه‌اندازی کانال">
    برای بلوک‌های وضعیت راه‌اندازی کانال که فقط از نظر برچسب‌ها، امتیازها و خطوط اضافی اختیاری تفاوت دارند، به‌جای ساخت دستی همان شیء `status` در هر Plugin، `createStandardChannelSetupStatus(...)` از `openclaw/plugin-sdk/setup` را ترجیح دهید.
  </Accordion>
  <Accordion title="سطح اختیاری راه‌اندازی کانال">
    برای سطوح راه‌اندازی اختیاری که باید فقط در زمینه‌های مشخصی ظاهر شوند، از `createOptionalChannelSetupSurface` در `openclaw/plugin-sdk/channel-setup` استفاده کنید:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    هنگامی که فقط به یکی از دو بخش آن سطح نصب اختیاری نیاز دارید، `plugin-sdk/channel-setup` همچنین سازنده‌های سطح‌پایین‌تر `createOptionalChannelSetupAdapter(...)` و `createOptionalChannelSetupWizard(...)` را ارائه می‌دهد.

    آداپتور/جادوگر اختیاری تولیدشده در نوشتن واقعی پیکربندی به‌صورت بسته و امن شکست می‌خورند. آن‌ها یک پیام واحدِ نیاز به نصب را در `validateInput`، `applyAccountConfig` و `finalize` دوباره استفاده می‌کنند و وقتی `docsPath` تنظیم شده باشد، یک پیوند مستندات می‌افزایند.

  </Accordion>
  <Accordion title="توابع کمکی راه‌اندازی متکی بر فایل اجرایی">
    برای رابط‌های کاربری راه‌اندازی متکی بر فایل اجرایی، به‌جای کپی‌کردن همان اتصال فایل اجرایی/وضعیت در هر کانال، توابع کمکی واگذارشده مشترک را ترجیح دهید:

    - `createDetectedBinaryStatus(...)` برای بلوک‌های وضعیتی که فقط از نظر برچسب‌ها، راهنمایی‌ها، امتیازها و تشخیص فایل اجرایی تفاوت دارند
    - `createCliPathTextInput(...)` برای ورودی‌های متنی متکی بر مسیر
    - `createDelegatedSetupWizardProxy(...)` هنگامی که `setupEntry` باید رفتار وضعیت، آماده‌سازی یا نهایی‌سازی را با تأخیر به یک جادوگر کامل و سنگین‌تر واگذار کند
    - `createDelegatedTextInputShouldPrompt(...)` هنگامی که `setupEntry` فقط باید یک تصمیم `textInputs[*].shouldPrompt` را واگذار کند

  </Accordion>
</AccordionGroup>

## انتشار و نصب

**Pluginهای خارجی:** در [ClawHub](/clawhub) منتشر کنید، سپس نصب کنید:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    هنگام گذار راه‌اندازی، مشخصه‌های ساده بسته از npm نصب می‌شوند، مگر اینکه نام با شناسه یک Plugin همراه یا رسمی مطابقت داشته باشد؛ در این صورت OpenClaw به‌جای آن از نسخه محلی/رسمی استفاده می‌کند. برای انتخاب قطعی منبع از `clawhub:`، `npm:`، `git:` یا `npm-pack:` استفاده کنید — به [مدیریت Pluginها](/fa/plugins/manage-plugins) مراجعه کنید.

  </Tab>
  <Tab title="فقط ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="مشخصه بسته npm">
    وقتی بسته‌ای هنوز به ClawHub منتقل نشده است، یا هنگامی که در طول مهاجرت به یک
    مسیر نصب مستقیم npm نیاز دارید، از npm استفاده کنید:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**Pluginهای داخل مخزن:** آن‌ها را زیر درخت فضای کاری Pluginهای همراه قرار دهید؛ در طول ساخت به‌طور خودکار شناسایی می‌شوند.

<Info>
برای نصب‌هایی که منبع آن‌ها npm است، `openclaw plugins install` بسته را در یک پروژه مختص هر Plugin زیر `~/.openclaw/npm/projects` و با اسکریپت‌های چرخه حیات غیرفعال (`--ignore-scripts`) نصب می‌کند. درخت وابستگی Plugin را کاملاً JS/TS نگه دارید و از بسته‌هایی که به ساخت‌های `postinstall` نیاز دارند، اجتناب کنید.
</Info>

<Note>
راه‌اندازی Gateway وابستگی‌های Plugin را نصب نمی‌کند. جریان‌های نصب npm/git/ClawHub مسئول همگرایی وابستگی‌ها هستند؛ وابستگی‌های Pluginهای محلی باید از قبل نصب شده باشند.
</Note>

فراداده بسته همراه صریح است و هنگام راه‌اندازی Gateway از JavaScript ساخته‌شده استنباط نمی‌شود. وابستگی‌های زمان اجرا به بسته Plugin مالک آن‌ها تعلق دارند؛ راه‌اندازی OpenClaw بسته‌بندی‌شده هرگز وابستگی‌های Plugin را ترمیم یا بازتاب نمی‌دهد.

## مرتبط

- [ساخت Pluginها](/fa/plugins/building-plugins) — راهنمای گام‌به‌گام شروع کار
- [مانیفست Plugin](/fa/plugins/manifest) — مرجع کامل شِمای مانیفست
- [نقاط ورود SDK](/fa/plugins/sdk-entrypoints) — `definePluginEntry` و `defineChannelPluginEntry`
