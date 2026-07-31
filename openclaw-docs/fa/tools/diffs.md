---
read_when:
    - می‌خواهید عامل‌ها ویرایش‌های کد یا Markdown را به‌صورت diff نمایش دهند
    - یک URL نمایشگر آماده برای canvas یا یک فایل diff رندرشده می‌خواهید
    - به آرتیفکت‌های موقت و کنترل‌شدهٔ diff با پیش‌فرض‌های امن نیاز دارید
sidebarTitle: Diffs
summary: نمایشگر تفاوتِ فقط‌خواندنی و رندرکنندهٔ فایل برای عامل‌ها (ابزار اختیاری Plugin)
title: تفاوت‌ها
x-i18n:
    generated_at: "2026-07-27T15:52:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs` یک ابزار اختیاریِ Plugin همراه است که متن قبل/بعد یا یک وصلهٔ یکپارچه را به یک مصنوع diff فقط‌خواندنی تبدیل می‌کند. همچنین راهنمایی کوتاهی برای عامل به ابتدای اعلان سیستمی می‌افزاید و یک Skill همراه برای دستورالعمل‌های کامل‌تر ارائه می‌کند.

ورودی: متن `before` + `after`، یا یک `patch` یکپارچه (مانعةالجمع).

خروجی: یک نشانی URL نمایشگر Gateway برای ارائه در بوم، مسیر فایل PNG/PDF رندرشده برای تحویل در پیام، یا هر دو.

## شروع سریع

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="فعال‌سازی Plugin">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="انتخاب یک حالت">
    <Tabs>
      <Tab title="view">
        جریان‌های بوم‌محور: عامل‌ها `diffs` را با `mode: "view"` فراخوانی می‌کنند و `details.viewerUrl` را با `canvas present` باز می‌کنند.
      </Tab>
      <Tab title="file">
        تحویل فایل در گفت‌وگو: عامل‌ها `diffs` را با `mode: "file"` فراخوانی می‌کنند و `details.filePath` را با `message`، با استفاده از `path` یا `filePath`، ارسال می‌کنند.
      </Tab>
      <Tab title="both">
        ترکیبی (پیش‌فرض): عامل‌ها `diffs` را با `mode: "both"` فراخوانی می‌کنند تا هر دو مصنوع را در یک فراخوانی دریافت کنند.
      </Tab>
    </Tabs>
  </Step>
</Steps>

## غیرفعال‌سازی راهنمای سیستمی داخلی

برای نگه‌داشتن ابزار و حذف راهنمای افزوده‌شده به اعلان سیستمی، `plugins.entries.diffs.hooks.allowPromptInjection` را روی `false` تنظیم کنید:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

این کار hook مربوط به `before_prompt_build` در Plugin را مسدود می‌کند و در عین حال ابزار و Skill را در دسترس نگه می‌دارد. برای غیرفعال‌سازی هم راهنما و هم ابزار، خود Plugin را غیرفعال کنید.

## مرجع ورودی ابزار

همهٔ فیلدها اختیاری‌اند، مگر آنکه خلافش ذکر شده باشد.

<ParamField path="before" type="string">
  متن اصلی. هنگامی که `patch` حذف شده باشد، همراه با `after` الزامی است.
</ParamField>
<ParamField path="after" type="string">
  متن به‌روزشده. هنگامی که `patch` حذف شده باشد، همراه با `before` الزامی است.
</ParamField>
<ParamField path="patch" type="string">
  متن diff یکپارچه. با `before` و `after` مانعةالجمع است.
</ParamField>
<ParamField path="path" type="string">
  نام فایل نمایشی برای حالت قبل/بعد.
</ParamField>
<ParamField path="lang" type="string">
  راهنمای بازنویسی زبان برای حالت قبل/بعد. مقادیر ناشناخته و زبان‌های خارج از مجموعهٔ پیش‌فرض نمایشگر، مگر آنکه Plugin بستهٔ زبان نمایشگر Diff نصب شده باشد، به متن ساده بازمی‌گردند.
</ParamField>
<ParamField path="title" type="string">
  بازنویسی عنوان نمایشگر.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  حالت خروجی. مقدار پیش‌فرض، مقدار پیش‌فرض Plugin یعنی `defaults.mode` (`both`) است. نام مستعار منسوخ: `"image"` دقیقاً مانند `"file"` رفتار می‌کند.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  پوستهٔ نمایشگر. مقدار پیش‌فرض، مقدار پیش‌فرض Plugin یعنی `defaults.theme` است.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  چیدمان diff. مقدار پیش‌فرض، مقدار پیش‌فرض Plugin یعنی `defaults.layout` است.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  در صورت وجود زمینهٔ کامل، بخش‌های بدون تغییر را باز کنید. فقط گزینه‌ای برای هر فراخوانی است (نه کلید پیش‌فرض Plugin).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  قالب فایل رندرشده. مقدار پیش‌فرض، مقدار پیش‌فرض Plugin یعنی `defaults.fileFormat` است.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  پیش‌تنظیم کیفیت برای رندر PNG/PDF.
</ParamField>
<ParamField path="fileScale" type="number">
  بازنویسی مقیاس دستگاه (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  حداکثر عرض رندر برحسب پیکسل CSS ‏(`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  TTL مصنوع برحسب ثانیه برای خروجی‌های نمایشگر و فایل مستقل. حداکثر `21600`.
</ParamField>
<ParamField path="baseUrl" type="string">
  بازنویسی مبدأ نشانی URL نمایشگر. `viewerBaseUrl` مربوط به Plugin را بازنویسی می‌کند. باید `http` یا `https` و بدون پرس‌وجو/هش باشد.
</ParamField>

<AccordionGroup>
  <Accordion title="اعتبارسنجی و محدودیت‌ها">
    - `before`/`after`: حداکثر 512 KiB برای هرکدام.
    - `patch`: حداکثر 2 MiB.
    - `path`: حداکثر 2048 بایت.
    - `lang`: حداکثر 128 بایت.
    - `title`: حداکثر 1024 بایت.
    - سقف پیچیدگی وصله: حداکثر 128 فایل و در مجموع 120000 خط.
    - استفادهٔ هم‌زمان از `patch` با `before`/`after` رد می‌شود.
    - محدودیت‌های ایمنی فایل رندرشده (PNG و PDF):
      - `fileQuality: "standard"`: حداکثر 8 MP ‏(8,000,000 پیکسل رندرشده).
      - `fileQuality: "hq"`: حداکثر 14 MP.
      - `fileQuality: "print"`: حداکثر 24 MP.
      - PDF همچنین به 50 صفحه محدود است.

  </Accordion>
</AccordionGroup>

## برجسته‌سازی نحو

زبان‌های داخلی:

`javascript`، `typescript`، `tsx`، `jsx`، `json`، `markdown`، `yaml`، `css`، `html`، `sh`، `python`، `go`، `rust`، `java`، `c`، `cpp`، `csharp`، `php`، `sql`، `docker`، `ruby`، `swift`، `kotlin`، `r`، `dart`، `lua`، `powershell`، `xml` و `toml`.

نام‌های مستعار رایج (`js`، `ts`، `bash`، `md`، `yml`، `c++`، `dockerfile`، `rb`، `kt`، `ps1` و غیره) به آن زبان‌ها نرمال‌سازی می‌شوند.

برای زبان‌های بیشتر (Astro، Vue، Svelte، MDX، GraphQL، Terraform/HCL، Nix، Clojure، Elixir، Haskell، OCaml، Scala، Zig، Solidity، Verilog/VHDL، Fortran، MATLAB، LaTeX، Mermaid، Sass/Less/SCSS، Nginx، Apache، CSV، dotenv، INI، diff و موارد بیشتر)، Plugin بستهٔ زبان نمایشگر Diff را نصب کنید:

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

بدون این بسته، زبان‌های پشتیبانی‌نشده همچنان به‌صورت متن ساده و خوانا رندر می‌شوند. برای فهرست بالادستی، به [Plugin بستهٔ زبان Diffs](/fa/plugins/reference/diffs-language-pack) و [زبان‌های Shiki](https://shiki.style/languages) مراجعه کنید.

## قرارداد جزئیات خروجی

همهٔ نتایج موفق شامل `changed` هستند: ورودی یکسان قبل/بعد، بدون ایجاد مصنوع، `false` را برمی‌گرداند؛ نتایج رندرشده `true` را برمی‌گردانند.

<AccordionGroup>
  <Accordion title="فیلدهای نمایشگر (حالت‌های view و both)">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (`agentId`، `sessionId`، `messageChannel`، `agentAccountId` در صورت وجود)

  </Accordion>
  <Accordion title="فیلدهای فایل (حالت‌های file و both)">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (همان مقدار `filePath`، برای سازگاری با ابزار پیام)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| حالت     | موارد بازگشتی                                                                                         |
| -------- | ----------------------------------------------------------------------------------------------- |
| `"view"` | فقط فیلدهای نمایشگر.                                                                             |
| `"file"` | فقط فیلدهای فایل، بدون مصنوع نمایشگر.                                                           |
| `"both"` | فیلدهای نمایشگر به‌همراه فیلدهای فایل. اگر رندر فایل ناموفق باشد، نمایشگر همچنان با `fileError` بازگردانده می‌شود. |

### بخش‌های بدون تغییرِ جمع‌شده

نمایشگر ردیف‌هایی مانند `N unmodified lines` نشان می‌دهد. کنترل‌های بازکردن فقط زمانی ظاهر می‌شوند که diff رندرشده دادهٔ زمینهٔ قابل‌بازشدن داشته باشد (معمولاً برای ورودی قبل/بعد). بسیاری از وصله‌های یکپارچه بدنهٔ زمینه را در قطعه‌های خود حذف می‌کنند؛ بنابراین ممکن است ردیف بدون کنترل بازکردن ظاهر شود -- این رفتار مورد انتظار است، نه یک اشکال. `expandUnchanged` فقط زمانی اعمال می‌شود که زمینهٔ قابل‌بازشدن وجود داشته باشد.

### پیمایش چندفایلی

وصله‌هایی که بیش از یک فایل را تغییر می‌دهند، با یک کارت خلاصهٔ فایل‌های تغییرکرده آغاز می‌شوند: تعداد کل `+N` / `-N`، تعداد هر فایل، نشان‌های افزوده‌شده/حذف‌شده/تغییرنام‌یافته و پیوندهای لنگری که به هر فایل می‌پرند. فایل‌های PNG/PDF رندرشده تعدادهای سربرگ هر فایل را حفظ می‌کنند، اما کلیدهای تغییر نمای تعاملی را حذف می‌کنند؛ زیرا این کنترل‌ها در یک فایل ایستا کارایی ندارند.

## مقادیر پیش‌فرض Plugin

مقادیر پیش‌فرض سراسری Plugin را در `~/.openclaw/openclaw.json` تنظیم کنید:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

کلیدهای پشتیبانی‌شدهٔ `defaults`: ‏`fontFamily`، `fontSize`، `lineSpacing`، `layout`، `showLineNumbers`، `diffIndicators`، `wordWrap`، `background`، `theme`، `fileFormat`، `fileQuality`، `fileScale`، `fileMaxWidth`، `mode`، `ttlSeconds`. پارامترهای صریح فراخوانی ابزار این موارد را بازنویسی می‌کنند.

### پیکربندی پایدار نشانی URL نمایشگر

<ParamField path="viewerBaseUrl" type="string">
  مقدار جایگزین تحت مالکیت Plugin برای پیوندهای نمایشگر بازگشتی، هنگامی که فراخوانی ابزار `baseUrl` را ارسال نمی‌کند. باید `http` یا `https` و بدون پرس‌وجو/هش باشد.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## پیکربندی امنیت

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: درخواست‌های غیر-loopback به مسیرهای نمایشگر رد می‌شوند. `true`: اگر مسیر توکن‌دار معتبر باشد، نمایشگرهای راه‌دور مجازند.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## چرخهٔ عمر و ذخیره‌سازی مصنوع

- HTML نمایشگر و فراداده‌ها در پایگاه دادهٔ مشترک `state/openclaw.sqlite` در فضای نام blob افزونهٔ Diffs نگهداری می‌شوند. HTML با gzip فشرده می‌شود؛ SQLite فقط هش SHA-256 توکن تصادفی URL را ذخیره می‌کند، نه خود توکن را.
- فایل‌های PNG/PDF رندرشده به‌صورت نمونه‌های موقت در `$TMPDIR/openclaw-diffs` باقی می‌مانند، زیرا تحویل از طریق کانال به مسیر فایل نیاز دارد. SQLite مالک فرادادهٔ انقضای آن‌هاست؛ هیچ فایل جانبی JSON نوشته نمی‌شود.
- TTL پیش‌فرض آرتیفکت: 30 دقیقه. حداکثر TTL پذیرفته‌شده: 6 ساعت.
- پاک‌سازی پس از هر فراخوانی ایجاد آرتیفکت، در صورت فراهم‌بودن فرصت اجرا می‌شود. ابتدا ردیف‌های منقضی‌شدهٔ SQLite حذف می‌شوند و سپس هر دایرکتوری PNG/PDF متناظر حذف می‌شود.
- یک پیمایش پشتیبان، پوشه‌های موقت بدون ردیف را که بیش از 24 ساعت قدمت دارند حذف می‌کند. کش‌های قدیمی `meta.json`، `file-meta.json` و `viewer.html` وارد یا خوانده نمی‌شوند.

## URL نمایشگر و رفتار شبکه

مسیر نمایشگر: `/plugins/diffs/view/{artifactId}/{token}`

دارایی‌های نمایشگر:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js` (فقط زمانی که diff از یکی از زبان‌های بستهٔ زبانی استفاده می‌کند)

سند نمایشگر این دارایی‌ها را نسبت به URL نمایشگر تفکیک می‌کند، بنابراین پیشوند اختیاری مسیر `baseUrl` به درخواست‌های دارایی نیز منتقل می‌شود.

ترتیب تفکیک URL: ‏`baseUrl` در فراخوانی ابزار (پس از اعتبارسنجی سخت‌گیرانه) -> ‏`viewerBaseUrl` افزونه -> پیش‌فرض loopback یعنی `127.0.0.1`. اگر حالت bind در Gateway برابر `custom` باشد و `gateway.customBindHost` تنظیم شده باشد، به‌جای loopback از آن میزبان استفاده می‌شود.

قواعد `baseUrl`: باید `http://` یا `https://` باشد؛ query و hash رد می‌شوند؛ origin به‌همراه مسیر پایهٔ اختیاری مجاز است.

## مدل امنیتی

<AccordionGroup>
  <Accordion title="سخت‌سازی نمایشگر">
    - به‌طور پیش‌فرض فقط loopback.
    - مسیرهای توکن‌دار نمایشگر با اعتبارسنجی سخت‌گیرانهٔ الگوی شناسه و توکن.
    - سیاست CSP پاسخ نمایشگر: `default-src 'none'`؛ اسکریپت‌ها/دارایی‌ها فقط از خود مبدأ؛ بدون `connect-src` خروجی.
    - محدودسازی خطاهای دسترسی راه دور در صورت فعال‌بودن دسترسی راه دور: 40 خطا در هر 60 ثانیه، قفل‌شدن 60ثانیه‌ای را فعال می‌کند (`429 Too Many Requests`).

  </Accordion>
  <Accordion title="سخت‌سازی رندر فایل">
    - مسیریابی درخواست مرورگر برای ثبت تصویر، به‌طور پیش‌فرض همه‌چیز را رد می‌کند.
    - فقط دارایی‌های محلی نمایشگر از `http://127.0.0.1/plugins/diffs/assets/*` مجاز هستند.
    - درخواست‌های شبکهٔ خارجی مسدود می‌شوند.

  </Accordion>
</AccordionGroup>

## الزامات مرورگر برای حالت فایل

`mode: "file"` و `mode: "both"` به مرورگری سازگار با Chromium نیاز دارند.

ترتیب تفکیک:

<Steps>
  <Step title="پیکربندی">
    `browser.executablePath` در پیکربندی OpenClaw.
  </Step>
  <Step title="متغیرهای محیطی">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="راهکار جایگزین پلتفرم">
    مسیرهای رایج نصب و جست‌وجوهای `PATH` برای Chrome، Chromium، Edge و Brave.
  </Step>
</Steps>

متن رایج خطا: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`. برای رفع آن، Chrome، Chromium، Edge یا Brave را نصب کنید یا یکی از گزینه‌های مسیر فایل اجرایی بالا را تنظیم کنید.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="خطاهای اعتبارسنجی ورودی">
    - `Provide patch or both before and after text.` -- هر دو `before` و `after` را وارد کنید، یا `patch` را ارائه دهید.
    - `Provide either patch or before/after input, not both.` -- حالت‌های ورودی را با هم ترکیب نکنید.
    - `Invalid baseUrl: ...` -- از origin از نوع `http(s)` با مسیر اختیاری و بدون query/hash استفاده کنید.
    - `{field} exceeds maximum size (...)` -- اندازهٔ payload را کاهش دهید.
    - ردشدن patch بزرگ -- تعداد فایل‌های patch یا مجموع خطوط را کاهش دهید.

  </Accordion>
  <Accordion title="دسترسی‌پذیری نمایشگر">
    - URL نمایشگر به‌طور پیش‌فرض به `127.0.0.1` تفکیک می‌شود.
    - برای دسترسی راه دور، یا `viewerBaseUrl` افزونه را تنظیم کنید، در هر فراخوانی `baseUrl` را ارسال کنید، یا از `gateway.bind=custom` همراه با `gateway.customBindHost` استفاده کنید.
    - اگر `gateway.trustedProxies` شامل loopback برای پراکسی روی همان میزبان باشد (برای مثال Tailscale Serve)، درخواست‌های مستقیم نمایشگر روی loopback که هدرهای ارسال‌شدهٔ IP کارخواه را ندارند، بنا بر طراحی به‌صورت امن رد می‌شوند.
    - برای این توپولوژی پراکسی، برای پیوست `mode: "file"`/`"both"` را ترجیح دهید، یا برای پیوند نمایشگر قابل‌اشتراک، عمداً `security.allowRemoteViewer` را به‌همراه `viewerBaseUrl` افزونه/یک `baseUrl` پراکسی فعال کنید.
    - ‏`security.allowRemoteViewer` را فقط زمانی فعال کنید که دسترسی خارجی به نمایشگر مدنظر است.

  </Accordion>
  <Accordion title="ردیف خطوط تغییریافته‌نشده دکمهٔ بازکردن ندارد">
    این رفتار برای ورودی patch فاقد محتوای زمینه‌ای قابل‌گسترش مورد انتظار است؛ خطای نمایشگر نیست.
  </Accordion>
  <Accordion title="آرتیفکت یافت نشد">
    - آرتیفکت به‌دلیل TTL منقضی شده است.
    - توکن یا مسیر تغییر کرده است.
    - پاک‌سازی داده‌های قدیمی را حذف کرده است.

  </Accordion>
</AccordionGroup>

## راهنمای عملیاتی

- برای بازبینی‌های تعاملی محلی در canvas، ‏`mode: "view"` را ترجیح دهید.
- برای کانال‌های چت خروجی که به پیوست نیاز دارند، ‏`mode: "file"` را ترجیح دهید.
- ‏`allowRemoteViewer` را غیرفعال نگه دارید، مگر اینکه استقرار شما به URLهای راه دور نمایشگر نیاز داشته باشد.
- برای diffهای حساس، یک `ttlSeconds` کوتاه و صریح تنظیم کنید.
- در صورت عدم نیاز، از ارسال اطلاعات محرمانه در ورودی diff خودداری کنید.
- اگر کانال شما تصاویر را به‌شدت فشرده می‌کند (برای مثال Telegram یا WhatsApp)، خروجی PDF را ترجیح دهید (`fileFormat: "pdf"`).

<Note>
موتور رندر diff با پشتیبانی [Diffs](https://diffs.com).
</Note>

## مرتبط

- [مرورگر](/fa/tools/browser)
- [Pluginها](/fa/tools/plugin)
- [نمای کلی ابزارها](/fa/tools)
