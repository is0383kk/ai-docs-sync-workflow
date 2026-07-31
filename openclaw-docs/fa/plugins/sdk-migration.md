---
read_when:
    - هشدار OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED را مشاهده می‌کنید
    - هشدار OPENCLAW_EXTENSION_API_DEPRECATED را مشاهده می‌کنید
    - پیش از OpenClaw 2026.4.25 از api.registerEmbeddedExtensionFactory استفاده می‌کردید
    - در حال به‌روزرسانی یک Plugin به معماری مدرن Plugin هستید
    - شما یک Plugin خارجی OpenClaw را نگهداری می‌کنید
sidebarTitle: Migrate to SDK
summary: از لایهٔ قدیمی سازگاری با نسخه‌های پیشین به SDK مدرن Plugin مهاجرت کنید
title: مهاجرت SDK افزونه
x-i18n:
    generated_at: "2026-07-27T14:26:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw یک لایهٔ گستردهٔ سازگاری با نسخه‌های پیشین را با معماری مدرن Plugin
که از importهای کوچک و متمرکز ساخته شده است جایگزین کرد. اگر Plugin شما به پیش از آن
تغییر مربوط است، این راهنما آن را با قراردادهای کنونی منطبق می‌کند.

## چه چیزهایی تغییر کرد

پیش‌تر چندین سطح import بسیار باز به Pluginها اجازه می‌دادند تقریباً به هر چیزی
از یک نقطهٔ ورود واحد دسترسی پیدا کنند:

- **`openclaw/plugin-sdk`** و **`openclaw/plugin-sdk/compat`** - در مدتی که SDK متمرکز ساخته می‌شد،
  ده‌ها ابزار کمکی را دوباره export می‌کردند. اکنون هر دو ریشه
  حذف شده‌اند؛ به‌جای آن، یک زیرمسیر مستندشده را import کنید.
- **`openclaw/plugin-sdk/infra-runtime`** - یک barrel گسترده که رویدادهای سیستم،
  وضعیت Heartbeat، صف‌های تحویل، ابزارهای کمکی fetch/proxy، ابزارهای کمکی فایل،
  انواع تأیید و ابزارهای نامرتبط را درهم می‌آمیخت.
- **`openclaw/plugin-sdk/config-runtime`** - یک barrel گستردهٔ پیکربندی که
  فقط برای بازهٔ سازگاری بعدی خود حفظ شده بود؛ ابزارهای کمکی مستقیم بارگذاری/نوشتن در زمان اجرا
  حذف شده‌اند.
- **`openclaw/extension-api`** - یک پل حذف‌شده که به Pluginها دسترسی مستقیم
  به ابزارهای کمکی سمت میزبان، مانند اجراکنندهٔ عامل تعبیه‌شده، می‌داد.
- **`api.registerEmbeddedExtensionFactory(...)`** - یک hook حذف‌شدهٔ مختص اجراکنندهٔ تعبیه‌شده
  که رویدادهای اجراکنندهٔ تعبیه‌شده مانند `tool_result` را مشاهده می‌کرد. به‌جای آن از میان‌افزار
  نتیجهٔ ابزار عامل استفاده کنید (نگاه کنید به [انتقال افزونه‌های نتیجهٔ ابزار تعبیه‌شده
  به میان‌افزار](#how-to-migrate)).

SDK ریشه، barrel سازگاری، پل افزونه و کارخانهٔ افزونهٔ تعبیه‌شده
حذف شده‌اند. `infra-runtime` و `config-runtime` فقط برای بازه‌های بعدی
که جداگانه ثبت شده‌اند باقی می‌مانند؛ Pluginهای جدید باید از زیرمسیرهای متمرکز استفاده کنند.

<Warning>
  Pluginهایی که سطوح حذف‌شدهٔ ریشه، سازگاری یا افزونه را import می‌کنند دیگر
  بارگذاری نمی‌شوند. پیش از ارتقا، نگاشت‌های زیر را دنبال کنید.
</Warning>

OpenClaw رفتار مستندشدهٔ Plugin را هم‌زمان با
معرفی جایگزین حذف یا بازتفسیر نمی‌کند. تغییرات شکنندهٔ قرارداد ابتدا از
آداپتور سازگاری، عیب‌یابی، مستندات و یک بازهٔ منسوخ‌سازی عبور می‌کنند. این موضوع
برای importهای SDK، فیلدهای manifest، APIهای راه‌اندازی، hookها و رفتار
ثبت در زمان اجرا صدق می‌کند.

### چرا

- **راه‌اندازی کند** - import کردن یک ابزار کمکی، ده‌ها ماژول نامرتبط را بارگذاری می‌کرد.
- **وابستگی‌های چرخه‌ای** - exportهای مجدد گسترده، ایجاد چرخه‌های import را آسان
  می‌کردند.
- **سطح API نامشخص** - راهی برای تشخیص exportهای پایدار از موارد داخلی وجود نداشت.

اکنون هر `openclaw/plugin-sdk/<subpath>` یک ماژول کوچک و مستقل با
قراردادی مستندشده است.

مسیرهای تسهیل‌کنندهٔ قدیمی ارائه‌دهندگان برای کانال‌های همراه نیز حذف شده‌اند -
میان‌برهای ابزار کمکی با نام تجاری کانال، تسهیلات خصوصی مونو‌ریپو بودند، نه
قراردادهای پایدار Plugin. به‌جای آن از زیرمسیرهای عمومی و محدود SDK استفاده کنید. درون
فضای کاری Pluginهای همراه، ابزارهای کمکی متعلق به ارائه‌دهنده را در
`api.ts` یا `runtime-api.ts` همان Plugin نگه دارید:

- Anthropic ابزارهای کمکی جریان مختص Claude را در مسیر اختصاصی `api.ts` /
  `contract-api.ts` خود نگه می‌دارد.
- OpenAI سازنده‌های ارائه‌دهنده، ابزارهای کمکی مدل پیش‌فرض و سازنده‌های ارائه‌دهندهٔ
  بلادرنگ را در `api.ts` اختصاصی خود نگه می‌دارد.
- OpenRouter سازندهٔ ارائه‌دهنده و ابزارهای کمکی ورود اولیه/پیکربندی را در
  `api.ts` اختصاصی خود نگه می‌دارد.

## خط‌مشی سازگاری

کار سازگاری Pluginهای خارجی این ترتیب را دنبال می‌کند:

1. قرارداد جدید را اضافه کنید.
2. رفتار قدیمی را از طریق یک آداپتور سازگاری متصل نگه دارید.
3. یک پیام عیب‌یابی یا هشدار منتشر کنید که مسیر قدیمی و جایگزین آن را نام می‌برد.
4. هر دو مسیر را در آزمون‌ها پوشش دهید.
5. منسوخ‌سازی و مسیر مهاجرت را مستند کنید.
6. فقط پس از پایان بازهٔ مهاجرت اعلام‌شده، معمولاً در یک انتشار
   اصلی، آن را حذف کنید.

اگر یک فیلد manifest همچنان پذیرفته می‌شود، تا زمانی که مستندات و
پیام‌های عیب‌یابی خلاف آن را اعلام نکرده‌اند، به استفاده از آن ادامه دهید. کد جدید باید جایگزین مستندشده را ترجیح دهد؛
Pluginهای موجود نباید طی انتشارهای فرعی معمولی از کار بیفتند.

### سازگاری راه‌اندازی کانال‌های منتشرشده

بسته‌های Slack، Discord، Signal و Microsoft Teams که از طریق
`2026.7.1` منتشر شده‌اند، schemaهای پیکربندی مختص کانال را از
`openclaw/plugin-sdk/bundled-channel-config-schema` import می‌کنند. بسته‌های منتشرشدهٔ Slack و
Discord همچنین `createLegacyCompatChannelDmPolicy` و
`promptLegacyChannelAllowFromForAccount` را از
`openclaw/plugin-sdk/setup-runtime` import می‌کنند.

این exportها به‌عنوان آداپتورهای منسوخ‌شدهٔ سازگاری زمان اجرا در دسترس باقی می‌مانند.
Pluginهای جدید و بازنشرشده باید schemaهای پیکربندی و خط‌مشی راه‌اندازی خود را
به‌صورت محلی در اختیار داشته باشند و از اجزای عمومی `channel-config-schema` و
`setup-runtime` استفاده کنند. exportهای سازگاری فقط زمانی قابل حذف‌اند که
حداقل نسخه‌های پشتیبانی‌شدهٔ بسته‌های منتشرشده دیگر آن‌ها را import نکنند.

### سازگاری فیلدهای ورودی راه‌اندازی کانال

`ChannelSetupInput` اکنون فقط پوشش راه‌اندازی مشترک میان کانال‌ها را به‌طور
دائمی دارای نوع نگه می‌دارد. فیلدهای مختص کانال در یک سطح سازگاری منسوخ‌شده
همچنان دارای نوع باقی می‌مانند تا Pluginهای خارجی موجود در مدتی که نویسندگان Plugin آن
فیلدها را به انواع ورودی راه‌اندازی محلی Plugin منتقل می‌کنند، همچنان کامپایل شوند.

OpenClaw انتشار اصلی ارائه نمی‌کند. یک پیمایش registry در 2026-07-22،
426 Plugin کانال منتشرشدهٔ خارج از درخت را بررسی و 21 فیلد بدون خواننده را حذف کرد.
هر یک از 22 فیلد حفظ‌شده، یک خوانندهٔ منتشرشدهٔ شناخته‌شده دارد. هر فیلد بعدی
به‌محض اینکه هیچ Plugin منتشرشده‌ای آن را نخواند حذف می‌شود؛ مجموعهٔ حفظ‌شده با
مهاجرت نویسندگان Plugin به انواع ورودی راه‌اندازی محلی Plugin کوچک‌تر می‌شود.

همان پیمایش، 23 کلید قدیمی ارتقای آداپتور اعلام‌نشده را که وابستهٔ
منتشرشده‌ای نداشتند حذف کرد. شش کلید رایج و کلید مختص راه‌اندازی `rooms` باقی مانده‌اند.
این مجموعه نیز با اعلام `singleAccountKeysToMove` توسط Pluginهای منتشرشده کوچک‌تر می‌شود.

نوع مشترک هیچ index signature ندارد. کلیدهای متعلق به Plugin همچنان می‌توانند
در اشیای ورودی زمان اجرا وجود داشته باشند؛ آن‌ها را در یک intersection محلی Plugin اعلام کنید یا
از طریق schema راه‌اندازی Plugin مالک محدودشان کنید.

| `code`                                  | `owner`   | `replacement`                                                                                    | شرط حذف                                                     |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | `ChannelSetupInput` را با یک نوع محلی Plugin که فیلدهای کانال مالک را اعلام می‌کند intersect کنید | وقتی پیمایش registry Pluginهای منتشرشده هیچ خواننده‌ای ندارد، فیلد را حذف کنید |

سطح قدیمی ارتقای آداپتور اعلام‌نشده نیز از همان خط‌مشی
مبتنی بر خواننده پیروی می‌کند. `singleAccountKeysToMove` را اعلام کنید، از جمله یک آرایهٔ خالی زمانی که
Plugin به کلیدهای ارتقای اضافی نیاز ندارد، تا fallback مشترک بتواند هر بار یک
کلید را بازنشسته کند.

#### تأیید خواننده‌ها

1. با هر `nextCursor` در `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` صفحه‌به‌صفحه پیش بروید و بسته‌هایی را نگه دارید که `categories` آن‌ها شامل `channels` است.
2. گزینه‌های npm را از `npm search --json --searchlimit=1000 "openclaw channel plugin"` اضافه کنید. گزینه‌های صرفاً منبع را از جست‌وجوهای کد GitHub برای `openclaw/plugin-sdk/channel-setup`، `openclaw/plugin-sdk/setup` و `openclaw/plugin-sdk/core` اضافه کنید.
3. آخرین نسخهٔ منتشرشدهٔ هر گزینه را تعیین کنید. `npm pack <package>@<version> --json --pack-destination <temp-dir>` را اجرا و آن را باز کنید، سپس JavaScript و declarationهای عرضه‌شدهٔ `dist` را برای خواندن مستقیم یا destructured فیلد بررسی کنید. وقتی بسته‌ای انتشار npm ندارد، artifact مربوط به ClawHub را دانلود کنید.
4. بسته، نسخه، فیلد یا کلید ارتقا و فایل منطبق را ثبت کنید. یک فیلد یا کلید فقط زمانی قابل حذف است که هیچ artifact منتشرشدهٔ Plugin آن را نخواند. نام خواننده‌ها را در توضیحات کد کنار فهرست فیلدها و کلیدهای حفظ‌شده، همگام با پیمایش نگه دارید.

این فقط یک سابقهٔ سازگاری منبع/نوع است. هیچ آداپتور زمان اجرا یا
ورودی registry سازگاری ندارد، زیرا اشیای ورودی راه‌اندازی زمان اجرا و رفتار
راه‌اندازی تغییری نکرده‌اند.

صف مهاجرت کنونی را با `pnpm plugins:boundary-report` ممیزی کنید:

| پرچم                                                    | اثر                                                                         |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (یا `pnpm plugins:boundary-report:summary`) | شمارش‌های فشرده به‌جای جزئیات کامل.                                         |
| `--json`                                                | گزارش قابل‌خواندن توسط ماشین.                                                       |
| `--owner <id>`                                          | محدود کردن به یک Plugin یا مالک سازگاری.                                   |
| `--fail-on-cross-owner`                                 | خروج با کد غیرصفر برای importهای رزروشدهٔ SDK میان مالکان.                             |
| `--fail-on-eligible-compat`                             | خروج با کد غیرصفر هنگامی که تاریخ `removeAfter` یک رکورد سازگاری منسوخ‌شده گذشته باشد. |
| `--fail-on-unclassified-unused-reserved`                | خروج با کد غیرصفر برای shimهای رزروشده و استفاده‌نشدهٔ SDK.                                    |

`pnpm plugins:boundary-report:ci` با هر سه پرچم شکست اجرا می‌شود. رکوردهای
منسوخ‌شده معمولاً به‌جای عبارت مبهم «انتشار اصلی بعدی»، تاریخ صریح `removeAfter` دارند.
رکوردی که مالک آن تاریخی را تأیید نکرده است،
`removeAfter` ندارد، به‌شکل `no-date` نمایش داده می‌شود و هرگز واجد شرایط حذف نیست.
گزارش، رکوردهای منسوخ‌شده را براساس تاریخ گروه‌بندی می‌کند، ارجاعات محلی کد/مستندات را می‌شمارد،
importهای رزروشدهٔ SDK میان مالکان را نشان می‌دهد و پل خصوصی SDK
میزبان حافظه را خلاصه می‌کند. زیرمسیرهای رزروشدهٔ SDK باید استفادهٔ ردیابی‌شده توسط مالک داشته باشند؛
exportهای رزروشدهٔ بدون استفاده باید از SDK عمومی حذف شوند.

### نگاشت قدیمی رسانه

رکورد سازگاری `media-legacy-projection` فیلدهای موازی قدیمی
رسانه، سازنده‌های payload، نام‌های مستعار metadata مربوط به hook و نام‌های template
رسانه را پوشش می‌دهد. تاریخ تأییدشدهٔ `removeAfter` آن **2026-10-01** است (دو چرخهٔ انتشار
پس از عرضهٔ جایگزین‌های facts-first). حذف همچنین مستلزم
یک پیمایش پاک از artifactهای Plugin منتشرشده در آن زمان است؛ پیش از تاریخ مهاجرت کنید.

برای ورودی کانال، `MediaPath`، `MediaUrl`،
`MediaType`، `MediaPaths`، `MediaUrls`، `MediaTypes`،
`MediaTranscribedIndexes`، `MediaWorkspaceDir` و `MediaStaged` مفرد/جمع را با
facts مرتب‌شده جایگزین کنید:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

از `event.media` در hookهای `inbound_claim` و `message_received` استفاده کنید. اگر رسانهٔ
راه‌دور به‌صورت محلی stage نشده است، از `event.originalMedia` برای هویت/عیب‌یابی
استفاده کنید و منتظر `event.media` بمانید؛ `event.mediaStagingPending` آن
وضعیت را متمایز می‌کند. ویژگی‌های منسوخ‌شدهٔ مفرد/جمع را از
`event.metadata` نخوانید.

برای مدل‌های رسانهٔ CLI، `{{MediaPath}}`، `{{MediaUrl}}`، `{{MediaType}}`
و `{{MediaDir}}` را با `{{AttachmentPath}}`، `{{AttachmentUrl}}`،
`{{AttachmentContentType}}` و `{{AttachmentDir}}` جایگزین کنید. هنگامی که
موقعیت پیوست اهمیت دارد، از `{{AttachmentIndex}}` استفاده کنید.

برای خط‌مشی خواندن رسانهٔ محلی، `getAgentScopedMediaLocalRoots(...)` یا
`getAgentScopedMediaLocalRootsForSources(...)` را از
`openclaw/plugin-sdk/media-local-roots` import کنید. facade مربوط به
`openclaw/plugin-sdk/agent-media-payload` و نگاشت
`buildAgentMediaPayload(...)` آن منسوخ شده‌اند.

## نحوهٔ مهاجرت

<Steps>
  <Step title="مهاجرت ابزارهای کمکی بارگذاری/نوشتن پیکربندی زمان اجرا">
    Pluginهای همراه باید فراخوانی مستقیم `api.runtime.config.loadConfig()` و
    `api.runtime.config.writeConfigFile(...)` را متوقف کنند. پیکربندی‌ای را ترجیح دهید که از قبل
    به مسیر فراخوانی فعال ارسال شده است. handlerهای طولانی‌عمر که به snapshot
    فرایند کنونی نیاز دارند می‌توانند از `api.runtime.config.current()` استفاده کنند. ابزارهای عامل
    طولانی‌عمر باید `ctx.getRuntimeConfig()` را درون `execute` بخوانند تا ابزاری
    که پیش از نوشتن پیکربندی ایجاد شده است، همچنان پیکربندی تازه‌سازی‌شده را ببیند.

    نوشتن پیکربندی از طریق ابزار کمکی تراکنشی با خط‌مشی صریح
    پس از نوشتن انجام می‌شود:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    از `afterWrite: { mode: "restart", reason: "..." }` زمانی استفاده کنید که تغییر به
    راه‌اندازی مجدد تمیز Gateway نیاز دارد، و از `afterWrite: { mode: "none", reason: "..." }`
    فقط زمانی استفاده کنید که فراخواننده مالک پیگیری بعدی است و عمداً
    برنامه‌ریز بارگذاری مجدد را غیرفعال می‌کند. نتایج جهش شامل یک خلاصهٔ نوع‌دار `followUp` برای
    آزمون‌ها و ثبت گزارش هستند؛ Gateway همچنان مسئول اعمال یا
    زمان‌بندی راه‌اندازی مجدد است.

    `loadConfig` و `writeConfigFile` از زمان‌اجرای Plugin
    حذف شده‌اند. Pluginهای همراه و کد زمان‌اجرای مخزن با
    `pnpm check:deprecated-api-usage` و
    `pnpm check:no-runtime-action-load-config` محافظت می‌شوند: استفادهٔ جدید در Plugin
    تولیدی مستقیماً شکست می‌خورد، نوشتن مستقیم پیکربندی شکست می‌خورد، متدهای سرور Gateway باید از
    تصویر لحظه‌ای زمان‌اجرای درخواست استفاده کنند، ابزارهای کمکی ارسال/کنش/کلاینت کانال در زمان‌اجرا
    باید پیکربندی را از مرز خود دریافت کنند، و ماژول‌های زمان‌اجرای
    بلندعمر اجازهٔ هیچ فراخوانی محیطی `loadConfig()` را ندارند.

    کد جدید Plugin باید از barrel گستردهٔ `openclaw/plugin-sdk/config-runtime`
    اجتناب کند. برای کار موردنظر از زیرمسیر محدود استفاده کنید:

    | نیاز | درون‌ریزی |
    | --- | --- |
    | نوع‌های پیکربندی مانند `OpenClawConfig` | `openclaw/plugin-sdk/config-contracts` |
    | جست‌وجوی پیکربندی در نقطهٔ ورود Plugin | `api.pluginConfig` |
    | ادغام پیکربندی | منطق محلی Plugin در مرز پیکربندی |
    | خواندن تصویر لحظه‌ای زمان‌اجرای فعلی | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | نوشتن پیکربندی | `openclaw/plugin-sdk/config-mutation` |
    | ابزارهای کمکی ذخیره‌گاه نشست | `openclaw/plugin-sdk/session-store-runtime` |
    | پیکربندی جدول Markdown | `openclaw/plugin-sdk/markdown-table-runtime` |
    | ابزارهای کمکی زمان‌اجرای خط‌مشی گروه | `openclaw/plugin-sdk/runtime-group-policy` |
    | تفکیک ورودی محرمانه | `openclaw/plugin-sdk/secret-input-runtime` |
    | بازنویسی‌های مدل/نشست | `openclaw/plugin-sdk/model-session-runtime` |

    Pluginهای همراه و آزمون‌هایشان در برابر barrel گسترده با اسکنر
    محافظت می‌شوند تا درون‌ریزی‌ها و mockها به رفتار موردنیازشان محدود بمانند.
    barrel همچنان برای سازگاری خارجی وجود دارد، اما کد جدید نباید
    به آن وابسته باشد.

  </Step>

  <Step title="انتقال افزونه‌های تعبیه‌شدهٔ نتیجهٔ ابزار به میان‌افزار">
    Pluginهای همراه باید کنترل‌کننده‌های نتیجهٔ ابزار `api.registerEmbeddedExtensionFactory(...)` را که
    فقط برای اجراکنندهٔ تعبیه‌شده هستند، با میان‌افزار مستقل از زمان‌اجرا
    جایگزین کنند:

    ```typescript
    // ابزارهای زمان‌اجرای OpenClaw و ابزارهای پویای زمان‌اجرای Codex (نتیجه ممکن است
    // تبدیل شود). نتایج ابزارهای بومی Codex نیز برای مشاهده منتقل می‌شوند،
    // اما خروجی تبدیل‌شدهٔ آن‌ها هرگز به مدل نمی‌رسد: قرارداد hook مربوط به
    // PostToolUse در Codex نمی‌تواند پاسخ یک ابزار بومی را جایگزین کند.
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    هم‌زمان مانیفست Plugin را به‌روزرسانی کنید:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    Pluginهای نصب‌شده نیز می‌توانند میان‌افزار نتیجهٔ ابزار را ثبت کنند، مشروط بر اینکه صریحاً
    فعال شده باشند و هر زمان‌اجرای هدف در
    `contracts.agentToolResultMiddleware` اعلام شده باشد. ثبت میان‌افزار نصب‌شدهٔ
    اعلام‌نشده رد می‌شود.

  </Step>

  <Step title="انتقال کنترل‌کننده‌های بومی تأیید به واقعیت‌های قابلیت">
    Pluginهای کانال دارای قابلیت تأیید، رفتار بومی تأیید را از طریق
    `approvalCapability.nativeRuntime` به‌همراه رجیستری مشترک زمینهٔ زمان‌اجرا
    ارائه می‌کنند:

    - `approvalCapability.handler.loadRuntime(...)` را با
      `approvalCapability.nativeRuntime` جایگزین کنید.
    - احراز هویت/تحویل ویژهٔ تأیید را از سیم‌کشی قدیمی `plugin.auth` /
      `plugin.approvals` به `approvalCapability` منتقل کنید.
    - `ChannelPlugin.approvals` از قرارداد عمومی
      Plugin کانال حذف شده است؛ فیلدهای تحویل/بومی/رندر را به
      `approvalCapability` منتقل کنید.
    - `plugin.auth` فقط برای جریان‌های ورود/خروج کانال باقی می‌ماند؛ هسته دیگر
      hookهای احراز هویت تأیید را در آن نمی‌خواند.
    - اشیای زمان‌اجرای متعلق به کانال (کلاینت‌ها، توکن‌ها، برنامه‌های Bolt) را
      از طریق `openclaw/plugin-sdk/channel-runtime-context` ثبت کنید.
    - از کنترل‌کننده‌های بومی تأیید، اعلان‌های تغییر مسیر متعلق به Plugin ارسال نکنید؛
      هسته مالک اعلان‌های مسیریابی‌شده به مقصدی دیگر بر اساس نتایج واقعی تحویل است.
    - هنگام ارسال `channelRuntime` به `createChannelManager(...)`، یک
      سطح واقعی `createPluginRuntime().channel` ارائه کنید؛ stubهای ناقص
      رد می‌شوند.

    برای چیدمان فعلی قابلیت تأیید، [Pluginهای کانال](/fa/plugins/sdk-channel-plugins) را ببینید.

  </Step>

  <Step title="ممیزی رفتار fallback پوشش‌دهندهٔ Windows">
    اگر Plugin شما از `openclaw/plugin-sdk/windows-spawn` استفاده می‌کند، پوشش‌دهنده‌های Windows
    `.cmd`/`.bat` که تفکیک نمی‌شوند، اکنون بسته شکست می‌خورند؛ مگر اینکه صریحاً
    `allowShellFallback: true` را ارسال کنید:

    ```typescript
    // پیش از تغییر
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // پس از تغییر
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // این مقدار را فقط برای فراخواننده‌های سازگاری مورداعتماد تنظیم کنید که عمداً
      // fallback با واسطهٔ shell را می‌پذیرند.
      allowShellFallback: true,
    });
    ```

    اگر فراخوانندهٔ شما عمداً به fallback پوسته متکی نیست،
    `allowShellFallback` را تنظیم نکنید و در عوض خطای پرتاب‌شده را مدیریت کنید.

  </Step>

  <Step title="یافتن درون‌ریزی‌های منسوخ">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="جایگزینی با درون‌ریزی‌های متمرکز">
    هر export از سطح قدیمی به یک مسیر درون‌ریزی مدرن و مشخص نگاشت می‌شود:

    ```typescript
    // پیش از تغییر (لایهٔ منسوخ سازگاری با نسخه‌های پیشین)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // پس از تغییر (درون‌ریزی‌های مدرن و متمرکز)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    برای ابزارهای کمکی سمت میزبان، به‌جای درون‌ریزی مستقیم از زمان‌اجرای تزریق‌شدهٔ Plugin
    استفاده کنید:

    ```typescript
    // پیش از تغییر (پل منسوخ extension-api)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // پس از تغییر (زمان‌اجرای تزریق‌شده)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    همین الگو برای دیگر ابزارهای کمکی پل قدیمی نیز به‌کار می‌رود:

    | درون‌ریزی قدیمی | معادل مدرن |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | ابزارهای کمکی ذخیره‌گاه نشست | `api.runtime.agent.session.*` |

  </Step>

  <Step title="جایگزینی درون‌ریزی‌های گستردهٔ infra-runtime">
    `openclaw/plugin-sdk/infra-runtime` همچنان برای سازگاری خارجی وجود دارد،
    اما کد جدید باید سطح متمرکزی را درون‌ریزی کند که واقعاً
    به آن نیاز دارد:

    | نیاز | درون‌ریزی |
    | --- | --- |
    | ابزارهای کمکی صف رویداد سیستم | `openclaw/plugin-sdk/system-event-runtime` |
    | ابزارهای کمکی بیدارسازی، رویداد و مشاهده‌پذیری Heartbeat | `openclaw/plugin-sdk/heartbeat-runtime` |
    | تخلیهٔ صف تحویل‌های در انتظار | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | تله‌متری فعالیت کانال | `openclaw/plugin-sdk/channel-activity-runtime` |
    | حافظه‌های نهان حذف تکرار درون‌حافظه‌ای و متکی بر ذخیره‌گاه پایدار | `openclaw/plugin-sdk/dedupe-runtime` |
    | ابزارهای کمکی امن مسیر فایل محلی/رسانه | `openclaw/plugin-sdk/file-access-runtime` |
    | واکشی آگاه از dispatcher | `openclaw/plugin-sdk/runtime-fetch` |
    | ابزارهای کمکی واکشی از طریق پراکسی و محافظت‌شده | `openclaw/plugin-sdk/fetch-runtime` |
    | نوع‌های خط‌مشی dispatcher مربوط به SSRF | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | نوع‌های درخواست/تفکیک تأیید | `openclaw/plugin-sdk/approval-runtime` |
    | ابزارهای کمکی payload پاسخ تأیید و فرمان | `openclaw/plugin-sdk/approval-reply-runtime` |
    | ابزارهای کمکی قالب‌بندی خطا | `openclaw/plugin-sdk/error-runtime` |
    | انتظار برای آمادگی انتقال | `openclaw/plugin-sdk/transport-ready-runtime` |
    | ابزارهای کمکی امن توکن | `openclaw/plugin-sdk/secure-random-runtime` |
    | هم‌زمانی محدود وظایف ناهمگام | `openclaw/plugin-sdk/concurrency-runtime` |
    | assertionهای مقدار الزامی برای ناورداهای اثبات‌پذیر | `openclaw/plugin-sdk/expect-runtime` |
    | تبدیل اجباری عددی | `openclaw/plugin-sdk/number-runtime` |
    | قفل ناهمگام محلی فرایند | `openclaw/plugin-sdk/async-lock-runtime` |
    | قفل‌های فایل | `openclaw/plugin-sdk/file-lock` |

    Pluginهای همراه در برابر `infra-runtime` با اسکنر محافظت می‌شوند، بنابراین کد مخزن
    نمی‌تواند به barrel گسترده بازگردد.

  </Step>

  <Step title="انتقال ابزارهای کمکی مسیر کانال">
    کد جدید مسیر کانال از `openclaw/plugin-sdk/channel-route` استفاده می‌کند. نام‌های قدیمی‌تر
    کلید مسیر به‌عنوان aliasهای سازگاری باقی می‌مانند:

    | ابزار کمکی قدیمی | ابزار کمکی مدرن |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    ابزارهای کمکی مدرن مسیر، `{ channel, to, accountId, threadId }` را در تأییدهای بومی،
    جلوگیری از پاسخ، حذف تکرار ورودی، تحویل Cron و مسیریابی نشست
    به‌شکل سازگار نرمال‌سازی می‌کنند.

    استفادهٔ جدیدی از `ChannelMessagingAdapter.parseExplicitTarget` یا
    `resolveChannelRouteTargetWithParser(...)` از
    `plugin-sdk/channel-route` اضافه نکنید؛ این موارد منسوخ شده‌اند و فقط برای Pluginهای
    قدیمی‌تر باقی مانده‌اند. Pluginهای کانال جدید باید برای نرمال‌سازی شناسهٔ هدف
    و fallback هنگام نبود نتیجه در دایرکتوری از
    `messaging.targetResolver.resolveTarget(...)`،
    هنگامی که هسته زودهنگام به نوع همتا نیاز دارد از `messaging.inferTargetChatType(...)`،
    و برای هویت بومی ارائه‌دهندهٔ
    نشست و رشته از `messaging.resolveOutboundSessionRoute(...)` استفاده کنند.

  </Step>

  <Step title="ساخت و آزمون">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## مرجع مسیر درون‌ریزی

نگاشت export عمومی بسته، منبع حقیقت برای زیرمسیرهای قابل درون‌ریزی SDK
است. از راهنماهای موضوعی SDK که در [نمای کلی SDK](/fa/plugins/sdk-overview)
پیوند داده شده‌اند استفاده کنید و محدودترین زیرمسیر عمومی مستندشده را ترجیح دهید. فهرست کامپایلر در
`scripts/lib/plugin-sdk-entrypoints.json` همچنین شامل ورودی‌های خصوصی-محلی مورداستفاده
برای ساخت Pluginهای همراه است؛ وجود آن‌ها در آنجا به‌معنای export عمومی بسته نیست.

این جدول زیرمجموعهٔ رایج انتقال است، نه کل سطح SDK. فهرست نقطهٔ ورود
کامپایلر در `scripts/lib/plugin-sdk-entrypoints.json` قرار دارد؛
exportهای بسته از زیرمجموعهٔ عمومی تولید می‌شوند.

درزهای ابزار کمکی رزروشده برای Pluginهای همراه از نگاشت export عمومی SDK
بازنشسته شده‌اند، به‌جز facadeهای سازگاری صریحاً مستندشده مانند shim
منسوخ `plugin-sdk/discord` که برای Pluginهای خارجی نگه داشته شده است که هنوز
بستهٔ منتشرشدهٔ `@openclaw/discord` را مستقیماً درون‌ریزی می‌کنند. ابزارهای کمکی
ویژهٔ مالک درون بستهٔ Plugin مالک قرار دارند؛ رفتار مشترک میزبان
از طریق قراردادهای عمومی SDK مانند `plugin-sdk/gateway-runtime`،
`plugin-sdk/security-runtime` و API تزریق‌شدهٔ Plugin منتقل می‌شود.

از محدودترین درون‌ریزی متناسب با کار استفاده کنید. اگر exportی پیدا نمی‌کنید،
منبع را در `src/plugin-sdk/` بررسی کنید یا از نگه‌دارندگان بپرسید کدام قرارداد عمومی
باید مالک آن باشد.

## سطوح سازگاری حذف‌شده

پاک‌سازی ژوئیهٔ 2026، barrelهای SDK ریشه و سازگاری، پل API
افزونه، aliasهای منقضی‌شدهٔ زیرمسیر SDK، زیرمسیرهای بلااستفادهٔ SDK و exportهای عمومی
ماژول‌های SDK ویژهٔ Pluginهای همراه را حذف کرد. ماژول‌های ویژهٔ Pluginهای همراه از طریق
نگاشت‌های ساخت خصوصی-محلی برای مالکان مخزنشان در دسترس می‌مانند؛ این ماژول‌ها از
بستهٔ منتشرشده قابل درون‌ریزی نیستند.

### انتشار سراسری فرایندِ ارائه‌دهندهٔ API

`registerApiProvider(...)` و `unregisterApiProviders(...)` از
`openclaw/plugin-sdk/llm` حذف شدند. آن‌ها انتقال‌های API را در وضعیت سراسری
فرایند منتشر می‌کردند و سپس زمان‌اجراهای مدلِ دارای مالک چرخهٔ حیات مجبور بودند آن‌ها را در هر
رجیستری آماده‌شده کپی کنند.

Pluginهای ارائه‌دهنده باید ارائه‌دهندگان استنتاج متن را از طریق
`api.registerProvider(...)` ثبت کنند. کد و آزمون‌های متعلق به میزبان که یک
`ApiRegistry` می‌سازند باید مستقیماً در همان رجیستری ثبت کنند تا مالکیت
ارائه‌دهنده و پاک‌سازی در محدودهٔ زمان‌اجرای آماده‌شده باقی بماند.

### barrel خصوصی آزمون

`openclaw/plugin-sdk/testing` محلی مخزن بود و از مصنوعات بستهٔ منتشرشده
حذف می‌شد، بنابراین پیش از تاریخ `removeAfter` آن در 2026-07-28 حذف شد. آزمون‌های مخزن
از زیرمسیرهای متمرکزی مانند `plugin-sdk/plugin-test-runtime`،
`plugin-sdk/channel-test-helpers`، `plugin-sdk/channel-target-testing`،
`plugin-sdk/test-env` و `plugin-sdk/test-fixtures` استفاده می‌کنند.

## مرجع انتقال

  این نگاشت‌ها هم سطوح حذف‌شده در ژوئیهٔ 2026 و هم منسوخ‌سازی‌های فعال در بازه‌های بعدی را پوشش می‌دهند. هر نگاشت، راهنمای مهاجرت است، نه مدرکی بر این‌که سطح قدیمی همچنان در دسترس است؛ برای وضعیت فعلی، به رجیستری سازگاری و جدول زمانی حذف مراجعه کنید.

  <AccordionGroup>
  <Accordion title="سازنده‌های راهنمای command-auth -> command-status">
    **قدیمی (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`،
    `buildCommandsMessagePaginated`، `buildHelpMessage`.

    **جدید (`openclaw/plugin-sdk/command-status`)**: همان امضاها که
    از زیرمسیر محدودتر وارد می‌شوند. بازصادرهای سازگاری `command-auth`
    حذف شده‌اند.

    ```typescript
    // پیش از این
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // پس از این
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="کمک‌کننده‌های کنترل منشن -> resolveInboundMentionDecision">
    **قدیمی**: `resolveMentionGating(params)` و
    `resolveMentionGatingWithBypass(params)` از
    `openclaw/plugin-sdk/channel-inbound` یا
    `openclaw/plugin-sdk/channel-mention-gating`.

    **جدید**: `resolveInboundMentionDecision({ facts, policy })` ــ یک شیء تصمیم‌گیری
    به‌جای دو شکل فراخوانی مجزا.

    این تغییر در Discord، iMessage، Matrix، MS Teams، QQBot، Signal،
    Telegram، WhatsApp و Zalo اعمال شده است. مدل رویداد `app_mention` خود Slack
    از این کمک‌کننده استفاده نمی‌کند.

  </Accordion>

  <Accordion title="شیم زمان اجرای کانال و کمک‌کننده‌های کنش‌های کانال">
    `openclaw/plugin-sdk/channel-runtime` حذف شده است. برای ثبت اشیای
    زمان اجرا از `openclaw/plugin-sdk/channel-runtime-context` استفاده کنید.

    کمک‌کننده‌های شِمای پیام بومی در `openclaw/plugin-sdk/channel-actions`
    همراه با خروجی‌های خام «actions» کانال حذف شدند. در عوض، قابلیت‌ها را
    از طریق سطح معنایی `presentation` ارائه کنید ــ Pluginهای کانال
    به‌جای نام کنش‌های خامی که می‌پذیرند، مواردی را که رندر می‌کنند
    (کارت‌ها، دکمه‌ها، انتخاب‌گرها) اعلام می‌کنند.

  </Accordion>

  <Accordion title="کمک‌کنندهٔ tool() ارائه‌دهندهٔ جست‌وجوی وب -> createTool() روی Plugin">
    **قدیمی**: کارخانهٔ `tool()` از `openclaw/plugin-sdk/provider-web-search`.

    **جدید**: `createTool(...)` را مستقیماً روی Plugin ارائه‌دهنده پیاده‌سازی کنید.
    OpenClaw دیگر برای ثبت پوشش ابزار به کمک‌کنندهٔ SDK نیاز ندارد.

  </Accordion>

  <Accordion title="پاکت‌های متن سادهٔ کانال -> BodyForAgent">
    **قدیمی**: `api.runtime.channel.reply.formatInboundEnvelope(...)` (و فیلد
    `channelEnvelope` روی اشیای پیام ورودی) برای ساخت یک پاکت پرامپت
    متن ساده و تخت از پیام‌های ورودی کانال.

    **جدید**: `BodyForAgent` به‌همراه بلوک‌های ساخت‌یافتهٔ زمینهٔ کاربر. Pluginهای
    کانال، فرادادهٔ مسیریابی (رشته، موضوع، پاسخ‌به، واکنش‌ها) را
    به‌صورت فیلدهای نوع‌دار پیوست می‌کنند، نه این‌که آن‌ها را در یک رشتهٔ پرامپت به هم بچسبانند.
    کمک‌کنندهٔ `formatAgentEnvelope(...)` همچنان برای پاکت‌های ترکیبی
    روبه‌دستیار پشتیبانی می‌شود، اما پاکت‌های متن سادهٔ ورودی در مسیر
    حذف قرار دارند.

    نواحی تحت‌تأثیر: `inbound_claim`، `message_received` و هر
    Plugin سفارشی کانالی که متن پاکت قدیمی را پس‌پردازش می‌کرد.

  </Accordion>

  <Accordion title="قلاب deactivate -> gateway_stop">
    **قدیمی**: `api.on("deactivate", handler)`.

    **جدید**: `api.on("gateway_stop", handler)`. قرارداد پاک‌سازی هنگام خاموش‌شدن
    یکسان است؛ فقط نام قلاب تغییر می‌کند.

    ```typescript
    // پیش از این
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // پس از این
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate` تا زمان حذف پس از 2026-08-16، به‌عنوان یک نام مستعار
    سازگاری منسوخ‌شده متصل باقی می‌ماند.

  </Accordion>

  <Accordion title="قلاب subagent_spawning -> اتصال رشته در هسته">
    **قدیمی**: `api.on("subagent_spawning", handler)` که
    `threadBindingReady` یا `deliveryOrigin` را برمی‌گرداند.

    **جدید**: اجازه دهید هسته اتصال‌های عامل فرعی `thread: true` را از طریق
    آداپتور اتصال نشست کانال آماده کند. از `api.on("subagent_spawned", handler)`
    فقط برای مشاهدهٔ پس از راه‌اندازی استفاده کنید.

    ```typescript
    // پیش از این
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // پس از این
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`، `PluginHookSubagentSpawningEvent`،
    `PluginHookSubagentSpawningResult` و
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)` فقط به‌عنوان
    سطوح سازگاری منسوخ‌شده تا زمان مهاجرت Pluginهای خارجی باقی می‌مانند و
    پس از 2026-08-30 حذف می‌شوند.

  </Accordion>

  <Accordion title="نوع‌های کشف ارائه‌دهنده -> نوع‌های کاتالوگ ارائه‌دهنده">
    چهار نام مستعار نوع کشف اکنون پوشش‌های نازکی روی نوع‌های
    دورهٔ کاتالوگ هستند:

    | نام مستعار قدیمی                 | نوع جدید                  |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    نام‌های مستعار و مجموعهٔ ایستای قدیمی `ProviderCapabilities`
    حذف شده‌اند. Pluginهای ارائه‌دهنده
    باید به‌جای یک شیء ایستا، از قلاب‌های صریح ارائه‌دهنده مانند `buildReplayPolicy`،
    `normalizeToolSchemas` و `wrapStreamFn` استفاده کنند.

  </Accordion>

  <Accordion title="قلاب‌های سیاست تفکر -> resolveThinkingProfile">
    **قدیمی** (سه قلاب مجزا روی `ProviderThinkingPolicy`):
    `isBinaryThinking(ctx)`، `supportsXHighThinking(ctx)` و
    `resolveDefaultThinkingLevel(ctx)`.

    **جدید**: یک `resolveThinkingProfile(ctx)` واحد که یک
    `ProviderThinkingProfile` را با `id` معیار، `label` اختیاری و یک
    فهرست رتبه‌بندی‌شده از سطوح برمی‌گرداند. OpenClaw مقادیر ذخیره‌شدهٔ کهنه را بر اساس رتبهٔ پروفایل
    به‌طور خودکار تنزل می‌دهد.

    زمینه شامل `provider`، `modelId`، `reasoning` ادغام‌شدهٔ اختیاری
    و واقعیت‌های مدل `compat` ادغام‌شدهٔ اختیاری است. Pluginهای ارائه‌دهنده می‌توانند از این
    واقعیت‌های کاتالوگ استفاده کنند تا فقط هنگامی یک پروفایل مختص مدل ارائه دهند که قرارداد
    درخواست پیکربندی‌شده از آن پشتیبانی کند.

    به‌جای سه قلاب، یک قلاب پیاده‌سازی کنید. قلاب‌های قدیمی حذف شده‌اند.

  </Accordion>

  <Accordion title="ارائه‌دهندگان احراز هویت خارجی -> contracts.externalAuthProviders">
    **قدیمی**: پیاده‌سازی قلاب‌های احراز هویت خارجی بدون اعلام ارائه‌دهنده
    در مانیفست Plugin.

    **جدید**: `contracts.externalAuthProviders` را در مانیفست Plugin اعلام
    **و** `resolveExternalAuthProfiles(...)` را پیاده‌سازی کنید.

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="جست‌وجوی متغیر محیطی ارائه‌دهنده -> setup.providers[].envVars">
    فیلد مانیفست **قدیمی**: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`.

    **جدید**: همان جست‌وجوی متغیر محیطی را در `setup.providers[].envVars`
    روی مانیفست بازتاب دهید. این کار فرادادهٔ محیطی راه‌اندازی/وضعیت را در یک مکان یکپارچه می‌کند
    و از راه‌اندازی زمان اجرای Plugin فقط برای پاسخ‌دادن به جست‌وجوهای متغیر محیطی جلوگیری می‌کند.

    `providerAuthEnvVars` دیگر پذیرفته نمی‌شود.

  </Accordion>

  <Accordion title="ثبت Plugin حافظه -> registerMemoryCapability">
    **قدیمی**: سه فراخوانی مجزا ــ `api.registerMemoryPromptSection(...)`،
    `api.registerMemoryFlushPlan(...)`، `api.registerMemoryRuntime(...)`.

    **جدید**: یک فراخوانی روی API وضعیت حافظه ــ
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`.

    همان شکاف‌ها، یک فراخوانی ثبت. کمک‌کننده‌های افزایشی پرامپت و پیکره
    (`registerMemoryPromptSupplement`، `registerMemoryCorpusSupplement`)
    تحت‌تأثیر قرار نمی‌گیرند.

  </Accordion>

  <Accordion title="API ارائه‌دهندهٔ جاسازی حافظه">
    **قدیمی**: `api.registerMemoryEmbeddingProvider(...)` به‌همراه
    `contracts.memoryEmbeddingProviders`.

    **جدید**: `api.registerEmbeddingProvider(...)` به‌همراه
    `contracts.embeddingProviders`.

    قرارداد عمومی ارائه‌دهندهٔ جاسازی خارج از حافظه نیز قابل استفادهٔ مجدد است و
    مسیر پشتیبانی‌شده برای ارائه‌دهندگان جدید محسوب می‌شود. API ثبت مختص حافظه
    در حین مهاجرت ارائه‌دهندگان موجود، به‌عنوان سازگاری منسوخ‌شده
    متصل باقی می‌ماند. بازرسی Plugin، استفادهٔ غیرباندل‌شده را به‌عنوان
    بدهی سازگاری گزارش می‌کند.

  </Accordion>

  <Accordion title="نتایج خام ارسال کانال -> OutboundDeliveryResult">
    **قدیمی**: برگرداندن `{ ok, messageId, error }` از طریق
    `ChannelSendRawResult` و نرمال‌سازی آن با
    `createRawChannelSendResultAdapter(...)`.

    **جدید**: فیلدهای `OutboundDeliveryResult` را برگردانید و کانال را با
    `createAttachedChannelResultAdapter(...)` پیوست کنید. ارسال‌های ناموفق باید به‌جای
    برگرداندن رشتهٔ خطا، استثنا ایجاد کنند. نوع نتیجهٔ خام تا
    انتشار اصلی بعدی SDK مربوط به Plugin در دسترس باقی می‌ماند.

  </Accordion>

  <Accordion title="تغییر نام نوع‌های پیام نشست عامل فرعی">
    دو نام مستعار نوع قدیمی همچنان از `src/plugins/runtime/types.ts` صادر می‌شوند:

    | قدیمی                           | جدید                             |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    متد زمان اجرای `readSession` به‌نفع
    `getSessionMessages` منسوخ شده است. امضا یکسان است؛ متد قدیمی فراخوانی را به
    متد جدید واگذار می‌کند.

  </Accordion>

  <Accordion title="APIهای حذف‌شدهٔ فایل نشست و رونوشت">
    تغییر نشست/رونوشت به SQLite، APIهای روبه‌Plugin را که
    مخزن‌های فعال `sessions.json`، مسیرهای رونوشت JSONL یا فهرست‌های
    فایل نشست را افشا می‌کردند، حذف یا منسوخ می‌کند. Pluginهای زمان اجرا باید به‌جای
    تفکیک یا تغییر فایل‌های فعال، از هویت نشست و کمک‌کننده‌های زمان اجرای SDK
    استفاده کنند.

    | سطح در حال مهاجرت | جایگزین |
    | ----------------- | ----------- |
    | `loadSessionStore(...)`، `updateSessionStore(...)` و `resolveSessionStoreEntry(...)` منسوخ‌شده | `getSessionEntry(...)`، `listSessionEntries(...)` و تغییرات نشست در سطح ردیف. |
    | `resolveSessionFilePath(...)` منسوخ‌شده | هویت نشست (`sessionKey`، `sessionId` و کمک‌کننده‌های هدف زمان اجرای SDK) به‌همراه متدهای Gateway که روی نشست فعلی عمل می‌کنند. |
    | `saveSessionStore(...)` حذف‌شده | APIهای زمان اجرای نشست تحت مالکیت Gateway؛ کد Plugin باید به‌جای نوشتن در فایل مخزن فعال، وضعیت نشست را از طریق کمک‌کننده‌های مستندشدهٔ زمان اجرا/زمینه درخواست یا تغییر دهد. |
    | `resolveSessionTranscriptPathInDir(...)` و `resolveAndPersistSessionFile(...)` حذف‌شده | هویت نشست و متدهای Gateway که روی نشست فعلی عمل می‌کنند. |
    | `readLatestAssistantTextFromSessionTranscript(...)` | خوانشگرهای رونوشت مبتنی بر هویت که زمینهٔ زمان اجرای فعلی ارائه می‌کند، یا متدهای تاریخچه/نشست Gateway هنگامی که Plugin خارج از مسیر مالک رونوشت است. |
    | `SessionTranscriptUpdate.sessionFile` | `SessionTranscriptUpdate.target` با `agentId`، `sessionKey` و `sessionId`. |
    | ورودی‌های همگام‌سازی حافظه مانند `sessionFiles` | منابع رونوشت/نشست مبتنی بر هویت که میزبان ارائه می‌کند؛ فایل‌های فعال JSONL را برای نشست‌های زنده پیمایش نکنید. |
    | گزینه‌های زمان اجرا با نام `transcriptPath` یا `sessionFile` برای نشست‌های فعال | اشیای `sessionTarget`/هدف زمان اجرا که هویت نشست مستقل از ذخیره‌سازی را حمل می‌کنند. |

    فایل‌های قدیمی رونوشت JSONL همچنان به‌عنوان مصنوعات واردکردن، بایگانی،
    صادرکردن و پشتیبانی معتبر هستند. آن‌ها دیگر قرارداد پایدار زمان اجرا برای
    نشست‌های فعال نیستند.

    Pluginهای رسمی منتشرشده با `v2026.7.1-beta.5` چهار
    کمک‌کنندهٔ منسوخ‌شدهٔ بالا را وارد می‌کردند. `openclaw/plugin-sdk/session-store-runtime`
    دقیقاً همان پل را تا 2026-10-12 حفظ می‌کند؛ Pluginهای جدید باید از جایگزین‌ها استفاده کنند.
    `resolveStorePath(...)` همچنان یک کمک‌کنندهٔ پشتیبانی‌شدهٔ SDK است و بخشی از
    این منسوخ‌سازی نیست.

    `openclaw plugins inspect --all --runtime`، Pluginهای غیرباندل‌شده‌ای را گزارش می‌کند که
    خطاهای بارگذاری یا عیب‌یابی‌هایشان همچنان به این APIهای فایل حذف‌شده اشاره دارند. پیمایش
    مشورتی `@openclaw/plugin-inspector` باید از نسخهٔ `0.3.17` یا
    جدیدتر استفاده کند تا اسکن بسته‌های خارجی نیز کمک‌کننده‌های نشست در سطح کل مخزن،
    کمک‌کننده‌های مسیر فایل نشست، هدف‌های قدیمی فایل رونوشت و کمک‌کننده‌های سطح پایین
    رونوشت را پیش از انتشار علامت‌گذاری کند.

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **قدیمی**: `runtime.tasks.flow` (مفرد) یک دسترسی‌دهندهٔ زندهٔ جریان وظیفه
    برمی‌گرداند.

    **جدید**: `runtime.tasks.managedFlows` زمان اجرای تغییر مدیریت‌شدهٔ TaskFlow را
    برای Pluginهایی که از یک جریان، وظایف فرزند را ایجاد، به‌روزرسانی، لغو یا اجرا می‌کنند حفظ می‌کند.
    هنگامی که Plugin فقط به خواندن مبتنی بر DTO نیاز دارد، از `runtime.tasks.flows` استفاده کنید.

    ```typescript
    // پیش از تغییر
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // پس از تغییر
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    نام‌های مستعار قدیمی در ژوئیهٔ ۲۰۲۶ حذف شدند.

  </Accordion>

  <Accordion title="کارخانه‌های افزونهٔ تعبیه‌شده -> میان‌افزار نتیجهٔ ابزار عامل">
    این موضوع در بخش [نحوهٔ مهاجرت](#how-to-migrate) در بالا پوشش داده شده است. برای
    تکمیل اطلاعات، مسیر حذف‌شدهٔ مختص اجراکنندهٔ تعبیه‌شدهٔ
    `api.registerEmbeddedExtensionFactory(...)` با
    `api.registerAgentToolResultMiddleware(...)` و یک فهرست صریح از زمان‌های اجرا
    در `contracts.agentToolResultMiddleware` جایگزین می‌شود.
  </Accordion>

  <Accordion title="نام مستعار OpenClawSchemaType -> OpenClawConfig">
    نام مستعار SDK ریشهٔ `OpenClawSchemaType` حذف شد. از نام متعارف
    `OpenClawConfig` استفاده کنید.

    ```typescript
    // پیش از تغییر
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // پس از تغییر
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
موارد منسوخ‌شده در سطح افزونه (داخل Pluginهای کانال/ارائه‌دهندهٔ همراه در
`extensions/`) در barrelهای `api.ts` و `runtime-api.ts`
خودشان پیگیری می‌شوند. آن‌ها بر قراردادهای Pluginهای شخص ثالث تأثیری ندارند و
در اینجا فهرست نشده‌اند. اگر barrel محلی یک Plugin همراه را مستقیماً مصرف می‌کنید،
پیش از ارتقا دیدگاه‌های مربوط به منسوخ‌شدن در آن barrel را بخوانید.
</Note>

## مهاجرت Talk و صدای بی‌درنگ

کد صدای بی‌درنگ، تلفن، جلسه و Talk مرورگر، یک کنترل‌کنندهٔ نشست Talk مشترک دارد
که توسط `openclaw/plugin-sdk/realtime-voice` صادر می‌شود. این
کنترل‌کننده مالک پوش رویداد مشترک Talk، وضعیت نوبت فعال، وضعیت ضبط،
وضعیت صدای خروجی، تاریخچهٔ رویدادهای اخیر و رد نوبت‌های منقضی‌شده است.
Pluginهای ارائه‌دهنده مالک نشست‌های بی‌درنگ مختص فروشنده هستند. Pluginهای جلسهٔ مرورگر
از `openclaw/plugin-sdk/meeting-runtime` برای سازوکارهای نشست، مرورگر، صدا، میزبان Node،
مشاوره با عامل و تماس صوتی استفاده می‌کنند و سپس `MeetingPlatformAdapter`
را برای قواعد URL، اسکریپت‌های DOM، نگاشت اقدام دستی، زیرنویس‌ها، ایجاد و طرح‌های
شماره‌گیری ورودی پیاده‌سازی می‌کنند. APIهای REST پلتفرم، OAuth، مصنوعات، انتخابگرها
و نام‌های سیمی در Plugin باقی می‌مانند. طرح‌های مجوز مرورگر URL جلسهٔ درخواستی را
دریافت می‌کنند تا هر پلتفرم بتواند فقط به مبدأهای دقیقاً پشتیبانی‌شدهٔ خود مجوز دهد.
زمان‌های اجرای نشست باید سلامت زندهٔ مختص پلتفرم را نیز پس از خروج تأییدشده از مرورگر
عادی‌سازی کنند؛ فیلدهای رونوشت تاریخی می‌توانند باقی بمانند، اما آمادگی زیرنویس و صدا
نباید پس از خروج فعال بماند.

همهٔ سطوح همراه روی کنترل‌کنندهٔ مشترک اجرا می‌شوند: رلهٔ مرورگر،
واگذاری اتاق مدیریت‌شده، بی‌درنگ تماس صوتی، STT جریانی تماس صوتی، بی‌درنگ Google
Meet و فشردن برای صحبت بومی. Gateway یک کانال رویداد زندهٔ Talk را
در `hello-ok.features.events` اعلام می‌کند: `talk.event`.

کد جدید نباید `createTalkEventSequencer(...)` را مستقیماً فراخوانی کند، مگر اینکه
یک آداپتور سطح‌پایین یا فیکسچر آزمون را پیاده‌سازی کند. از کنترل‌کنندهٔ مشترک استفاده کنید
تا رویدادهای محدود به نوبت بدون شناسهٔ نوبت منتشر نشوند، فراخوانی‌های منقضی‌شدهٔ
`turnEnd` / `turnCancel` نتوانند نوبت فعال جدیدتری را پاک کنند و
رویدادهای چرخهٔ عمر صدای خروجی در تلفن، جلسات، رلهٔ مرورگر، واگذاری اتاق مدیریت‌شده
و کلاینت‌های بومی Talk سازگار بمانند.

شکل API عمومی:

```typescript
// API نشست Talk تحت مالکیت Gateway.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// API نشست ارائه‌دهنده تحت مالکیت کلاینت.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

نشست‌های WebRTC/وب‌سوکت ارائه‌دهنده تحت مالکیت مرورگر از `talk.client.create`
استفاده می‌کنند، زیرا مرورگر مالک مذاکره با ارائه‌دهنده و انتقال رسانه است، درحالی‌که
Gateway مالک اعتبارنامه‌ها، دستورالعمل‌ها و سیاست ابزار است. `talk.session.*`
سطح مشترک مدیریت‌شده توسط Gateway برای بی‌درنگ رلهٔ Gateway، رونویسی رلهٔ Gateway
و نشست‌های بومی STT/TTS اتاق مدیریت‌شده است.

پیکربندی‌های قدیمی که انتخابگرهای بی‌درنگ را کنار `talk.provider` /
`talk.providers` قرار می‌دهند باید با `openclaw doctor --fix` تعمیر شوند؛ Talk در زمان اجرا
پیکربندی ارائه‌دهندهٔ گفتار/TTS را به‌عنوان پیکربندی ارائه‌دهندهٔ بی‌درنگ بازتفسیر نمی‌کند.

ترکیب‌های پشتیبانی‌شدهٔ `talk.session.create` عمداً محدود هستند:

| حالت            | انتقال       | مغز           | مالک              | یادداشت‌ها                                                                                                              |
| --------------- | --------------- | --------------- | ------------------ | ------------------------------------------------------------------------------------------------------------------ |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | صدای تمام‌دوطرفهٔ ارائه‌دهنده از طریق Gateway پل می‌شود؛ فراخوانی‌های ابزار از مسیر ابزار مشاوره با عامل هدایت می‌شوند.           |
| `transcription` | `gateway-relay` | `none`          | Gateway            | فقط STT جریانی؛ فراخوانندگان صدای ورودی را ارسال و رویدادهای رونوشت را دریافت می‌کنند.                                        |
| `stt-tts`       | `managed-room`  | `agent-consult` | اتاق بومی/کلاینت | اتاق‌هایی به سبک فشردن برای صحبت و واکی‌تاکی که در آن‌ها کلاینت مالک ضبط/پخش و Gateway مالک وضعیت نوبت است. |
| `stt-tts`       | `managed-room`  | `direct-tools`  | اتاق بومی/کلاینت | حالت اتاق مختص مدیر برای سطوح شخص اول مورداعتماد که اقدامات ابزار Gateway را مستقیماً اجرا می‌کنند.                  |

نگاشت متد برای خوانندگانی که از خانواده‌های قدیمی `talk.realtime.*` /
`talk.transcription.*` / `talk.handoff.*` مهاجرت می‌کنند (همگی حذف شده‌اند):

| قدیمی                              | جدید                                                      |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` یا `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

واژگان کنترل یکپارچه نیز عمداً محدود هستند:

| متد                          | قابل‌اعمال به                                              | قرارداد                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`، `transcription/gateway-relay` | یک قطعهٔ صوتی PCM با کدگذاری base64 را به نشست ارائه‌دهندهٔ متعلق به همان اتصال Gateway اضافه می‌کند.                                                                                                                             |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | یک نوبت کاربر در اتاق مدیریت‌شده را آغاز می‌کند.                                                                                                                                                                                           |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | نوبت فعال را پس از اعتبارسنجی نوبت منقضی‌شده پایان می‌دهد.                                                                                                                                                                          |
| `talk.session.cancelTurn`       | همهٔ نشست‌های تحت مالکیت Gateway                              | کار فعال ضبط/ارائه‌دهنده/عامل/TTS را برای یک نوبت لغو می‌کند.                                                                                                                                                                 |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | خروجی صدای دستیار را بدون اینکه لزوماً نوبت کاربر پایان یابد متوقف می‌کند.                                                                                                                                                     |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | فراخوانی ابزار ارائه‌دهنده را پس از هر تکمیل ناهمگام در معرض‌گذاری‌شده توسط پل آن کامل می‌کند؛ برای خروجی موقت `options.willContinue` یا، در صورت پشتیبانی، برای جلوگیری از پاسخ دیگری از دستیار `options.suppressResponse` را ارسال کنید. |
| `talk.session.steer`            | نشست‌های Talk متکی به عامل                              | کنترل گفتاری `status`، `steer`، `cancel` یا `followup` را به اجرای تعبیه‌شدهٔ فعال که از نشست Talk حل شده است ارسال می‌کند.                                                                                                 |
| `talk.session.close`            | همهٔ نشست‌های یکپارچه                                    | نشست‌های رله را متوقف یا وضعیت اتاق مدیریت‌شده را لغو می‌کند و سپس شناسهٔ نشست یکپارچه را فراموش می‌کند.                                                                                                                                     |

برای عملی‌کردن این سازوکار، موارد خاص ارائه‌دهنده یا پلتفرم را در هسته معرفی نکنید.
هسته مالک معناشناسی نشست Talk است. Pluginهای ارائه‌دهنده مالک راه‌اندازی نشست فروشنده هستند.
تماس صوتی و Google Meet مالک آداپتورهای تلفن/جلسه هستند. برنامه‌های مرورگر و بومی
مالک تجربهٔ کاربری ضبط/پخش دستگاه هستند.

## جدول زمانی حذف

| زمان                                        | چه اتفاقی می‌افتد                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **اکنون**                                     | سطوح منسوخ‌شده‌ای که قابلیت هشدار دارند، هشدارهای زمان اجرا صادر می‌کنند؛ محافظ‌های مخزن، واردسازی‌های SDK منسوخ‌شده از هسته و Pluginهای همراه را رد می‌کنند. |
| **در انتظار تصمیم مالک**                  | رکوردهای بدون تاریخ تا زمانی که مالکشان یک تاریخ `removeAfter` منتشر نکند، منسوخ باقی می‌مانند و واجد شرایط حذف نیستند.                          |
| **تاریخ `removeAfter` هر رکورد سازگاری** | آن سطح مشخص واجد شرایط حذف می‌شود؛ پس از گذشت تاریخ، `pnpm plugins:boundary-report --fail-on-eligible-compat` باعث شکست CI می‌شود.    |
| **نسخه اصلی بعدی**                      | سطوح تاریخ‌دار فقط پس از تاریخ `removeAfter` خود قابل حذف هستند؛ رکوردهای بدون تاریخ همچنان به تأیید مالک و یک تاریخ منتشرشده نیاز دارند.   |

زیرمسیرهای عمومی باقی‌مانده SDK در زیر، بازه‌های حذف مبتنی بر رجیستری دارند.
ردیف‌های 30 ژوئیه پس از پاک‌سازی زودهنگامِ مجازشده توسط نگه‌دارنده حذف شدند:
زیرمسیرهای استفاده‌نشده حذف شدند، نام‌های مستعار سازگاری پیشین حذف شدند و
ماژول‌های مخصوص نسخه همراه به نگاشت‌های ساخت خصوصی و محلی تنزل یافتند.

| `removeAfter` | رده                               | زیرمسیرهای SDK                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | منسوخ‌سازی‌های سازگاری پیشین | `agent-config-primitives`، `channel-logging`، `channel-secret-runtime`، `channel-streaming`، `group-access`، `inbound-reply-dispatch`، `matrix`، `text-runtime`، `zod`              |
| `2026-09-01`  | منسوخ‌سازی‌های سازگاری پیشین | `channel-lifecycle`، `channel-message`، `channel-reply-pipeline`، `config-runtime`، `infra-runtime`                                                                                 |
| `2026-10-01`  | نگاشت قدیمی رسانه            | `agent-media-payload`، به‌علاوه فیلدهای غیرزیرمسیرِ `MsgContext Media*`، سازنده‌های محموله رسانه ورودی کانال، `buildMediaPayload`، نام‌های مستعار رسانه‌ای هوک و قالب‌های `{{Media*}}` |

همه Pluginهای هسته از قبل مهاجرت کرده‌اند. Pluginهای خارجی باید
پیش از نسخه اصلی بعدی مهاجرت کنند. برای مشاهده اینکه کدام
رکوردهای سازگاری برای سطوح مورداستفاده Plugin شما زودتر سررسید می‌شوند، `pnpm plugins:boundary-report` را اجرا کنید.

## سرکوب موقت هشدارها

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

این یک راه فرار موقت است، نه راه‌حلی دائمی.

## مرتبط

- [شروع به کار](/fa/plugins/building-plugins) - نخستین Plugin خود را بسازید
- [نمای کلی SDK](/fa/plugins/sdk-overview) - مرجع کامل واردسازی زیرمسیرها
- [Pluginهای کانال](/fa/plugins/sdk-channel-plugins) - ساخت Pluginهای کانال
- [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) - ساخت Pluginهای ارائه‌دهنده
- [جزئیات داخلی Plugin](/fa/plugins/architecture) - بررسی عمیق معماری
- [مانیفست Plugin](/fa/plugins/manifest) - مرجع شِمای مانیفست
