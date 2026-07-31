---
read_when:
    - می‌خواهید از تولید ویدئوی Alibaba Wan در OpenClaw استفاده کنید
    - برای تولید ویدئو باید کلید API مربوط به Model Studio یا DashScope را تنظیم کنید
summary: تولید ویدئو با Wan در Alibaba Model Studio در OpenClaw
title: استودیوی مدل Alibaba
x-i18n:
    generated_at: "2026-07-27T15:47:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb74e2361500ccfbc5d3c4f2d08c3b62aacba8c79c704570952e2181abacf9fb
    source_path: providers/alibaba.md
    workflow: 16
---

پلاگین همراه `alibaba` یک ارائه‌دهندهٔ تولید ویدئو را برای مدل‌های Wan در Alibaba Model Studio (نام بین‌المللی DashScope) ثبت می‌کند. این پلاگین به‌طور پیش‌فرض فعال است؛ فقط یک کلید API لازم است.

| ویژگی            | مقدار                                                                           |
| ---------------- | ------------------------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده | `alibaba`                                                                       |
| پلاگین           | همراه، `enabledByDefault: true`                                               |
| متغیرهای محیطی احراز هویت | `MODELSTUDIO_API_KEY` → `DASHSCOPE_API_KEY` → `QWEN_API_KEY` (اولین تطابق برنده است) |
| پرچم راه‌اندازی اولیه | `--auth-choice alibaba-model-studio-api-key`                                    |
| پرچم مستقیم CLI  | `--alibaba-model-studio-api-key <key>`                                          |
| مدل پیش‌فرض      | `alibaba/wan2.6-t2v`                                                            |
| URL پایهٔ پیش‌فرض | `https://dashscope-intl.aliyuncs.com`                                           |

## شروع به کار

<Steps>
  <Step title="تنظیم کلید API">
    کلید را از طریق راه‌اندازی اولیه برای ارائه‌دهندهٔ `alibaba` ذخیره کنید:

    ```bash
    openclaw onboard --auth-choice alibaba-model-studio-api-key
    ```

    یا کلید را مستقیماً وارد کنید:

    ```bash
    openclaw onboard --alibaba-model-studio-api-key <your-key>
    ```

    یا پیش از راه‌اندازی Gateway، یکی از متغیرهای محیطی پذیرفته‌شده را صادر کنید:

    ```bash
    export MODELSTUDIO_API_KEY=sk-...
    # یا DASHSCOPE_API_KEY=...
    # یا QWEN_API_KEY=...
    ```

  </Step>
  <Step title="تنظیم مدل پیش‌فرض ویدئو">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "alibaba/wan2.6-t2v",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="بررسی پیکربندی ارائه‌دهنده">
    ```bash
    openclaw models list --provider alibaba
    ```

    این فهرست شامل هر پنج مدل همراه Wan است. اگر `MODELSTUDIO_API_KEY` قابل حل نباشد، `openclaw models status --json` اعتبارنامهٔ مفقود را در بخش `auth.unusableProfiles` گزارش می‌کند.

  </Step>
</Steps>

<Note>
  پلاگین Alibaba و [پلاگین Qwen](/fa/providers/qwen) هر دو در برابر DashScope احراز هویت می‌کنند و متغیرهای محیطی هم‌پوشان را می‌پذیرند. برای سطح اختصاصی ویدئوی Wan از شناسه‌های مدل `alibaba/...` استفاده کنید؛ برای گفت‌وگو، تعبیه‌سازی یا درک رسانه‌ای Qwen از شناسه‌های `qwen/...` استفاده کنید.
</Note>

## مدل‌های داخلی Wan

| ارجاع مدل                  | حالت                      |
| -------------------------- | ------------------------- |
| `alibaba/wan2.6-t2v`       | متن به ویدئو (پیش‌فرض)   |
| `alibaba/wan2.6-i2v`       | تصویر به ویدئو            |
| `alibaba/wan2.6-r2v`       | مرجع به ویدئو             |
| `alibaba/wan2.6-r2v-flash` | مرجع به ویدئو (سریع) |
| `alibaba/wan2.7-r2v`       | مرجع به ویدئو             |

## قابلیت‌ها و محدودیت‌ها

هر سه حالت، تعداد ویدئوی یکسان در هر درخواست و سقف مدت یکسانی دارند؛ فقط شکل ورودی متفاوت است.

| حالت               | حداکثر ویدئوهای خروجی | حداکثر تصاویر ورودی | حداکثر ویدئوهای ورودی | حداکثر مدت | کنترل‌های پشتیبانی‌شده                                        |
| ------------------ | ----------------- | ---------------- | ---------------- | ------------ | --------------------------------------------------------- |
| متن به ویدئو      | 1                 | نامرتبط              | نامرتبط              | 10 s         | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| تصویر به ویدئو     | 1                 | 1                | نامرتبط              | 10 s         | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |
| مرجع به ویدئو | 1                 | نامرتبط              | 4                | 10 s         | `size`, `aspectRatio`, `resolution`, `audio`, `watermark` |

درخواستی که `durationSeconds` را حذف کند، مقدار پیش‌فرض پذیرفته‌شدهٔ DashScope یعنی **5 ثانیه** را دریافت می‌کند. برای افزایش مدت تا 10 s، مقدار `durationSeconds` را به‌صراحت در [ابزار تولید ویدئو](/fa/tools/video-generation) تنظیم کنید.

<Warning>
  ورودی‌های تصویر و ویدئوی مرجع باید URLهای راه دور `http(s)` باشند؛ حالت‌های مرجع DashScope مسیرهای فایل محلی را رد می‌کنند. ابتدا آن‌ها را در فضای ذخیره‌سازی شیء بارگذاری کنید، یا از جریان [ابزار رسانه](/fa/tools/media-overview) استفاده کنید که از قبل یک URL عمومی تولید می‌کند.
</Warning>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="بازنویسی URL پایهٔ DashScope">
    ارائه‌دهنده به‌طور پیش‌فرض از نقطهٔ پایانی بین‌المللی DashScope استفاده می‌کند. برای هدف‌گیری نقطهٔ پایانی منطقهٔ چین:

    ```json5
    {
      models: {
        providers: {
          alibaba: {
            baseUrl: "https://dashscope.aliyuncs.com",
          },
        },
      },
    }
    ```

    ارائه‌دهنده پیش از ساخت URLهای وظیفهٔ AIGC، اسلش‌های انتهایی را حذف می‌کند.

  </Accordion>

  <Accordion title="اولویت متغیرهای محیطی احراز هویت">
    OpenClaw کلید API مربوط به Alibaba را با این ترتیب از متغیرهای محیطی حل می‌کند و نخستین مقدار غیرخالی را برمی‌گزیند:

    1. `MODELSTUDIO_API_KEY`
    2. `DASHSCOPE_API_KEY`
    3. `QWEN_API_KEY`

    ورودی‌های پیکربندی‌شدهٔ `auth.profiles` (که از طریق `openclaw models auth login` تنظیم شده‌اند) بر حل متغیرهای محیطی اولویت دارند. برای چرخش پروفایل، دورهٔ انتظار و سازوکارهای بازنویسی، به [پروفایل‌های احراز هویت در پرسش‌های متداول مدل‌ها](/fa/help/faq-models#auth-profiles-what-they-are-and-how-to-manage-them) مراجعه کنید.

  </Accordion>

  <Accordion title="ارتباط با پلاگین Qwen">
    هر دو پلاگین همراه با DashScope ارتباط برقرار می‌کنند و کلیدهای API هم‌پوشان را می‌پذیرند. استفاده کنید از:

    - `alibaba/wan*.*` شناسه‌ها برای ارائه‌دهندهٔ اختصاصی ویدئوی Wan که در این صفحه مستند شده است.
    - `qwen/*` شناسه‌ها برای گفت‌وگو، تعبیه‌سازی و درک رسانه‌ای Qwen (به [Qwen](/fa/providers/qwen) مراجعه کنید).

    تنظیم یک‌بارهٔ `MODELSTUDIO_API_KEY` هر دو پلاگین را احراز هویت می‌کند، زیرا فهرست متغیرهای محیطی احراز هویت عمداً هم‌پوشانی دارد؛ راه‌اندازی اولیهٔ جداگانهٔ هر پلاگین لازم نیست.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="تولید ویدئو" href="/fa/tools/video-generation" icon="video">
    پارامترهای مشترک ابزار ویدئو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="Qwen" href="/fa/providers/qwen" icon="microchip">
    راه‌اندازی گفت‌وگو، تعبیه‌سازی و درک رسانه‌ای Qwen با همان احراز هویت DashScope.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    پیش‌فرض‌های عامل و پیکربندی مدل.
  </Card>
  <Card title="پرسش‌های متداول مدل‌ها" href="/fa/help/faq-models" icon="circle-question">
    پروفایل‌های احراز هویت، جابه‌جایی مدل‌ها و رفع خطاهای «بدون پروفایل».
  </Card>
</CardGroup>
