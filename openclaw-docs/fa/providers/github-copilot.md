---
read_when:
    - می‌خواهید از GitHub Copilot به‌عنوان ارائه‌دهنده مدل استفاده کنید
    - به جریان `openclaw models auth login-github-copilot` نیاز دارید
    - شما در حال انتخاب میان ارائه‌دهندهٔ داخلی Copilot، چارچوب Copilot SDK و Copilot Proxy هستید
summary: با استفاده از جریان دستگاه یا وارد کردن توکن به‌صورت غیرتعاملی، از OpenClaw وارد GitHub Copilot شوید
title: GitHub Copilot
x-i18n:
    generated_at: "2026-07-27T15:40:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e839e6c72e7e7cb106a2f98c62c4994b4f3d6f34a2e76b549f2f6ccfdac91fe6
    source_path: providers/github-copilot.md
    workflow: 16
---

GitHub Copilot دستیار کدنویسی هوش مصنوعی GitHub است. این ابزار دسترسی به مدل‌های
Copilot را برای حساب و طرح GitHub شما فراهم می‌کند. OpenClaw می‌تواند به سه روش
مختلف از Copilot به‌عنوان ارائه‌دهنده مدل یا محیط اجرای عامل استفاده کند.

## سه روش استفاده از Copilot در OpenClaw

<Tabs>
  <Tab title="ارائه‌دهنده داخلی (github-copilot)">
    از جریان ورود بومی دستگاه برای دریافت توکن GitHub استفاده کنید و سپس هنگام اجرای
    OpenClaw، آن را با توکن‌های API مربوط به Copilot مبادله کنید. این مسیر **پیش‌فرض**
    و ساده‌ترین مسیر است، زیرا به VS Code نیاز ندارد.

    <Steps>
      <Step title="اجرای فرمان ورود">
        ```bash
        openclaw models auth login-github-copilot
        ```

        از شما خواسته می‌شود به یک URL مراجعه کنید و کدی یک‌بارمصرف وارد کنید. تا
        زمان تکمیل فرایند، ترمینال را باز نگه دارید.
      </Step>
      <Step title="تنظیم یک مدل پیش‌فرض">
        ```bash
        openclaw models set github-copilot/claude-opus-4.7
        ```

        یا در پیکربندی:

        ```json5
        {
          agents: {
            defaults: { model: { primary: "github-copilot/claude-opus-4.7" } },
          },
        }
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Plugin مهار SDK مربوط به Copilot (copilot)">
    هنگامی که می‌خواهید CLI و SDK مربوط به Copilot در GitHub حلقه سطح پایین عامل
    را برای مدل‌های انتخاب‌شده `github-copilot/*` مدیریت کنند، Plugin خارجی
    `@openclaw/copilot` را نصب کنید.

    ```bash
    openclaw plugins install @openclaw/copilot
    ```

    سپس یک مدل یا ارائه‌دهنده را برای استفاده از محیط اجرا فعال کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: "github-copilot/gpt-5.5",
          models: {
            "github-copilot/gpt-5.5": {
              agentRuntime: { id: "copilot" },
            },
          },
        },
      },
    }
    ```

    زمانی این گزینه را انتخاب کنید که نشست‌های بومی CLI مربوط به Copilot، وضعیت
    رشته مدیریت‌شده با SDK و Compaction تحت مدیریت Copilot را برای آن نوبت‌های عامل
    می‌خواهید. بدون فعال‌سازی صریح `agentRuntime`، مدل‌های `github-copilot/*`
    همچنان از ارائه‌دهنده داخلی استفاده می‌کنند. برای قرارداد کامل محیط اجرا،
    [مهار SDK مربوط به Copilot](/fa/plugins/copilot) را ببینید.

  </Tab>

  <Tab title="Plugin پراکسی Copilot (copilot-proxy)">
    از افزونه VS Code با نام **Copilot Proxy** به‌عنوان پل محلی استفاده کنید. OpenClaw
    با نقطه پایانی `/v1` پراکسی (پیش‌فرض `http://localhost:3000/v1`) ارتباط
    برقرار می‌کند و از فهرست مدل‌هایی که پیکربندی می‌کنید استفاده می‌کند.

    Plugin مربوط به `copilot-proxy` همراه OpenClaw ارائه می‌شود و به‌طور پیش‌فرض
    فعال است. URL پایه و شناسه‌های مدل را با فرمان زیر پیکربندی کنید:

    ```bash
    openclaw models auth login --provider copilot-proxy --set-default
    ```

    <Note>
    زمانی این گزینه را انتخاب کنید که Copilot Proxy را از قبل در VS Code اجرا
    می‌کنید یا باید مسیریابی از طریق آن انجام شود. افزونه VS Code باید در حال اجرا بماند.
    </Note>

  </Tab>
</Tabs>

## GitHub Enterprise (محل اقامت داده‌ها)

اگر سازمان شما از مستأجر GitHub Enterprise با قابلیت اقامت داده‌ها استفاده می‌کند
(میزبان `*.ghe.com` مانند `your-org.ghe.com`)، Copilot به‌جای
`github.com` عمومی روی نقاط پایانی محلی مستأجر قرار دارد. OpenClaw این
امکان را به‌عنوان گزینه احراز هویت درجه‌یک ارائه می‌دهد تا نیازی به ویرایش دستی
URLها نداشته باشید.

<Steps>
  <Step title="انتخاب گزینه احراز هویت Enterprise">
    هنگام راه‌اندازی اولیه یا `openclaw models auth`، گزینه
    **GitHub Copilot (Enterprise / data residency)** را انتخاب کنید. از شما دامنه
    Enterprise خواسته می‌شود (برای مثال `your-org.ghe.com`) و سپس ورود دستگاه
    در همان مستأجر انجام می‌شود.

    فقط ریشه مستأجر (`your-org.ghe.com`) را وارد کنید. میزبان‌های سرویس مشتق‌شده
    مانند `api.your-org.ghe.com` یا `copilot-api.your-org.ghe.com` پذیرفته نمی‌شوند؛
    OpenClaw این نقاط پایانی را به‌طور خودکار از ریشه مستأجر استخراج می‌کند.

    ```bash
    openclaw models auth login --provider github-copilot --method device-enterprise
    ```

  </Step>
  <Step title="ذخیره دامنه در پیکربندی">
    میزبان انتخاب‌شده در پارامترهای ارائه‌دهنده ذخیره می‌شود تا نوسازی‌های بعدی
    توکن و تکمیل‌ها به‌طور خودکار مستأجر را هدف بگیرند:

    ```json5
    {
      models: {
        providers: {
          "github-copilot": { params: { githubDomain: "your-org.ghe.com" } },
        },
      },
    }
    ```

  </Step>
</Steps>

جریان دستگاه، مبادله توکن و تکمیل‌ها به‌ترتیب به
`https://your-org.ghe.com/login/device/code`،
`https://api.your-org.ghe.com/copilot_internal/v2/token` و
`https://copilot-api.your-org.ghe.com` هدایت می‌شوند. توکن‌های اقامت داده دارای
مهر مستأجر هستند و اشاره‌ای به پراکسی ندارند؛ بنابراین URL پایه تکمیل‌ها به‌جای
نقطه پایانی عمومی، از میزبان Copilot مستأجر استفاده می‌کند.

<Note>
تغییر دامنه همیشه باعث اجرای دوباره ورود دستگاه می‌شود. اگر توکن Copilot
ذخیره‌شده‌ای دارید و دامنه دیگری را انتخاب کنید (`github.com` عمومی ↔ یک
مستأجر `*.ghe.com` یا تغییر از یک مستأجر به مستأجر دیگر)، OpenClaw از
توکن موجود دوباره استفاده نمی‌کند؛ ورود تازه‌ای را اجباری می‌کند تا دامنه توکن با
دامنه‌ای که در پیکربندی نوشته می‌شود مطابقت داشته باشد. اجرای دوباره ورود برای
*همان* دامنه همچنان امکان استفاده مجدد از توکن فعلی را پیشنهاد می‌دهد. بازگشت به
`github.com` عمومی، مقدار ذخیره‌شده `githubDomain` را پاک می‌کند تا
پیکربندی به حالت پیش‌فرض بازگردد.
</Note>

<Note>
متغیر محیطی `COPILOT_GITHUB_DOMAIN` دامنه حل‌شده را برای تمام مسیرهای Copilot که آن
را حل می‌کنند بازنویسی می‌کند: ورود دستگاه Enterprise
(`--method device-enterprise`)، میان‌بر مستقل
`openclaw models auth login-github-copilot`، نوسازی توکن، تعبیه‌ها
و تکمیل‌ها. برای راه‌اندازی‌های کاملاً بدون رابط یا پایپ‌لاین CI، آن را روی میزبان
`*.ghe.com` خود تنظیم کنید. برای استفاده از `github.com` عمومی،
آن را تنظیم‌نشده باقی بگذارید (و پارامتر پیکربندی نیز وجود نداشته باشد).
ورودها دامنه‌ای را که توکن برای آن صادر شده است ذخیره می‌کنند (و هنگام ورود
در برابر `github.com` عمومی آن را پاک می‌کنند)، بنابراین حتی پس از حذف
تنظیم متغیر محیطی نیز مسیریابی درست باقی می‌ماند.
</Note>

## پرچم‌های اختیاری

| فرمان                                                                | پرچم            | توضیحات                                          |
| ---------------------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| `openclaw models auth login-github-copilot`                            | `--yes`         | بازنویسی نمایه احراز هویت موجود بدون درخواست تأیید |
| `openclaw models auth login --provider github-copilot --method device` | `--set-default` | اعمال مدل پیش‌فرض پیشنهادی ارائه‌دهنده نیز  |

```bash
# رد کردن تأیید ورود دوباره
openclaw models auth login-github-copilot --yes

# ورود و تنظیم مدل پیش‌فرض در یک مرحله
openclaw models auth login --provider github-copilot --method device --set-default
```

## راه‌اندازی اولیه غیرتعاملی

جریان ورود دستگاه به TTY تعاملی نیاز دارد. برای راه‌اندازی بدون رابط، یک توکن
دسترسی OAuth موجود GitHub را با `openclaw onboard --non-interactive` وارد کنید:

```bash
openclaw onboard --non-interactive --accept-risk \
  --auth-choice github-copilot \
  --github-copilot-token "$COPILOT_GITHUB_TOKEN" \
  --skip-channels --skip-health
```

همچنین می‌توانید `--auth-choice` را حذف کنید؛ ارائه
`--github-copilot-token` گزینه احراز هویت ارائه‌دهنده GitHub Copilot را استنباط می‌کند.
اگر پرچم حذف شود، راه‌اندازی اولیه به‌ترتیب به `COPILOT_GITHUB_TOKEN`،
`GH_TOKEN` و سپس `GITHUB_TOKEN` بازمی‌گردد. برای ذخیره
`tokenRef` مبتنی بر محیط به‌جای متن ساده در `auth-profiles.json`،
از `--secret-input-mode ref` همراه با تنظیم `COPILOT_GITHUB_TOKEN` استفاده کنید.

<AccordionGroup>
  <Accordion title="TTY تعاملی الزامی است">
    جریان ورود دستگاه به TTY تعاملی نیاز دارد. آن را مستقیماً در یک ترمینال اجرا
    کنید، نه در اسکریپت غیرتعاملی یا پایپ‌لاین CI.
  </Accordion>

  <Accordion title="دسترس‌پذیری مدل به طرح شما بستگی دارد">
    دسترس‌پذیری مدل‌های Copilot به طرح GitHub شما بستگی دارد. اگر مدلی رد شد،
    شناسه دیگری را امتحان کنید (برای مثال `github-copilot/gpt-5.5`). برای مشاهده
    فهرست فعلی مدل‌ها، [مدل‌های پشتیبانی‌شده برای هر طرح Copilot](https://docs.github.com/en/copilot/reference/ai-models/supported-models#supported-ai-models-per-copilot-plan)
    در GitHub را ببینید.
  </Accordion>

  <Accordion title="نوسازی زنده کاتالوگ از API مربوط به Copilot">
    پس از اینکه مسیر احراز هویت ورود دستگاه (یا متغیر محیطی) یک توکن GitHub را
    تعیین کرد، OpenClaw کاتالوگ مدل را هنگام نیاز از `${baseUrl}/models`
    (همان نقطه پایانی که Copilot در VS Code استفاده می‌کند) نوسازی می‌کند تا محیط
    اجرا مجوزهای هر حساب و پنجره‌های زمینه دقیق را بدون تغییر مداوم مانیفست دنبال
    کند. مدل‌های تازه منتشرشده Copilot بدون ارتقای OpenClaw قابل مشاهده می‌شوند
    و پنجره‌های زمینه محدودیت‌های واقعی هر مدل را بازتاب می‌دهند
    (برای مثال 400k برای سری gpt-5.x و 1M برای گونه‌های داخلی
    `claude-opus-*-1m`).

    هنگامی که کشف غیرفعال باشد، کاربر نمایه احراز هویت GitHub نداشته باشد، مبادله
    توکن ناموفق شود یا فراخوانی HTTPS مربوط به `/models` خطا دهد،
    کاتالوگ ایستای همراه محصول به‌عنوان جایگزین قابل مشاهده باقی می‌ماند. برای
    انصراف و اتکای کامل به کاتالوگ ایستای مانیفست (سناریوهای آفلاین / جدا از شبکه):

    ```json5
    {
      plugins: {
        entries: {
          "github-copilot": {
            config: { discovery: { enabled: false } },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="انتخاب انتقال">
    شناسه مدل‌های Claude به‌طور خودکار از انتقال Anthropic Messages استفاده
    می‌کنند. مدل‌های Gemini از انتقال OpenAI Chat Completions استفاده می‌کنند؛
    مدل‌های GPT و سری o همچنان از انتقال OpenAI Responses استفاده می‌کنند.
    OpenClaw انتقال صحیح را بر اساس مرجع مدل انتخاب می‌کند.
  </Accordion>

  <Accordion title="سازگاری درخواست">
    OpenClaw سرآیندهای درخواست به سبک Copilot IDE را در انتقال‌های Copilot ارسال
    می‌کند (نسخه‌های ویرایشگر/Plugin در VS Code و شناسه یکپارچه‌سازی
    `vscode-chat`)، نوبت‌های پیگیری نتیجه ابزار را به‌عنوان آغازشده توسط
    عامل علامت‌گذاری می‌کند و هنگامی که نوبتی ورودی تصویر دارد، سرآیند بینایی
    Copilot را تنظیم می‌کند.
  </Accordion>

  <Accordion title="ترتیب حل متغیرهای محیطی">
    OpenClaw احراز هویت Copilot را از متغیرهای محیطی با ترتیب اولویت زیر حل
    می‌کند:

    | اولویت | متغیر              | یادداشت‌ها                            |
    | -------- | --------------------- | -------------------------------- |
    | 1        | `COPILOT_GITHUB_TOKEN` | بالاترین اولویت، مختص Copilot |
    | 2        | `GH_TOKEN`            | توکن CLI مربوط به GitHub (جایگزین)      |
    | 3        | `GITHUB_TOKEN`        | توکن استاندارد GitHub (پایین‌ترین)   |

    هنگامی که چند متغیر تنظیم شده باشند، OpenClaw از متغیری با بالاترین اولویت
    استفاده می‌کند. جریان ورود دستگاه (`openclaw models auth login-github-copilot`) توکن خود را در
    مخزن نمایه احراز هویت ذخیره می‌کند و بر همه متغیرهای محیطی اولویت دارد.

  </Accordion>

  <Accordion title="ذخیره‌سازی توکن">
    ورود، یک توکن GitHub را در مخزن نمایه احراز هویت (شناسه نمایه
    `github-copilot:github`) ذخیره می‌کند و هنگام اجرای OpenClaw آن را با یک توکن
    کوتاه‌عمر API مربوط به Copilot مبادله می‌کند. نیازی نیست توکن را دستی مدیریت کنید.
  </Accordion>
</AccordionGroup>

## تعبیه‌های جست‌وجوی حافظه

GitHub Copilot همچنین می‌تواند به‌عنوان ارائه‌دهنده تعبیه برای
[جست‌وجوی حافظه](/fa/concepts/memory-search) عمل کند. اگر اشتراک Copilot دارید و
وارد شده‌اید، OpenClaw می‌تواند بدون کلید API جداگانه از آن برای تعبیه‌ها استفاده کند.

### پیکربندی

برای استفاده از تعبیه‌های GitHub Copilot، مقدار `memory.search.provider` را صریحاً
تنظیم کنید. اگر توکن GitHub در دسترس باشد، OpenClaw مدل‌های تعبیه موجود را از
API مربوط به Copilot کشف می‌کند و به‌طور خودکار بهترین مدل را انتخاب می‌کند.

```json5
{
  memory: {
    search: {
      provider: "github-copilot",
      // اختیاری: بازنویسی مدل کشف‌شده به‌طور خودکار
      model: "text-embedding-3-small",
    },
  },
}
```

### نحوه کار

1. OpenClaw توکن GitHub شما را حل می‌کند (از متغیرهای محیطی یا نمایه احراز هویت).
2. آن را با یک توکن کوتاه‌عمر API مربوط به Copilot مبادله می‌کند.
3. برای کشف مدل‌های تعبیه موجود، نقطه پایانی `/models` مربوط به Copilot را پرس‌وجو می‌کند.
4. بهترین مدل را انتخاب می‌کند (ترتیب ترجیح: `text-embedding-3-small`،
   `text-embedding-3-large`، `text-embedding-ada-002`).
5. درخواست‌های تعبیه را به نقطه پایانی `/embeddings` مربوط به Copilot ارسال می‌کند.

دسترس‌پذیری مدل به طرح GitHub شما بستگی دارد. اگر هیچ مدل تعبیه‌ای در دسترس
نباشد، OpenClaw از Copilot صرف‌نظر می‌کند و ارائه‌دهنده بعدی را امتحان می‌کند.

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="OAuth و احراز هویت" href="/fa/gateway/authentication" icon="key">
    جزئیات احراز هویت و قواعد استفادهٔ مجدد از اطلاعات اعتباری.
  </Card>
</CardGroup>
