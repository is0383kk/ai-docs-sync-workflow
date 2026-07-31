---
read_when:
    - می‌خواهید از مدل‌های متن‌باز میزبانی‌شده در Bedrock Mantle با OpenClaw استفاده کنید
    - به نقطه پایانی سازگار با OpenAI در Mantle برای GPT-OSS، Qwen، Kimi یا GLM نیاز دارید
    - می‌خواهید از Claude Opus 5، Sonnet 5 یا Mythos 5 از طریق Amazon Bedrock Mantle استفاده کنید
summary: از مدل‌های سازگار با OpenAI و Claude Messages در Amazon Bedrock Mantle همراه با OpenClaw استفاده کنید
title: Amazon Bedrock Mantle
x-i18n:
    generated_at: "2026-07-27T14:28:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3d2b49120560c4466aff217c3101fab057dd87c1c501f1b8eb94d74f62bd1037
    source_path: providers/bedrock-mantle.md
    workflow: 16
---

OpenClaw شامل ارائه‌دهندهٔ همراه **Amazon Bedrock Mantle** است که به نقطهٔ پایانی سازگار با OpenAI در Mantle متصل می‌شود. Mantle مدل‌های متن‌باز و
شخص ثالث (GPT-OSS، Qwen، Kimi، GLM و موارد مشابه) را از طریق یک سطح استاندارد
`/v1/chat/completions` که زیرساخت Bedrock پشتیبان آن است، میزبانی می‌کند. Mantle همچنین
مدل‌های Anthropic Claude را از طریق یک مسیر Anthropic Messages ارائه می‌کند.

| ویژگی       | مقدار                                                                                  |
| -------------- | -------------------------------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده    | `amazon-bedrock-mantle`                                                                |
| API            | `openai-completions` برای مدل‌های OSS کشف‌شده، `anthropic-messages` برای مدل‌های Claude |
| احراز هویت           | `AWS_BEARER_TOKEN_BEDROCK` صریح یا تولید توکن حامل از زنجیرهٔ اعتبارنامهٔ IAM    |
| منطقهٔ پیش‌فرض | `us-east-1` (با `AWS_REGION` یا `AWS_DEFAULT_REGION` بازنویسی کنید)                       |

## شروع به کار

روش احراز هویت ترجیحی خود را انتخاب و مراحل راه‌اندازی را دنبال کنید.

<Tabs>
  <Tab title="توکن حامل صریح">
    **مناسب برای:** محیط‌هایی که از قبل توکن حامل Mantle دارید.

    <Steps>
      <Step title="تنظیم توکن حامل روی میزبان Gateway">
        ```bash
        export AWS_BEARER_TOKEN_BEDROCK="..."
        ```

        در صورت تمایل، یک منطقه تنظیم کنید (پیش‌فرض `us-east-1` است):

        ```bash
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="بررسی کشف‌شدن مدل‌ها">
        ```bash
        openclaw models list
        ```

        مدل‌های کشف‌شده زیر ارائه‌دهندهٔ `amazon-bedrock-mantle` نمایش داده می‌شوند. مگر
        اینکه بخواهید پیش‌فرض‌ها را بازنویسی کنید، پیکربندی دیگری لازم نیست.
      </Step>
    </Steps>

  </Tab>

  <Tab title="اعتبارنامه‌های IAM">
    **مناسب برای:** استفاده از اعتبارنامه‌های سازگار با AWS SDK (پیکربندی مشترک، SSO، هویت وب، نقش‌های نمونه یا وظیفه).

    <Steps>
      <Step title="پیکربندی اعتبارنامه‌های AWS روی میزبان Gateway">
        هر منبع احراز هویت سازگار با AWS SDK قابل استفاده است:

        ```bash
        export AWS_PROFILE="default"
        export AWS_REGION="us-west-2"
        ```
      </Step>
      <Step title="بررسی کشف‌شدن مدل‌ها">
        ```bash
        openclaw models list
        ```

        OpenClaw به‌طور خودکار از زنجیرهٔ اعتبارنامه، یک توکن حامل Mantle تولید می‌کند.
      </Step>
    </Steps>

    <Tip>
    وقتی `AWS_BEARER_TOKEN_BEDROCK` تنظیم نشده باشد، OpenClaw با استفاده از زنجیرهٔ اعتبارنامهٔ پیش‌فرض AWS، شامل اعتبارنامه‌ها/نمایه‌های پیکربندی مشترک، SSO، هویت وب و نقش‌های نمونه یا وظیفه، توکن حامل را برای شما ایجاد می‌کند.
    </Tip>

  </Tab>
</Tabs>

## کشف خودکار مدل

وقتی `AWS_BEARER_TOKEN_BEDROCK` تنظیم شده باشد، OpenClaw مستقیماً از آن استفاده می‌کند. در غیر این صورت،
OpenClaw تلاش می‌کند از زنجیرهٔ اعتبارنامهٔ پیش‌فرض AWS یک توکن حامل Mantle
تولید کند. سپس با پرس‌وجو از نقطهٔ پایانی `/v1/models` منطقه،
مدل‌های دردسترس Mantle را کشف می‌کند.

| رفتار          | جزئیات                                                                               |
| ----------------- | ------------------------------------------------------------------------------------ |
| حافظهٔ نهان کشف   | نتایج برای هر منطقه به‌مدت 1 ساعت در حافظهٔ نهان می‌مانند؛ شکست واکشی، آخرین نتیجهٔ ذخیره‌شده را برمی‌گرداند |
| نوسازی توکن IAM | هر 2 ساعت، به‌صورت ذخیره‌شده برای هر منطقه                                                     |

برای فعال نگه‌داشتن Plugin مربوط به Mantle و در عین حال جلوگیری از کشف خودکار و
تولید توکن حامل IAM، کلید کشف متعلق به Plugin را غیرفعال کنید:

```bash
openclaw config set plugins.entries.amazon-bedrock-mantle.config.discovery.enabled false
```

<Note>
توکن حامل همان `AWS_BEARER_TOKEN_BEDROCK` است که ارائه‌دهندهٔ استاندارد [Amazon Bedrock](/fa/providers/bedrock) استفاده می‌کند.
</Note>

### مناطق پشتیبانی‌شده

`us-east-1`، `us-east-2`، `us-west-2`، `ap-northeast-1`،
`ap-south-1`، `ap-southeast-3`، `eu-central-1`، `eu-west-1`، `eu-west-2`،
`eu-south-1`، `eu-north-1`، `sa-east-1`.

## پیکربندی دستی

اگر پیکربندی صریح را به کشف خودکار ترجیح می‌دهید:

```json5
{
  models: {
    providers: {
      "amazon-bedrock-mantle": {
        baseUrl: "https://bedrock-mantle.us-east-1.api.aws/v1",
        api: "openai-completions",
        auth: "api-key",
        apiKey: "env:AWS_BEARER_TOKEN_BEDROCK",
        models: [
          {
            id: "gpt-oss-120b",
            name: "GPT-OSS 120B",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32000,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

یک فهرست صریح و غیرخالی `models` مرجع نهایی است و همهٔ
ردیف‌های کشف‌شده، از جمله ردیف‌های Claude در ادامه را جایگزین می‌کند. برای حفظ
کاتالوگ خودکار Mantle، `models` را حذف کنید، یا ورودی‌های کامل مدل‌های Claude موردنظر
خود را بگنجانید.

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="پشتیبانی از استدلال">
    پشتیبانی از استدلال از شناسه‌های مدلی استنباط می‌شود که شامل الگوهایی مانند
    `thinking`، `reasoner`، `reasoning`، `deepseek.r`، `gpt-oss-120b` یا
    `gpt-oss-safeguard-120b` هستند. OpenClaw هنگام کشف، `reasoning: true` را برای
    مدل‌های منطبق به‌طور خودکار تنظیم می‌کند.
  </Accordion>

  <Accordion title="دردسترس‌نبودن نقطهٔ پایانی">
    اگر نقطهٔ پایانی Mantle دردسترس نباشد، هیچ مدلی برنگرداند یا رفع توکن حامل
    ناموفق باشد، کشف نتیجه‌ای خالی برمی‌گرداند و ارائه‌دهندهٔ ضمنی
    نادیده گرفته می‌شود. OpenClaw خطا نمی‌دهد؛ سایر ارائه‌دهندگان پیکربندی‌شده
    به‌طور عادی به کار خود ادامه می‌دهند.
  </Accordion>

  <Accordion title="Claude از طریق مسیر Anthropic Messages">
    وقتی کشف خودکار مالک فهرست مدل‌ها باشد، OpenClaw پس از یک جست‌وجوی موفق، فارغ از اینکه
    `/v1/models` چه چیزی برمی‌گرداند، پنج مدل Claude را اضافه می‌کند:
    `amazon-bedrock-mantle/anthropic.claude-opus-5` (Claude Opus 5)،
    `amazon-bedrock-mantle/anthropic.claude-sonnet-5` (Claude Sonnet 5)،
    `amazon-bedrock-mantle/anthropic.claude-opus-4-7` (Claude Opus 4.7) و
    `amazon-bedrock-mantle/anthropic.claude-mythos-5` (Claude Mythos 5)، به‌علاوهٔ
    `amazon-bedrock-mantle/anthropic.claude-mythos-preview` (پیش‌نمایش Claude Mythos).
    آن‌ها از سطح API ‏`anthropic-messages` استفاده می‌کنند و از طریق
    همان نقطهٔ پایانی سازگار با Anthropic و دارای احراز هویت حامل
    (`<mantle-base>/anthropic`) جریان می‌یابند؛ بنابراین توکن حامل AWS مانند
    کلید API ‏Anthropic تلقی نمی‌شود.

    Claude Opus 5 پنجرهٔ زمینهٔ 1,000,000 توکنی، محدودیت خروجی
    128,000 توکنی، ورودی تصویر و قیمت‌گذاری ورودی/خروجی `$5/$25` را ارائه می‌کند. تفکر تطبیقی
    به‌طور پیش‌فرض `high` است؛ `/think off` تفکر را غیرفعال می‌کند و
    `/think xhigh|max` از سطوح تلاش بومی مدل استفاده می‌کند. OpenClaw پارامترهای
    نمونه‌برداری انتخاب‌شده توسط فراخواننده را حذف می‌کند.

    Claude Sonnet 5 همیشه از تفکر تطبیقی استفاده می‌کند و تلاش آن به‌طور پیش‌فرض `high`
    است. `/think off` و `/think minimal` به `low` نگاشت می‌شوند، زیرا مسیر Mantle
    نمی‌تواند تفکر را غیرفعال کند. OpenClaw همچنین دمای سفارشی را از
    درخواست‌های Sonnet 5 حذف می‌کند.

    دسترسی به Claude Mythos 5 محدود است. این مدل پنجرهٔ زمینهٔ 1,000,000 توکنی
    و محدودیت خروجی 128,000 توکنی ارائه می‌کند، همیشه از تفکر تطبیقی استفاده می‌کند،
    `/think off` و `/think minimal` را به `low` نگاشت می‌کند و پارامترهای
    نمونه‌برداری انتخاب‌شده توسط فراخواننده را حذف می‌کند.

    Claude Mythos Preview همیشه استدلال را درخواست می‌کند و وقتی هیچ سطح
    `/think` تنظیم نشده باشد، تلاش پیش‌فرض آن `high` است
    (`xhigh`/`max` به `high` کاهش و
    `minimal` به `low` افزایش می‌یابد). Opus 4.7 در Mantle بدون
    استدلال ارائه‌شده توسط مدل جریان می‌یابد و OpenClaw پارامتر `temperature` آن را
    حذف می‌کند، زیرا Opus 4.7 در این مسیر بازنویسی نمونه‌برداری را نمی‌پذیرد؛ Mythos
    Preview بازنویسی `temperature` را به‌طور عادی می‌پذیرد.

    یک فهرست صریح و غیرخالی `models.providers["amazon-bedrock-mantle"].models`
    تمام کاتالوگ کشف‌شده را جایگزین می‌کند. وقتی این ردیف‌های داخلی Claude را
    می‌خواهید، آن فهرست را حذف کنید.

  </Accordion>

  <Accordion title="رابطه با ارائه‌دهندهٔ Amazon Bedrock">
    Bedrock Mantle ارائه‌دهنده‌ای جدا از ارائه‌دهندهٔ استاندارد
    [Amazon Bedrock](/fa/providers/bedrock) است. Mantle برای کاتالوگ OSS خود از
    سطح سازگار با OpenAI ‏`/v1` استفاده می‌کند، در حالی که ارائه‌دهندهٔ استاندارد
    Bedrock از API بومی Bedrock Converse استفاده می‌کند.

    هر دو ارائه‌دهنده در صورت وجود، از اعتبارنامهٔ یکسان `AWS_BEARER_TOKEN_BEDROCK`
    استفاده می‌کنند.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="Amazon Bedrock" href="/fa/providers/bedrock" icon="cloud">
    ارائه‌دهندهٔ بومی Bedrock برای Anthropic Claude، Titan و مدل‌های دیگر.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="OAuth و احراز هویت" href="/fa/gateway/authentication" icon="key">
    جزئیات احراز هویت و قواعد استفادهٔ مجدد از اعتبارنامه‌ها.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و روش رفع آن‌ها.
  </Card>
</CardGroup>
