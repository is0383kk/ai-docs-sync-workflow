---
read_when:
    - می‌خواهید OpenClaw را با مدل‌های ابری یا محلی از طریق Ollama اجرا کنید
    - به راهنمای راه‌اندازی و پیکربندی Ollama نیاز دارید
    - برای درک تصاویر، مدل‌های بینایی Ollama را می‌خواهید
summary: اجرای OpenClaw با Ollama (مدل‌های ابری و محلی)
title: Ollama
x-i18n:
    generated_at: "2026-07-27T15:41:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80ae833d006ce307406fac11fe3457809165035a38b7e0a970777baf126cc9cb
    source_path: providers/ollama.md
    workflow: 16
---

OpenClaw با API بومی Ollama (`/api/chat`) ارتباط برقرار می‌کند، نه نقطه پایانی سازگار با OpenAI یعنی
`/v1`. سه حالت پشتیبانی می‌شود:

| حالت          | آنچه استفاده می‌کند                                                                     |
| ------------- | -------------------------------------------------------------------------------- |
| ابری + محلی | یک میزبان Ollama در دسترس که مدل‌های محلی و (در صورت ورود به حساب) مدل‌های `:cloud` را ارائه می‌دهد |
| فقط ابری    | مستقیماً `https://ollama.com`، بدون دیمن محلی                                   |
| فقط محلی    | یک میزبان Ollama در دسترس، فقط مدل‌های محلی                                       |

برای راه‌اندازی صرفاً ابری با شناسه ارائه‌دهنده اختصاصی `ollama-cloud`، به
[Ollama Cloud](/fa/providers/ollama-cloud) مراجعه کنید. وقتی می‌خواهید مسیریابی ابری از یک ارائه‌دهنده محلی `ollama` جدا بماند، از ارجاع‌های `ollama-cloud/<model>`
استفاده کنید.

<Warning>
از نشانی سازگار با OpenAI یعنی `/v1` (`http://host:11434/v1`) استفاده نکنید. این نشانی فراخوانی ابزار را مختل می‌کند و ممکن است مدل‌ها JSON خام فراخوانی ابزار را به‌صورت متن ساده خروجی دهند. از نشانی بومی استفاده کنید: `baseUrl: "http://host:11434"` (بدون `/v1`).
</Warning>

کلید پیکربندی معیار `baseUrl` است. `baseURL` نیز برای
نمونه‌های سبک OpenAI SDK پذیرفته می‌شود، اما پیکربندی جدید باید از `baseUrl` استفاده کند.

## قواعد احراز هویت

<AccordionGroup>
  <Accordion title="میزبان‌های محلی و LAN">
    نشانی‌های Ollama مربوط به loopback، شبکه خصوصی، `.local` و نام میزبان ساده به توکن حامل واقعی نیاز ندارند. OpenClaw برای این موارد از نشانگر `ollama-local` استفاده می‌کند.
  </Accordion>
  <Accordion title="میزبان‌های راه دور و Ollama Cloud">
    میزبان‌های عمومی راه دور و `https://ollama.com` به اعتبارنامه واقعی نیاز دارند: `OLLAMA_API_KEY`، یک نمایه احراز هویت یا `apiKey` ارائه‌دهنده. برای استفاده مستقیم میزبانی‌شده، ارائه‌دهنده `ollama-cloud` را ترجیح دهید.
  </Accordion>
  <Accordion title="شناسه‌های سفارشی ارائه‌دهنده">
    ارائه‌دهنده سفارشی دارای `api: "ollama"` از همین قواعد پیروی می‌کند. برای نمونه، ارائه‌دهنده `ollama-remote` که به یک میزبان خصوصی LAN اشاره دارد می‌تواند از `apiKey: "ollama-local"` استفاده کند؛ زیرعامل‌ها این نشانگر را از طریق هوک ارائه‌دهنده Ollama حل می‌کنند، نه اینکه آن را اعتبارنامه‌ای مفقود تلقی کنند. `memory.search.provider` نیز می‌تواند به یک شناسه سفارشی ارائه‌دهنده اشاره کند تا تعبیه‌ها از همان نقطه پایانی Ollama استفاده کنند.
  </Accordion>
  <Accordion title="نمایه‌های احراز هویت">
    `auth-profiles.json` اعتبارنامه یک شناسه ارائه‌دهنده را ذخیره می‌کند؛ تنظیمات نقطه پایانی (`baseUrl`، `api`، مدل‌ها، سرآیندها و مهلت‌های زمانی) را در `models.providers.<id>` قرار دهید. فایل‌های تخت قدیمی مانند `{ "ollama-windows": { "apiKey": "ollama-local" } }` قالب زمان اجرا نیستند؛ `openclaw doctor --fix` آن‌ها را همراه با تهیه نسخه پشتیبان، به یک نمایه معیار کلید API در `ollama-windows:default` بازنویسی می‌کند. مقدار `baseUrl` در آن فایل قدیمی زائد است و باید به پیکربندی ارائه‌دهنده منتقل شود.
  </Accordion>
  <Accordion title="دامنه تعبیه حافظه">
    احراز هویت حامل برای تعبیه‌های حافظه Ollama به میزبانی محدود است که برای آن تعریف شده است:

    - کلید سطح ارائه‌دهنده فقط به میزبان همان ارائه‌دهنده ارسال می‌شود.
    - `memory.search.remote.apiKey` و بازنویسی‌های هر عامل فقط به میزبان راه دور تعبیه مربوط به خود ارسال می‌شوند.
    - مقدار صرفاً محیطی `OLLAMA_API_KEY` به‌عنوان قرارداد Ollama Cloud در نظر گرفته می‌شود و به‌طور پیش‌فرض به میزبان‌های محلی یا خودمیزبان ارسال نمی‌شود.

  </Accordion>
</AccordionGroup>

## شروع کار

<Tabs>
  <Tab title="آماده‌سازی اولیه (توصیه‌شده)">
    <Steps>
      <Step title="اجرای آماده‌سازی اولیه">
        ```bash
        openclaw onboard
        ```

        **Ollama** را انتخاب کنید، سپس یک حالت برگزینید: **Cloud + Local**، **Cloud only** یا **Local only**.

        در یک راه‌اندازی هدایت‌شده تازه، OpenClaw ابتدا میزبان پیش‌فرض یا پیکربندی‌شده
        Ollama را بررسی می‌کند. یک مدل نصب‌شده تنها زمانی به‌طور خودکار پیشنهاد می‌شود که
        `/api/show` پشتیبانی از ابزار و پنجره زمینه حداقل 16K را تأیید کند؛
        فراداده زمینه مفقود یا کوچک‌تر در مسیر راه‌اندازی دستی باقی می‌ماند. نردبان
        راه‌اندازی مشترک CLI/macOS همچنان پیش از ذخیره، مسیر انتخاب‌شده را با یک
        تکمیل واقعی تأیید می‌کند. این بررسی خودکار هرگز مدلی را دریافت نمی‌کند؛
        اگر مدل نصب‌شده مناسبی وجود نداشته باشد، آماده‌سازی اولیه با انتخاب‌گر
        معمول Ollama ادامه می‌یابد.
      </Step>
      <Step title="انتخاب مدل">
        `Cloud only` برای `OLLAMA_API_KEY` درخواست ورودی می‌کند و پیش‌فرض‌های ابری میزبانی‌شده را پیشنهاد می‌دهد. `Cloud + Local` و `Local only` نشانی پایه Ollama را درخواست می‌کنند، مدل‌های موجود را کشف می‌کنند و اگر مدل محلی انتخاب‌شده موجود نباشد، آن را خودکار دریافت می‌کنند. یک برچسب نصب‌شده `:latest` مانند `gemma4:latest` به‌جای تکرار `gemma4` فقط یک‌بار نمایش داده می‌شود. `Cloud + Local` همچنین بررسی می‌کند که آیا میزبان برای دسترسی ابری وارد حساب شده است.
      </Step>
      <Step title="تأیید">
        ```bash
        openclaw models list --provider ollama
        ```
      </Step>
    </Steps>

    غیرتعاملی:

    ```bash
    openclaw onboard --non-interactive \
      --auth-choice ollama \
      --custom-base-url "http://ollama-host:11434" \
      --custom-model-id "qwen3.5:27b" \
      --accept-risk
    ```

    `--custom-base-url` و `--custom-model-id` اختیاری هستند؛ با حذف آن‌ها از میزبان محلی پیش‌فرض و مدل پیشنهادی `gemma4` استفاده می‌شود.

  </Tab>

  <Tab title="راه‌اندازی دستی">
    <Steps>
      <Step title="نصب و راه‌اندازی Ollama">
        آن را از [ollama.com/download](https://ollama.com/download) دریافت کنید، سپس یک مدل را دریافت کنید:

        ```bash
        ollama pull gemma4
        ```

        برای دسترسی ابری ترکیبی، `ollama signin` را روی همان میزبان اجرا کنید.
      </Step>
      <Step title="تنظیم اعتبارنامه">
        ```bash
        export OLLAMA_API_KEY="ollama-local"    # میزبان محلی/LAN، هر مقداری کار می‌کند
        export OLLAMA_API_KEY="your-real-key"   # فقط https://ollama.com
        ```

        یا در پیکربندی: `openclaw config set models.providers.ollama.apiKey "OLLAMA_API_KEY"`.
      </Step>
      <Step title="انتخاب مدل">
        ```bash
        openclaw models list
        openclaw models set ollama/gemma4
        ```

        یا در پیکربندی:

        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "ollama/gemma4" },
            },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## مدل‌های ابری از طریق میزبان محلی

`Cloud + Local` مدل‌های محلی و `:cloud` را از طریق یک میزبان Ollama در دسترس
مسیریابی می‌کند — این جریان ترکیبی Ollama و حالتی است که هنگام راه‌اندازی،
وقتی هر دو را می‌خواهید، باید انتخاب کنید.

OpenClaw نشانی پایه را درخواست می‌کند، مدل‌های محلی را کشف می‌کند و وضعیت
`ollama signin` را بررسی می‌کند. پس از ورود به حساب، پیش‌فرض‌های میزبانی‌شده
(`kimi-k2.5:cloud`، `minimax-m2.7:cloud`، `glm-5.1:cloud`، `glm-5.2:cloud`) را پیشنهاد می‌دهد. اگر
وارد حساب نشده باشید، راه‌اندازی تا زمان اجرای `ollama signin` فقط محلی باقی می‌ماند.

برای دسترسی صرفاً ابری بدون دیمن محلی، از `openclaw onboard --auth-choice ollama-cloud` استفاده کنید و به [Ollama Cloud](/fa/providers/ollama-cloud) مراجعه کنید — آن مسیر به `ollama signin` یا سرور در حال اجرا نیاز ندارد:

```bash
openclaw onboard --auth-choice ollama-cloud
openclaw models set ollama-cloud/kimi-k2.5:cloud
```

فهرست مدل‌های ابری که هنگام `openclaw onboard` نمایش داده می‌شود، به‌صورت زنده از
`https://ollama.com/api/tags` پر می‌شود و حداکثر 500 ورودی دارد، بنابراین انتخاب‌گر
فهرست میزبانی‌شده فعلی را بازتاب می‌دهد. اگر `ollama.com` در دسترس نباشد یا هنگام
راه‌اندازی هیچ مدلی برنگرداند، OpenClaw به فهرست پیشنهادی کدنویسی‌شده خود بازمی‌گردد تا
آماده‌سازی اولیه همچنان کامل شود.

## کشف مدل (ارائه‌دهنده ضمنی)

وقتی `OLLAMA_API_KEY` (یا یک نمایه احراز هویت) تنظیم شده باشد و نه
`models.providers.ollama` و نه ارائه‌دهنده سفارشی دیگری با `api: "ollama"`
تعریف نشده باشد، OpenClaw مدل‌ها را از `http://127.0.0.1:11434` کشف می‌کند:

| رفتار             | جزئیات                                                                                                                                                                                                                                                                                        |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| پرس‌وجوی فهرست        | `/api/tags`                                                                                                                                                                                                                                                                                   |
| تشخیص قابلیت | خواندن بهترین‌تلاش `/api/show` از `contextWindow`، پارامترهای Modelfile در `num_ctx` و قابلیت‌ها (بینایی/ابزارها/تفکر)                                                                                                                                                                       |
| مدل‌های بینایی        | قابلیت `vision` از `/api/show` مدل را دارای قابلیت پردازش تصویر (`input: ["text", "image"]`) علامت‌گذاری می‌کند                                                                                                                                                                                             |
| تشخیص استدلال  | در صورت وجود، از قابلیت `thinking` در `/api/show` استفاده می‌کند؛ وقتی Ollama قابلیت‌ها را حذف کند، به روش اکتشافی نام (`r1`، `reason`، `reasoning`، `think`) بازمی‌گردد. `glm-5.2:cloud` و `deepseek-v4-flash\|pro:cloud` صرف‌نظر از قابلیت‌های گزارش‌شده همیشه استدلالی تلقی می‌شوند. |
| محدودیت توکن         | مقدار پیش‌فرض `maxTokens` سقف حداکثر توکن Ollama در OpenClaw است                                                                                                                                                                                                                                       |
| هزینه‌ها                | همه هزینه‌ها `0` هستند                                                                                                                                                                                                                                                                             |

```bash
ollama list
openclaw models list
```

تنظیم `models.providers.ollama` با آرایه صریح `models`، یا یک
ارائه‌دهنده سفارشی با `api: "ollama"` و `baseUrl` غیر-loopback،
کشف خودکار را غیرفعال می‌کند؛ سپس مدل‌ها باید به‌صورت دستی تعریف شوند (به
[پیکربندی](#configuration) مراجعه کنید). ورودی `models.providers.ollama` که به
`https://ollama.com` میزبانی‌شده اشاره دارد نیز کشف را رد می‌کند، زیرا مدل‌های Ollama Cloud
توسط ارائه‌دهنده مدیریت می‌شوند. ارائه‌دهندگان سفارشی loopback مانند
`http://127.0.0.2:11434` همچنان محلی محسوب می‌شوند و کشف خودکار را حفظ می‌کنند.

می‌توانید از یک ارجاع کامل مانند `ollama/<pulled-model>:latest` بدون
ورودی دست‌نویس `models.json` استفاده کنید؛ OpenClaw آن را به‌صورت زنده حل می‌کند. برای میزبان‌هایی که
وارد حساب شده‌اند، انتخاب ارجاع فهرست‌نشده `ollama/<model>:cloud` همان
مدل دقیق را با `/api/show` اعتبارسنجی می‌کند و تنها در صورتی آن را به فهرست زمان اجرا
می‌افزاید که Ollama فراداده را تأیید کند — خطاهای تایپی همچنان به‌عنوان مدل ناشناخته شکست می‌خورند.

### آزمون‌های دود

برای یک کاوش متنی محدود که سطح کامل ابزار عامل را رد می‌کند:

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/llama3.2:latest \
    --prompt "دقیقاً با این متن پاسخ دهید: pong" \
    --json
```

برای کاوش سبک مدل بینایی، `--file` را همراه یک تصویر اضافه کنید (PNG/JPEG/WebP پذیرفته می‌شوند؛
فایل‌های غیرتصویری پیش از فراخوانی Ollama رد می‌شوند — برای صوت از
`openclaw infer audio transcribe` استفاده کنید):

```bash
OLLAMA_API_KEY=ollama-local \
  openclaw infer model run \
    --local \
    --model ollama/qwen2.5vl:7b \
    --prompt "این تصویر را در یک جمله توصیف کنید." \
    --file ./photo.jpg \
    --json
```

هیچ‌یک از این مسیرها ابزارهای گفت‌وگو، حافظه یا زمینه نشست را بارگذاری نمی‌کنند. اگر این مسیر موفق شود
اما پاسخ‌های معمول عامل شکست بخورند، مشکل احتمالاً ظرفیت ابزار/عامل مدل است،
نه نقطه پایانی.

انتخاب مدل با `/model ollama/<model>` یک انتخاب دقیق کاربر است: اگر
`baseUrl` پیکربندی‌شده در دسترس نباشد، پاسخ بعدی به‌جای بازگشت بی‌سروصدا به مدل پیکربندی‌شده دیگری، با خطای ارائه‌دهنده
ناموفق می‌شود.

کارهای Cron ایزوله پیش از شروع نوبت عامل یک بررسی ایمنی محلی اضافه می‌کنند:
اگر مدل انتخاب‌شده به یک ارائه‌دهنده Ollama محلی/شبکه خصوصی/`.local`
منتهی شود و `/api/tags` در دسترس نباشد، OpenClaw آن اجرا را با وضعیت
`skipped` و مدل را در متن خطا ثبت می‌کند. این بررسی نقطه پایانی برای هر میزبان
به‌مدت 5 دقیقه ذخیره می‌شود، بنابراین کارهای Cron تکراری در برابر یک دیمن متوقف‌شده، همگی
درخواست‌های ناموفق را آغاز نمی‌کنند.

اعتبارسنجی زنده:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0 \
  pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

برای Ollama Cloud، همان آزمون زنده را به نقطه پایانی میزبانی‌شده هدایت کنید (به‌طور
پیش‌فرض تعبیه‌ها را رد می‌کند؛ با `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` اجبار کنید، زیرا ممکن است
کلید ابری مجوز `/api/embed` را نداشته باشد):

```bash
export OLLAMA_API_KEY='<your-ollama-cloud-api-key>'
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=glm-5.1:cloud \
OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=1 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

برای افزودن مدل، آن را دریافت کنید تا به‌طور خودکار کشف شود:

```bash
ollama pull mistral
```

## استنتاج محلی Node

عامل‌ها می‌توانند یک وظیفه کوتاه را به مدل Ollama روی دسکتاپ جفت‌شده یا
Node سرور واگذار کنند. پرامپت و پاسخ از اتصال احراز هویت‌شده موجود
Gateway/Node عبور می‌کنند؛ درخواست روی نقطه پایانی loopback خود Node برای Ollama
اجرا می‌شود (`http://127.0.0.1:11434`).

<Steps>
  <Step title="راه‌اندازی Ollama روی Node">
    ```bash
    ollama pull qwen3:0.6b
    ollama list
    ```
  </Step>
  <Step title="اتصال میزبان Node">
    ```bash
    openclaw node run \
      --host <gateway-host> \
      --port 18789 \
      --display-name "استنتاج محلی"
    ```

    دستگاه و فرمان‌های Node آن را روی میزبان Gateway تأیید کنید، سپس صحت اتصال را بررسی کنید:

    ```bash
    openclaw devices list
    openclaw devices approve <deviceRequestId>
    openclaw nodes pending
    openclaw nodes approve <nodeRequestId>
    openclaw nodes status --connected
    ```

    نخستین اتصال یا ارتقایی که فرمان‌های Ollama را اضافه می‌کند، ممکن است
    تأیید فرمان Node را فعال کند. اگر Node بدون اعلام
    `ollama.models` و `ollama.chat` متصل شد، دوباره `openclaw nodes pending` را بررسی کنید.

  </Step>
  <Step title="استفاده از آن در یک عامل">
    Plugin همراه Ollama ابزار `node_inference` را ارائه می‌کند. عامل‌ها ابتدا
    `action: "discover"` و سپس `action: "run"` را با یک Node و مدل از
    آن نتیجه فراخوانی می‌کنند (`run` می‌تواند وقتی دقیقاً یک Node توانمند
    متصل است، Node را حذف کند). برای مثال: «مدل‌های Ollama را روی Nodeهای من کشف کن، سپس از
    سریع‌ترین مدل بارگذاری‌شده برای خلاصه‌کردن این متن استفاده کن.»
  </Step>
</Steps>

فرایند کشف، `/api/tags` را می‌خواند، قابلیت‌های `/api/show` را بررسی می‌کند و در صورت
وجود از `/api/ps` استفاده می‌کند تا مدل‌های از قبل بارگذاری‌شده را در اولویت قرار دهد. این فرایند فقط
مدل‌های محلی را برمی‌گرداند که Ollama آن‌ها را دارای قابلیت گفتگو گزارش می‌کند (قابلیت `completion`) —
ردیف‌های Ollama Cloud و مدل‌های صرفاً تعبیه‌ای کنار گذاشته می‌شوند. هر اجرا
تفکر مدل را غیرفعال می‌کند و خروجی را به‌طور پیش‌فرض روی 512 توکن (سقف قطعی 8192) قرار می‌دهد، مگر اینکه
فراخوانی ابزار `maxTokens` متفاوتی درخواست کند؛ برخی مدل‌ها (برای مثال GPT-OSS)
از غیرفعال‌کردن تفکر پشتیبانی نمی‌کنند و ممکن است همچنان توکن‌های استدلال تولید کنند.

برای فعال نگه‌داشتن Ollama روی یک Node بدون در دسترس قرار دادن آن برای عامل‌ها:

```bash
openclaw config set plugins.entries.ollama.config.nodeInference.enabled false
```

Node را راه‌اندازی مجدد کنید (`openclaw node restart`، یا برای نشست پیش‌زمینه
`openclaw node run` را متوقف و دوباره اجرا کنید). Node دیگر `ollama.models` و
`ollama.chat` را اعلام نمی‌کند؛ خود Ollama و ارائه‌دهنده Ollama در Gateway تحت تأثیر قرار نمی‌گیرند.
برای فعال‌سازی مجدد، مقدار را به `true` بازگردانید و راه‌اندازی مجدد کنید؛ سطح فرمان تغییریافته
ممکن است پس از اتصال مجدد دوباره به تأیید `openclaw nodes pending` نیاز داشته باشد.

فرمان‌های Node را مستقیماً و بدون نوبت عامل بررسی کنید:

```bash
openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.models \
  --params '{}' \
  --invoke-timeout 90000 \
  --timeout 100000

openclaw nodes invoke \
  --node "Local inference" \
  --command ollama.chat \
  --params '{"model":"qwen3:0.6b","prompt":"Reply with exactly: pong","maxTokens":32,"timeoutMs":120000}' \
  --invoke-timeout 130000 \
  --timeout 140000
```

`--invoke-timeout` مدت‌زمانی را محدود می‌کند که Node برای اجرای فرمان در اختیار دارد؛
`--timeout` مدت‌زمان کلی فراخوانی Gateway را محدود می‌کند و باید بزرگ‌تر باشد.

استنتاج محلی Node همیشه از نقطه پایانی loopback خود Node استفاده می‌کند —
از `models.providers.ollama.baseUrl` راه‌دور/ابری پیکربندی‌شده دوباره استفاده نمی‌کند.
فرمان‌های Node به‌طور پیش‌فرض روی میزبان‌های Node در macOS، Linux و Windows
در دسترس‌اند و همچنان تابع سیاست عادی جفت‌سازی/فرمان Node هستند.

## بینایی و توصیف تصویر

Plugin همراه Ollama، آن را به‌عنوان ارائه‌دهنده دارای قابلیت تصویر
برای درک رسانه ثبت می‌کند تا OpenClaw بتواند درخواست‌های صریح توصیف تصویر
و پیش‌فرض‌های پیکربندی‌شده مدل تصویر را از طریق مدل‌های بینایی محلی یا میزبانی‌شده Ollama
مسیریابی کند.

```bash
ollama pull qwen2.5vl:7b
export OLLAMA_API_KEY="ollama-local"
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --json
```

`--model` باید یک ارجاع کامل `<provider/model>` باشد؛ وقتی تنظیم شود، `infer image
describe` به‌جای ردکردن توصیف برای مدل‌هایی
که از قبل از بینایی بومی پشتیبانی می‌کنند، ابتدا آن مدل را امتحان می‌کند. اگر فراخوانی ناموفق باشد، OpenClaw می‌تواند
از طریق `agents.defaults.imageModel.fallbacks` ادامه دهد؛ خطاهای آماده‌سازی فایل/نشانی اینترنتی
پیش از تلاش برای بازگشت جایگزین باعث شکست می‌شوند. برای جریان
درک تصویر OpenClaw و `imageModel` پیکربندی‌شده از `infer image describe` استفاده کنید؛ برای یک بررسی خام چندوجهی با پرامپت سفارشی از `infer model run
--file` استفاده کنید.

برای تبدیل Ollama به ارائه‌دهنده پیش‌فرض درک تصویر برای رسانه ورودی:

```json5
{
  agents: {
    defaults: {
      imageModel: {
        primary: "ollama/qwen2.5vl:7b",
      },
    },
  },
}
```

ارجاع کامل `ollama/<model>` را ترجیح دهید. یک ارجاع ساده `imageModel` مانند
`qwen2.5vl:7b` تنها زمانی به `ollama/qwen2.5vl:7b` نرمال‌سازی می‌شود که دقیقاً همان مدل
در `models.providers.ollama.models` همراه با
`input: ["text", "image"]` فهرست شده باشد و هیچ ارائه‌دهنده تصویر پیکربندی‌شده دیگری
همان شناسه ساده را ارائه نکند؛ در غیر این صورت پیشوند ارائه‌دهنده را صریحاً به‌کار ببرید.

مدل‌های بینایی محلی کند ممکن است نسبت به مدل‌های ابری به زمان پایان طولانی‌تری برای درک تصویر
نیاز داشته باشند و اگر Ollama تلاش کند کل زمینه بینایی اعلام‌شده مدل را
اختصاص دهد، ممکن است روی سخت‌افزار محدود از کار بیفتند. یک زمان پایان قابلیت
تنظیم کنید و `num_ctx` را محدود کنید:

```json5
{
  models: {
    providers: {
      ollama: {
        models: [
          {
            id: "qwen2.5vl:7b",
            name: "qwen2.5vl:7b",
            input: ["text", "image"],
            params: { num_ctx: 2048, keep_alive: "1m" },
          },
        ],
      },
    },
  },
  tools: {
    media: {
      image: {
        timeoutSeconds: 180,
        models: [{ provider: "ollama", model: "qwen2.5vl:7b", timeoutSeconds: 300 }],
      },
    },
  },
}
```

این زمان پایان برای درک تصویر ورودی و ابزار صریح
`image` اعمال می‌شود. `models.providers.ollama.timeoutSeconds` همچنان محافظ
درخواست HTTP زیربنایی Ollama را برای فراخوانی‌های عادی مدل کنترل می‌کند.

اعتبارسنجی زنده:

```bash
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_OLLAMA_IMAGE=1 \
  pnpm test:live -- src/agents/tools/image-tool.ollama.live.test.ts
```

اگر `models.providers.ollama.models` را به‌صورت دستی تعریف می‌کنید، مدل‌های بینایی را
صریحاً مشخص کنید:

```json5
{
  id: "qwen2.5vl:7b",
  name: "qwen2.5vl:7b",
  input: ["text", "image"],
  contextWindow: 128000,
  maxTokens: 8192,
}
```

OpenClaw درخواست‌های توصیف تصویر را برای مدل‌هایی که دارای قابلیت تصویر علامت‌گذاری نشده‌اند
رد می‌کند. با کشف ضمنی، این مورد از قابلیت بینایی `/api/show`
به‌دست می‌آید.

## پیکربندی

<Tabs>
  <Tab title="پایه (کشف ضمنی)">
    ```bash
    export OLLAMA_API_KEY="ollama-local"
    ```

    <Tip>
    اگر `OLLAMA_API_KEY` تنظیم شده باشد، می‌توانید `apiKey` را در ورودی ارائه‌دهنده حذف کنید؛ OpenClaw آن را برای بررسی‌های دسترس‌پذیری تکمیل می‌کند.
    </Tip>

  </Tab>

  <Tab title="صریح (مدل‌های دستی)">
    برای راه‌اندازی ابری میزبانی‌شده، میزبان/درگاه غیراستاندارد، اجبار
    پنجره‌های زمینه یا فهرست‌های کاملاً دستی مدل از پیکربندی صریح استفاده کنید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 128000,
                maxTokens: 8192
              }
            ]
          }
        }
      }
    }
    ```

  </Tab>

  <Tab title="نشانی پایه سفارشی">
    پیکربندی صریح، کشف خودکار را غیرفعال می‌کند، بنابراین مدل‌ها باید فهرست شوند:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            apiKey: "ollama-local",
            baseUrl: "http://ollama-host:11434", // بدون ‎/v1 — نشانی API بومی Ollama
            api: "ollama", // صریح: رفتار بومی فراخوانی ابزار را تضمین می‌کند
            timeoutSeconds: 300, // اختیاری: بودجه اتصال/جریان طولانی‌تر برای مدل‌های محلی سرد
            models: [
              {
                id: "qwen3:32b",
                name: "qwen3:32b",
                params: {
                  keep_alive: "15m", // اختیاری: مدل را بین نوبت‌ها بارگذاری‌شده نگه می‌دارد
                },
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    `/v1` را اضافه نکنید. آن مسیر حالت سازگار با OpenAI را انتخاب می‌کند که در آن فراخوانی ابزار قابل اعتماد نیست.
    </Warning>

  </Tab>
</Tabs>

## دستورالعمل‌های رایج

شناسه‌های مدل را با نام‌های دقیق از `ollama list` یا
`openclaw models list --provider ollama` جایگزین کنید.

<AccordionGroup>
  <Accordion title="مدل محلی با کشف خودکار">
    Ollama روی همان دستگاه Gateway، به‌طور خودکار کشف می‌شود:

    ```bash
    ollama serve
    ollama pull gemma4
    export OLLAMA_API_KEY="ollama-local"
    openclaw models list --provider ollama
    openclaw models set ollama/gemma4
    ```

    مگر اینکه به مدل‌های دستی نیاز داشته باشید، بلوک `models.providers.ollama` را اضافه نکنید.

  </Accordion>

  <Accordion title="میزبان Ollama در LAN با مدل‌های دستی">
    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                reasoning: true,
                input: ["text"],
                params: {
                  num_ctx: 32768,
                  thinking: false,
                  keep_alive: "15m",
                },
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/qwen3.5:9b" },
        },
      },
    }
    ```

    `contextWindow` بودجه زمینه OpenClaw است؛ `params.num_ctx` به
    Ollama ارسال می‌شود. وقتی سخت‌افزار نمی‌تواند زمینه کامل اعلام‌شده مدل را اجرا کند،
    آن‌ها را هم‌راستا نگه دارید.

  </Accordion>

  <Accordion title="فقط Ollama Cloud">
    بدون دیمن محلی، مستقیماً از مدل‌های میزبانی‌شده استفاده کنید:

    ```bash
    export OLLAMA_API_KEY="your-ollama-api-key"
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "https://ollama.com",
            apiKey: "OLLAMA_API_KEY",
            api: "ollama",
            models: [
              {
                id: "kimi-k2.5:cloud",
                name: "kimi-k2.5:cloud",
                reasoning: false,
                input: ["text", "image"],
                contextWindow: 128000,
                maxTokens: 8192,
              },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: { primary: "ollama/kimi-k2.5:cloud" },
        },
      },
    }
    ```

    برای استفاده از شناسه ارائه‌دهنده اختصاصی `ollama-cloud` به‌جای این ساختار، به
    [Ollama Cloud](/fa/providers/ollama-cloud) مراجعه کنید.

  </Accordion>

  <Accordion title="فضای ابری به‌همراه محیط محلی از طریق یک دیمن واردشده به حساب">
    ```bash
    ollama signin
    ollama pull gemma4
    ```

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 300,
            models: [
              { id: "gemma4", name: "gemma4", input: ["text"] },
              { id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text", "image"] },
            ],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama/gemma4",
            fallbacks: ["ollama/kimi-k2.5:cloud"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="چندین میزبان Ollama">
    هنگام اجرای بیش از یک سرور Ollama از شناسه‌های سفارشی ارائه‌دهنده استفاده کنید؛ هرکدام
    میزبان، مدل‌ها، احراز هویت و مهلت زمانی مخصوص خود را دارند.

    ```json5
    {
      models: {
        providers: {
          "ollama-fast": {
            baseUrl: "http://mini.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [{ id: "gemma4", name: "gemma4", input: ["text"] }],
          },
          "ollama-large": {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            api: "ollama",
            timeoutSeconds: 420,
            contextWindow: 131072,
            maxTokens: 16384,
            models: [{ id: "qwen3.5:27b", name: "qwen3.5:27b", input: ["text"] }],
          },
        },
      },
      agents: {
        defaults: {
          model: {
            primary: "ollama-fast/gemma4",
            fallbacks: ["ollama-large/qwen3.5:27b"],
          },
        },
      },
    }
    ```

    OpenClaw پیش از فراخوانی Ollama، پیشوند ارائه‌دهنده فعال را حذف می‌کند (و در صورت نبود آن
    به پیشوند ساده `ollama/` برمی‌گردد)، بنابراین `ollama-large/qwen3.5:27b`
    به‌شکل `qwen3.5:27b` به Ollama می‌رسد.

  </Accordion>

  <Accordion title="پروفایل سبک مدل محلی">
    برخی مدل‌های محلی از عهده پرامپت‌های ساده برمی‌آیند، اما با مجموعه کامل ابزارهای
    عامل مشکل دارند. پیش از تغییر تنظیمات سراسری زمان اجرای عامل، ابزارها و زمینه
    را محدود کنید:

    ```json5
    {
      agents: {
        list: [
          {
            id: "local",
            experimental: {
              localModelLean: true,
            },
            model: { primary: "ollama/gemma4" },
          },
        ],
      },
      models: {
        providers: {
          ollama: {
            baseUrl: "http://127.0.0.1:11434",
            apiKey: "ollama-local",
            api: "ollama",
            contextWindow: 32768,
            models: [
              {
                id: "gemma4",
                name: "gemma4",
                input: ["text"],
                params: { num_ctx: 32768 },
                compat: { supportsTools: false },
              },
            ],
          },
        },
      },
    }
    ```

    فقط زمانی از `compat.supportsTools: false` استفاده کنید که مدل یا سرور به‌طور قابل‌اعتماد
    در طرح‌واره‌های ابزار شکست می‌خورد — این گزینه قابلیت‌های عامل را با پایداری معاوضه می‌کند.
    `localModelLean` ابزارهای سنگین مرورگر، cron، پیام، تولید رسانه،
    صوت و PDF را، مگر در صورت نیاز صریح، از سطح مستقیم عامل حذف می‌کند
    و کاتالوگ‌های بزرگ‌تر را پشت جست‌وجوی ابزار قرار می‌دهد. این گزینه زمینه زمان اجرای Ollama
    یا حالت تفکر آن را تغییر نمی‌دهد. برای مدل‌های کوچک متفکر به‌سبک Qwen که وارد حلقه می‌شوند
    یا بودجه خود را صرف استدلال پنهان می‌کنند، آن را با `params.num_ctx` و
    `params.thinking: false` همراه کنید.

  </Accordion>
</AccordionGroup>

### انتخاب مدل

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "ollama/gpt-oss:20b",
        fallbacks: ["ollama/llama3.3", "ollama/qwen2.5-coder:32b"],
      },
    },
  },
}
```

شناسه‌های سفارشی ارائه‌دهنده نیز به همین شیوه کار می‌کنند: برای ارجاعی که از پیشوند
ارائه‌دهنده فعال استفاده می‌کند، مانند `ollama-spark/qwen3:32b`، OpenClaw پیش از
فراخوانی Ollama آن پیشوند را حذف کرده و `qwen3:32b` را ارسال می‌کند.

برای مدل‌های محلی کند، پیش از افزایش مهلت زمانی کل زمان اجرای عامل، تنظیمات
محدوده ارائه‌دهنده را ترجیح دهید:

```json5
{
  models: {
    providers: {
      ollama: {
        timeoutSeconds: 300,
        models: [
          {
            id: "gemma4:26b",
            name: "gemma4:26b",
            params: { keep_alive: "15m" },
          },
        ],
      },
    },
  },
}
```

`timeoutSeconds` کل درخواست HTTP مدل را پوشش می‌دهد: برقراری اتصال، سرآیندها،
جریان بدنه و لغو کلی واکشی محافظت‌شده. در درخواست‌های بومی `/api/chat`،
مقدار `params.keep_alive` به‌صورت `keep_alive` در سطح بالا ارسال می‌شود؛ وقتی
زمان بارگذاری نوبت نخست گلوگاه است، آن را برای هر مدل تنظیم کنید.

### تأیید سریع

```bash
# دیمن Ollama برای این دستگاه قابل‌مشاهده است
curl http://127.0.0.1:11434/api/tags

# کاتالوگ OpenClaw و مدل انتخاب‌شده
openclaw models list --provider ollama
openclaw models status

# آزمون دود مستقیم مدل
openclaw infer model run \
  --model ollama/gemma4 \
  --prompt "دقیقاً با این متن پاسخ دهید: ok"
```

برای میزبان‌های راه‌دور، `127.0.0.1` را با میزبان `baseUrl` جایگزین کنید. اگر `curl`
کار می‌کند اما OpenClaw کار نمی‌کند، بررسی کنید که آیا Gateway روی دستگاه،
کانتینر یا حساب سرویس دیگری اجرا می‌شود.

## جست‌وجوی وب Ollama

OpenClaw، **جست‌وجوی وب Ollama** را به‌عنوان ارائه‌دهنده `web_search` همراه خود دارد.

| ویژگی       | جزئیات                                                                                                                                                     |
| ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| میزبان      | در صورت تنظیم، `models.providers.ollama.baseUrl`؛ در غیر این صورت `http://127.0.0.1:11434`؛ مقدار `https://ollama.com` مستقیماً از API میزبانی‌شده استفاده می‌کند                          |
| احراز هویت  | برای میزبان محلی واردشده به حساب، بدون کلید؛ برای جست‌وجوی مستقیم `https://ollama.com` یا میزبان‌های محافظت‌شده با احراز هویت، `OLLAMA_API_KEY` یا احراز هویت پیکربندی‌شده ارائه‌دهنده |
| الزام       | میزبان‌های محلی/خودمیزبان باید در حال اجرا باشند و با `ollama signin` وارد حساب شده باشند؛ جست‌وجوی میزبانی‌شده مستقیم به `baseUrl: "https://ollama.com"` به‌همراه یک کلید API واقعی نیاز دارد |

آن را هنگام `openclaw onboard` یا `openclaw configure --section web` انتخاب کنید، یا این مقدار را تنظیم کنید:

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

برای جست‌وجوی میزبانی‌شده مستقیم از طریق Ollama Cloud:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
        api: "ollama",
        models: [{ id: "kimi-k2.5:cloud", name: "kimi-k2.5:cloud", input: ["text"] }],
      },
    },
  },
  tools: {
    web: {
      search: { provider: "ollama" },
    },
  },
}
```

برای یک میزبان خودمیزبان، OpenClaw ابتدا پروکسی محلی `/api/experimental/web_search`
را امتحان می‌کند و سپس به مسیر میزبانی‌شده `/api/web_search` روی همان میزبان برمی‌گردد؛
یک دیمن محلی واردشده به حساب معمولاً از طریق پروکسی محلی پاسخ می‌دهد. فراخوانی‌های مستقیم
`https://ollama.com` همیشه از نقطه پایانی میزبانی‌شده `/api/web_search` استفاده می‌کنند.

<Note>
برای راه‌اندازی و رفتار کامل، به [جست‌وجوی وب Ollama](/fa/tools/ollama-search) مراجعه کنید.
</Note>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="حالت قدیمی سازگار با OpenAI">
    <Warning>
    **فراخوانی ابزار در این حالت قابل‌اعتماد نیست.** فقط زمانی از آن استفاده کنید که یک پروکسی به قالب OpenAI نیاز دارد و به فراخوانی بومی ابزار وابسته نیستید.
    </Warning>

    برای پروکسی پشت `/v1/chat/completions`، مقدار `api: "openai-completions"`
    را صریحاً تنظیم کنید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: true, // پیش‌فرض: true
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

    این حالت ممکن است از جریان‌دهی و فراخوانی ابزار به‌طور هم‌زمان پشتیبانی نکند؛
    شاید لازم باشد `params: { streaming: false }` را روی مدل تنظیم کنید.

    OpenClaw در این حالت به‌طور پیش‌فرض `options.num_ctx` را تزریق می‌کند تا Ollama
    بی‌سروصدا به زمینه 4096 توکنی برنگردد. اگر پروکسی شما فیلدهای ناشناخته
    `options` را رد می‌کند، آن را غیرفعال کنید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434/v1",
            api: "openai-completions",
            injectNumCtxForOpenAICompat: false,
            apiKey: "ollama-local",
            models: [...]
          }
        }
      }
    }
    ```

  </Accordion>

  <Accordion title="پنجره‌های زمینه">
    برای مدل‌های کشف‌شده خودکار، OpenClaw از پنجره زمینه‌ای استفاده می‌کند که `/api/show`
    گزارش می‌دهد، از جمله مقادیر بزرگ‌تر `PARAMETER num_ctx` از Modelfileهای
    سفارشی؛ در غیر این صورت به پنجره زمینه پیش‌فرض Ollama در OpenClaw برمی‌گردد.

    مقادیر `contextWindow`، `contextTokens` و `maxTokens` در سطح ارائه‌دهنده،
    پیش‌فرض‌های همه مدل‌های زیرمجموعه آن ارائه‌دهنده را تعیین می‌کنند و می‌توان آن‌ها را برای هر
    مدل بازنویسی کرد. `contextWindow` بودجه پرامپت/Compaction خود OpenClaw است. درخواست‌های بومی
    `/api/chat`، مگر اینکه `params.num_ctx` را صریحاً تنظیم کنید، `options.num_ctx`
    را تنظیم‌نشده باقی می‌گذارند؛ بنابراین Ollama پیش‌فرض خود مدل،
    `OLLAMA_CONTEXT_LENGTH` یا پیش‌فرض مبتنی بر VRAM را اعمال می‌کند؛ مقادیر نامعتبر، صفر، منفی
    یا نامتناهی `params.num_ctx` نادیده گرفته می‌شوند. اگر یک پیکربندی قدیمی فقط از
    `contextWindow`/`maxTokens` برای اجبار زمینه درخواست بومی استفاده می‌کرد،
    `openclaw doctor --fix` را اجرا کنید تا آن مقادیر در `params.num_ctx` کپی شوند. آداپتور
    سازگار با OpenAI همچنان به‌طور پیش‌فرض `options.num_ctx` را از
    `params.num_ctx` یا `contextWindow` پیکربندی‌شده تزریق می‌کند؛ اگر سرویس بالادستی
    `options` را رد می‌کند، آن را با `injectNumCtxForOpenAICompat: false` غیرفعال کنید.

    ورودی‌های مدل بومی همچنین گزینه‌های رایج زمان اجرای Ollama را زیر
    `params` می‌پذیرند که به‌صورت `/api/chat` `options` بومی ارسال می‌شوند: `num_keep`، `seed`،
    `num_predict`، `top_k`، `top_p`، `min_p`، `typical_p`، `repeat_last_n`،
    `temperature`، `repeat_penalty`، `presence_penalty`، `frequency_penalty`،
    `stop`، `num_batch`، `num_gpu`، `main_gpu`، `use_mmap` و `num_thread`.
    چند کلید (`format`، `keep_alive`، `truncate`، `shift`) به‌جای قرار گرفتن زیر
    `options`، به‌صورت فیلدهای سطح بالای درخواست ارسال می‌شوند. OpenClaw فقط
    همین کلیدهای درخواست Ollama را ارسال می‌کند، بنابراین پارامترهای مختص زمان اجرا مانند
    `streaming` هرگز به Ollama ارسال نمی‌شوند. برای تنظیم `think` در سطح بالا
    از `params.think` (یا `params.thinking`) استفاده کنید؛ `false` تفکر در سطح API
    را برای مدل‌های متفکر به‌سبک Qwen غیرفعال می‌کند.

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            models: [
              {
                id: "llama3.3",
                contextWindow: 131072,
                maxTokens: 65536,
                params: {
                  num_ctx: 32768,
                  temperature: 0.7,
                  top_p: 0.9,
                  thinking: false,
                },
              }
            ]
          }
        }
      }
    }
    ```

    تنظیم به‌ازای هر مدل `agents.defaults.models["ollama/<model>"].params.num_ctx` نیز
    کار می‌کند؛ اگر هر دو تنظیم شده باشند، ورودی صریح مدل ارائه‌دهنده اولویت دارد.

  </Accordion>

  <Accordion title="کنترل تفکر">
    OpenClaw تفکر را به همان شکلی که Ollama انتظار دارد ارسال می‌کند: `think` در سطح بالا، نه
    `options.think`. مدل‌های شناسایی‌شده خودکار که `/api/show` آن‌ها قابلیت
    `thinking` را گزارش می‌کند، گزینه‌های `/think low`، `/think medium`، `/think high`
    و `/think max` را ارائه می‌دهند؛ مدل‌های بدون تفکر فقط `/think off` را ارائه می‌دهند.

    ```bash
    openclaw agent --model ollama/gemma4 --thinking off
    openclaw agent --model ollama/gemma4 --thinking low
    ```

    یا یک پیش‌فرض برای مدل تنظیم کنید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "ollama/gemma4": {
              thinking: "low",
            },
          },
        },
      },
    }
    ```

    تنظیم به‌ازای هر مدل `params.think`/`params.thinking` می‌تواند تفکر API را
    برای مدلی مشخص غیرفعال یا اجباری کند. OpenClaw این پیکربندی صریح را
    زمانی حفظ می‌کند که اجرای فعال فقط پیش‌فرض ضمنی `off` را داشته باشد؛ یک فرمان زمان اجرا که خاموش نیست،
    مانند `/think medium`، همچنان آن را بازنویسی می‌کند. درخواست تفکر با مقدار درست
    هرگز به مدلی که صراحتاً با
    `reasoning: false` علامت‌گذاری شده است ارسال نمی‌شود؛ درخواست `think: false` همیشه، بدون توجه به این مورد، ارسال می‌شود.

  </Accordion>

  <Accordion title="مدل‌های استدلالی">
    مدل‌هایی با نام `deepseek-r1`، `reasoning`، `reason` یا `think`
    به‌طور پیش‌فرض دارای قابلیت استدلال در نظر گرفته می‌شوند — نیازی به پیکربندی اضافی نیست:

    ```bash
    ollama pull deepseek-r1:32b
    ```

  </Accordion>

  <Accordion title="هزینه مدل‌ها">
    Ollama به‌صورت محلی اجرا می‌شود و رایگان است؛ بنابراین تمام هزینه‌های مدل برای مدل‌های
    شناسایی‌شده خودکار و تعریف‌شده دستی، `0` هستند.
  </Accordion>

  <Accordion title="تعبیه‌های حافظه">
    Plugin همراه Ollama یک ارائه‌دهنده تعبیه حافظه برای
    [جست‌وجوی حافظه](/fa/concepts/memory) ثبت می‌کند. این ارائه‌دهنده از URL پایه و کلید API پیکربندی‌شده Ollama
    استفاده می‌کند، `/api/embed` را فراخوانی می‌کند و در صورت امکان، چند قطعه حافظه را در
    یک درخواست `input` به‌صورت دسته‌ای ارسال می‌کند.

    وقتی `proxy.enabled=true` باشد، درخواست‌های تعبیه به مبدأ حلقه‌بازگشت دقیقاً محلیِ میزبان
    که از `baseUrl` پیکربندی‌شده مشتق شده است، به‌جای پروکسی هدایت مدیریت‌شده از مسیر مستقیم
    محافظت‌شده OpenClaw استفاده می‌کنند. نام میزبان پیکربندی‌شده باید خودِ `localhost`
    یا یک نشانی IP صریح حلقه‌بازگشت باشد — نام‌های DNS که صرفاً به حلقه‌بازگشت تفکیک می‌شوند،
    همچنان از مسیر پروکسی مدیریت‌شده استفاده می‌کنند. میزبان‌های Ollama در LAN،
    tailnet، شبکه خصوصی و عمومی همیشه در مسیر پروکسی مدیریت‌شده باقی می‌مانند
    و تغییر مسیر به میزبان/درگاه دیگری این اعتماد را به ارث نمی‌برد.
    `proxy.loopbackMode: "proxy"` ترافیک حلقه‌بازگشت را بااین‌حال از پروکسی عبور می‌دهد؛
    `proxy.loopbackMode: "block"` پیش از اتصال آن را رد می‌کند —
    [پروکسی مدیریت‌شده](/fa/security/network-proxy#gateway-loopback-mode) را ببینید.

    | ویژگی | مقدار |
    | --- | --- |
    | مدل پیش‌فرض | `nomic-embed-text` |
    | دریافت خودکار | بله، اگر به‌صورت محلی موجود نباشد |
    | هم‌زمانی درون‌خطی پیش‌فرض | 1 (پیش‌فرض سایر ارائه‌دهندگان بیشتر است؛ اگر میزبان توان آن را دارد با `nonBatchConcurrency` افزایش دهید) |

    تعبیه‌های زمان پرس‌وجو برای مدل‌هایی که به پیشوندهای بازیابی نیاز دارند یا
    آن‌ها را توصیه می‌کنند، از این پیشوندها استفاده می‌کنند: `nomic-embed-text`، `qwen3-embedding` و
    `mxbai-embed-large`. دسته‌های سند خام باقی می‌مانند؛ بنابراین نمایه‌های موجود به
    مهاجرت قالب نیاز ندارند.

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          remote: {
            // پیش‌فرض Ollama. اگر بازنمایه‌سازی در میزبان‌های بزرگ‌تر بیش‌ازحد کند است، آن را افزایش دهید.
            nonBatchConcurrency: 1,
          },
        },
      },
    }
    ```

    برای میزبان تعبیه راه‌دور، دامنه احراز هویت را به همان میزبان محدود نگه دارید:

    ```json5
    {
      memory: {
        search: {
          provider: "ollama",
          model: "nomic-embed-text",
          remote: {
            baseUrl: "http://gpu-box.local:11434",
            apiKey: "ollama-local",
            nonBatchConcurrency: 2,
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="پیکربندی جریان‌دهی">
    Ollama به‌طور پیش‌فرض از **API بومی** (`/api/chat`) استفاده می‌کند که از
    جریان‌دهی و فراخوانی ابزار به‌طور هم‌زمان پشتیبانی می‌کند — نیازی به پیکربندی ویژه نیست.

    برای درخواست‌های بومی، کنترل تفکر مستقیماً ارسال می‌شود: `/think off`
    و `openclaw agent --thinking off` مقدار سطح بالای `think: false` را ارسال می‌کنند، مگر اینکه
    `params.think`/`params.thinking` صراحتاً پیکربندی شده باشد؛ `/think
    low|medium|high` رشته تلاش متناظر را ارسال می‌کند؛ `/think max` به
    بالاترین سطح تلاش Ollama، یعنی `think: "high"`، نگاشت می‌شود.

    <Tip>
    برای استفاده از نقطه پایانی سازگار با OpenAI، بخش «حالت قدیمی سازگار با OpenAI» در بالا را ببینید — ممکن است جریان‌دهی و فراخوانی ابزار در آنجا با هم کار نکنند.
    </Tip>

  </Accordion>
</AccordionGroup>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="حلقه خرابی WSL2 (راه‌اندازی مجدد مکرر)">
    در WSL2 همراه با NVIDIA/CUDA، نصب‌کننده رسمی Linux برای Ollama یک
    واحد systemd با نام `ollama.service` و مقدار `Restart=always` ایجاد می‌کند. اگر آن سرویس
    خودکار شروع شود و هنگام راه‌اندازی WSL2 مدلی مبتنی بر GPU را بارگذاری کند، Ollama ممکن است
    هنگام بارگذاری حافظه میزبان را ثابت نگه دارد؛ بازیابی حافظه Hyper-V همیشه نمی‌تواند
    آن صفحه‌ها را پس بگیرد، در نتیجه Windows ممکن است ماشین مجازی WSL2 را خاتمه دهد، systemd
    دوباره Ollama را راه‌اندازی کند و این چرخه تکرار شود.

    شواهد: راه‌اندازی مجدد/خاتمه مکرر WSL2، مصرف بالای CPU در `app.slice` یا
    `ollama.service` بلافاصله پس از شروع WSL2، و دریافت SIGTERM از systemd به‌جای
    پایان‌دهنده OOM در Linux.

    OpenClaw وقتی WSL2، فعال‌بودن `ollama.service`
    همراه با `Restart=always` و نشانگرهای قابل‌مشاهده CUDA را تشخیص دهد، یک هشدار هنگام شروع ثبت می‌کند.

    راهکار کاهش مشکل:

    ```bash
    sudo systemctl disable ollama
    ```

    در سمت Windows، این مورد را به `%USERPROFILE%\.wslconfig` اضافه کنید، سپس
    `wsl --shutdown` را اجرا کنید:

    ```ini
    [experimental]
    autoMemoryReclaim=disabled
    ```

    یا زمان زنده‌ماندن را کوتاه کنید / Ollama را فقط هنگام نیاز به‌صورت دستی شروع کنید:

    ```bash
    export OLLAMA_KEEP_ALIVE=5m
    ollama serve
    ```

    [ollama/ollama#11317](https://github.com/ollama/ollama/issues/11317) را ببینید.

  </Accordion>

  <Accordion title="Ollama شناسایی نمی‌شود">
    تأیید کنید که Ollama در حال اجرا است، `OLLAMA_API_KEY` (یا یک نمایه احراز هویت) تنظیم شده
    و `models.providers.ollama` به‌صورت صریح تعریف **نشده** است:

    ```bash
    ollama serve
    curl http://localhost:11434/api/tags
    ```

  </Accordion>

  <Accordion title="هیچ مدلی در دسترس نیست">
    مدل را به‌صورت محلی دریافت کنید یا آن را صراحتاً در
    `models.providers.ollama` تعریف کنید:

    ```bash
    ollama list  # موارد نصب‌شده را ببینید
    ollama pull gemma4
    ollama pull gpt-oss:20b
    ollama pull llama3.3     # یا مدلی دیگر
    ```

  </Accordion>

  <Accordion title="اتصال رد شد">
    ```bash
    # بررسی کنید Ollama در حال اجرا باشد
    ps aux | grep ollama

    # یا Ollama را دوباره راه‌اندازی کنید
    ollama serve
    ```

  </Accordion>

  <Accordion title="میزبان راه‌دور با curl کار می‌کند اما با OpenClaw نه">
    از همان دستگاه و محیط اجرایی که Gateway را اجرا می‌کند بررسی کنید:

    ```bash
    openclaw gateway status --deep
    curl http://ollama-host:11434/api/tags
    ```

    علت‌های رایج:

    - `baseUrl` به `localhost` اشاره می‌کند، اما Gateway در Docker یا میزبان دیگری اجرا می‌شود.
    - URL از `/v1` استفاده می‌کند و به‌جای رفتار بومی Ollama، رفتار سازگار با OpenAI را انتخاب می‌کند.
    - میزبان راه‌دور به تغییرات دیوار آتش یا اتصال LAN نیاز دارد.
    - مدل در سرویس پس‌زمینه لپ‌تاپ شما وجود دارد، اما در سرویس راه‌دور نیست.

  </Accordion>

  <Accordion title="مدل JSON ابزار را به‌صورت متن خروجی می‌دهد">
    معمولاً ارائه‌دهنده در حالت سازگار با OpenAI است یا مدل نمی‌تواند
    طرح‌واره‌های ابزار را مدیریت کند. حالت بومی را ترجیح دهید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            baseUrl: "http://ollama-host:11434",
            api: "ollama",
          },
        },
      },
    }
    ```

    اگر یک مدل کوچک محلی همچنان در طرح‌واره‌های ابزار شکست می‌خورد،
    `compat.supportsTools: false` را در ورودی همان مدل تنظیم و دوباره آزمایش کنید.

  </Accordion>

  <Accordion title="Kimi یا GLM نمادهای مخدوش برمی‌گرداند">
    پاسخ‌های میزبانی‌شده Kimi/GLM که شامل دنباله‌های طولانی و غیرزبانی از نمادها هستند،
    به‌جای پاسخ موفق، فراخوانی ناموفق ارائه‌دهنده تلقی می‌شوند؛ بنابراین
    مدیریت عادی تلاش مجدد/جایگزینی/خطا وارد عمل می‌شود و متن
    خراب در نشست ماندگار نمی‌شود.

    اگر دوباره رخ داد، نام مدل، فایل نشست فعلی و
    اینکه اجرا از `Cloud + Local` یا `Cloud only` استفاده کرده است را ثبت کنید؛ سپس یک
    نشست تازه و مدل جایگزین را امتحان کنید:

    ```bash
    openclaw infer model run --model ollama/kimi-k2.5:cloud --prompt "Reply with exactly: ok" --json
    openclaw models set ollama/gemma4
    ```

  </Accordion>

  <Accordion title="زمان مدل محلی سرد تمام می‌شود">
    مدل‌های بزرگ محلی ممکن است برای نخستین بارگذاری به زمان زیادی نیاز داشته باشند. مهلت زمانی را به
    ارائه‌دهنده Ollama محدود کنید و در صورت تمایل، مدل را بین نوبت‌ها بارگذاری‌شده نگه دارید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            timeoutSeconds: 300,
            models: [
              {
                id: "gemma4:26b",
                name: "gemma4:26b",
                params: { keep_alive: "15m" },
              },
            ],
          },
        },
      },
    }
    ```

    اگر خود میزبان برای پذیرش اتصال کند است، `timeoutSeconds` نیز
    مهلت زمانی اتصال محافظت‌شده این ارائه‌دهنده را افزایش می‌دهد.

  </Accordion>

  <Accordion title="مدل با زمینه بزرگ بیش‌ازحد کند است یا حافظه‌اش تمام می‌شود">
    بسیاری از مدل‌ها اندازه زمینه‌ای بزرگ‌تر از آنچه سخت‌افزار شما می‌تواند
    به‌راحتی اجرا کند اعلام می‌کنند. Ollama بومی از پیش‌فرض محیط اجرایی خودش استفاده می‌کند، مگر اینکه
    `params.num_ctx` تنظیم شده باشد. برای دستیابی به تأخیر قابل‌پیش‌بینی تا نخستین توکن، هم بودجه OpenClaw و هم
    زمینه درخواست Ollama را محدود کنید:

    ```json5
    {
      models: {
        providers: {
          ollama: {
            contextWindow: 32768,
            maxTokens: 8192,
            models: [
              {
                id: "qwen3.5:9b",
                name: "qwen3.5:9b",
                params: { num_ctx: 32768, thinking: false },
              },
            ],
          },
        },
      },
    }
    ```

    اگر OpenClaw اعلان بیش‌ازحدی ارسال می‌کند، `contextWindow` را کاهش دهید.
    اگر زمینه محیط اجرایی Ollama برای دستگاه بیش‌ازحد بزرگ است، `params.num_ctx` را کاهش دهید.
    اگر تولید بیش‌ازحد طول می‌کشد، `maxTokens` را کاهش دهید.

  </Accordion>
</AccordionGroup>

<Note>
راهنمای بیشتر: [عیب‌یابی](/fa/help/troubleshooting) و [پرسش‌های متداول](/fa/help/faq).
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="Ollama Cloud" href="/fa/providers/ollama-cloud" icon="cloud">
    راه‌اندازی مختص فضای ابری با ارائه‌دهنده اختصاصی `ollama-cloud`.
  </Card>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    نمای کلی همه ارائه‌دهندگان، ارجاعات مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/models" icon="brain">
    چگونگی انتخاب و پیکربندی مدل‌ها.
  </Card>
  <Card title="جست‌وجوی وب Ollama" href="/fa/tools/ollama-search" icon="magnifying-glass">
    جزئیات کامل راه‌اندازی و رفتار جست‌وجوی وب مبتنی بر Ollama.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی.
  </Card>
</CardGroup>
