---
read_when:
    - تولید موسیقی یا صدا از طریق عامل
    - پیکربندی ارائه‌دهندگان و مدل‌های تولید موسیقی
    - درک پارامترهای ابزار music_generate
sidebarTitle: Music generation
summary: تولید موسیقی با استفاده از music_generate در گردش‌کارهای ComfyUI، fal، Google Lyria، MiniMax و OpenRouter
title: تولید موسیقی
x-i18n:
    generated_at: "2026-07-27T16:03:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f2a8a4a36e47839c7896046a556f7bf84f6c168492e2de46736635fe2a9358e
    source_path: tools/music-generation.md
    workflow: 16
---

ابزار `music_generate` از طریق قابلیت مشترک تولید موسیقی، با پشتیبانی
ComfyUI، fal، Google، MiniMax و OpenRouter، موسیقی یا صوت تولید می‌کند.

<Note>
`music_generate` فقط زمانی ظاهر می‌شود که دست‌کم یک ارائه‌دهندهٔ تولید موسیقی
در دسترس باشد: پیکربندی صریح `agents.defaults.mediaModels.music` یا یک
ارائه‌دهنده با احراز هویت پیکربندی‌شده (برای مثال، یک کلید API تنظیم‌شده).
</Note>

برای اجرای عامل مبتنی بر نشست، `music_generate` به‌صورت یک وظیفهٔ پس‌زمینه آغاز می‌شود،
پیشرفت را در دفتر وظایف پیگیری می‌کند و سپس، وقتی قطعه آماده شد، عامل را بیدار
می‌کند تا بتواند به کاربر اطلاع دهد و فایل صوتی نهایی را پیوست کند. عامل تکمیل
از قرارداد پاسخ قابل‌مشاهدهٔ نشست پیروی می‌کند: پاسخ نهایی خودکار
در صورت پیکربندی، یا `message(action="send")` وقتی نشست به
ابزار پیام نیاز دارد. اگر نشست درخواست‌کننده غیرفعال باشد یا بیدارسازی آن شکست بخورد و
فایل صوتی تولیدشده همچنان در پاسخ موجود نباشد، OpenClaw یک
جایگزین مستقیم و ایدمپوتنت را فقط با فایل صوتی مفقود ارسال می‌کند.

## شروع سریع

<Tabs>
  <Tab title="مبتنی بر ارائه‌دهندهٔ مشترک">
    <Steps>
      <Step title="پیکربندی احراز هویت">
        برای دست‌کم یک ارائه‌دهنده یک کلید API تنظیم کنید — برای مثال
        `GEMINI_API_KEY` یا `MINIMAX_API_KEY`.
      </Step>
      <Step title="انتخاب مدل پیش‌فرض (اختیاری)">
        ```json5
        {
          agents: {
            defaults: {
              musicGenerationModel: {
                primary: "google/lyria-3-clip-preview",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="درخواست از عامل">
        _«یک قطعهٔ سینث‌پاپ پرانرژی دربارهٔ رانندگی شبانه در شهری
        نئونی تولید کن.»_

        عامل به‌طور خودکار `music_generate` را فراخوانی می‌کند. نیازی به
        قرار دادن ابزار در فهرست مجاز نیست.
      </Step>
    </Steps>

    بدون اجرای عامل مبتنی بر نشست (در زمینه‌های مستقیم/محلی)، ابزار
    به‌صورت درون‌خطی اجرا می‌شود و مسیر نهایی رسانه را در همان نتیجهٔ ابزار برمی‌گرداند.

  </Tab>
  <Tab title="گردش‌کار ComfyUI">
    <Steps>
      <Step title="پیکربندی گردش‌کار">
        `plugins.entries.comfy.config.music` را با یک JSON گردش‌کار
        و گره‌های ورودی/خروجی پیکربندی کنید.
      </Step>
      <Step title="احراز هویت ابری (اختیاری)">
        برای Comfy Cloud، `COMFY_API_KEY` یا `COMFY_CLOUD_API_KEY` را تنظیم کنید.
      </Step>
      <Step title="فراخوانی ابزار">
        ```text
        /tool music_generate prompt="حلقهٔ سینث امبینت گرم با بافت نرم نوار"
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

نمونه پرامپت‌ها:

```text
یک قطعهٔ پیانوی سینمایی با سازهای زهی ملایم و بدون آواز تولید کن.
```

```text
یک حلقهٔ چیپ‌تیون پرانرژی دربارهٔ پرتاب موشک هنگام طلوع خورشید تولید کن.
```

برای بررسی ارائه‌دهندگان/مدل‌های موجود از `action: "list"` و
برای بررسی وظیفهٔ فعال موسیقی مبتنی بر نشست از `action: "status"` استفاده کنید:

```text
/tool music_generate action=list
/tool music_generate action=status
```

نمونهٔ تولید مستقیم:

```text
/tool music_generate prompt="هیپ‌هاپ لو-فای رؤیایی با بافت صفحهٔ وینیل و باران ملایم" instrumental=true
```

## ارائه‌دهندگان پشتیبانی‌شده

| ارائه‌دهنده | مدل پیش‌فرض                 | ورودی‌های مرجع | کنترل‌های پشتیبانی‌شده                                | احراز هویت                             |
| ---------- | ---------------------------- | ---------------- | ----------------------------------------------------- | -------------------------------------- |
| ComfyUI    | `workflow`                   | حداکثر 1 تصویر    | موسیقی یا صوت تعریف‌شده توسط گردش‌کار                 | `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY` |
| fal        | `fal-ai/minimax-music/v2.6`  | هیچ‌کدام         | `lyrics`, `instrumental`, `durationSeconds`, `format` | `FAL_KEY` یا `FAL_API_KEY`             |
| Google     | `lyria-3-clip-preview`       | حداکثر 10 تصویر  | `lyrics`, `instrumental`, `format`                    | `GEMINI_API_KEY`, `GOOGLE_API_KEY`     |
| MiniMax    | `music-2.6`                  | هیچ‌کدام         | `lyrics`, `instrumental`, `format` (فقط mp3)         | `MINIMAX_API_KEY` یا OAuth مربوط به MiniMax     |
| OpenRouter | `google/lyria-3-pro-preview` | حداکثر 1 تصویر    | `lyrics`, `instrumental`, `durationSeconds`, `format` | `OPENROUTER_API_KEY`                   |

MiniMax دو شناسهٔ ارائه‌دهنده را با مدل‌های مشترک ثبت می‌کند: `minimax` برای
احراز هویت با کلید API و `minimax-portal` برای OAuth. ارجاع‌های مدل از مسیر احراز هویت
پیروی می‌کنند (`minimax/music-2.6` در برابر `minimax-portal/music-2.6`)؛ به
[MiniMax](/fa/providers/minimax#music-generation) مراجعه کنید.

fal در کنار مدل پیش‌فرض مبتنی بر MiniMax خود، `fal-ai/ace-step/prompt-to-audio` (wav، بدون متن ترانه، بدون
کلید تغییر حالت بی‌کلام) و `fal-ai/stable-audio-25/text-to-audio` (wav،
فقط پرامپت) را نیز ارائه می‌دهد. مدل پیش‌فرض Google یعنی
`lyria-3-clip-preview` فقط mp3 خروجی می‌دهد؛ `lyria-3-pro-preview` از
wav نیز پشتیبانی می‌کند. MiniMax همچنین `music-2.6-free`، `music-cover` و
`music-cover-free` را ارائه می‌دهد. OpenRouter نیز `google/lyria-3-clip-preview` را ارائه می‌دهد.

### ماتریس قابلیت‌ها

قرارداد حالت صریح که `music_generate`، آزمون‌های قرارداد و
پویش زندهٔ مشترک از آن استفاده می‌کنند:

| ارائه‌دهنده | `generate` | `edit` | محدودیت ویرایش | مسیرهای زندهٔ مشترک                                                     |
| ---------- | :--------: | :----: | ---------- | ------------------------------------------------------------------------- |
| ComfyUI    |     ✓      |   ✓    | 1 تصویر    | در پویش مشترک نیست؛ تحت پوشش `extensions/comfy/comfy.live.test.ts` است |
| fal        |     ✓      |   —    | هیچ‌کدام   | `generate`                                                                |
| Google     |     ✓      |   ✓    | 10 تصویر   | `generate`, `edit`                                                        |
| MiniMax    |     ✓      |   —    | هیچ‌کدام   | `generate`                                                                |
| OpenRouter |     ✓      |   ✓    | 1 تصویر    | `generate`, `edit`                                                        |

## پارامترهای ابزار

<ParamField path="prompt" type="string" required>
  پرامپت تولید موسیقی. برای `action: "generate"` الزامی است.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  `"status"` وظیفهٔ نشست فعلی را برمی‌گرداند؛ `"list"` ارائه‌دهندگان را بررسی می‌کند.
</ParamField>
<ParamField path="model" type="string">
  بازنویسی ارائه‌دهنده/مدل (برای مثال `google/lyria-3-pro-preview`،
  `comfy/workflow`).
</ParamField>
<ParamField path="lyrics" type="string">
  متن ترانهٔ اختیاری، وقتی ارائه‌دهنده از ورودی صریح متن ترانه پشتیبانی می‌کند.
</ParamField>
<ParamField path="instrumental" type="boolean">
  درخواست خروجی فقط بی‌کلام، وقتی ارائه‌دهنده از آن پشتیبانی می‌کند.
</ParamField>
<ParamField path="image" type="string">
  مسیر یا URL یک تصویر مرجع.
</ParamField>
<ParamField path="images" type="string[]">
  چند تصویر مرجع (حداکثر 10 تصویر در ارائه‌دهندگان پشتیبان).
</ParamField>
<ParamField path="durationSeconds" type="number">
  مدت‌زمان هدف برحسب ثانیه، وقتی ارائه‌دهنده از راهنمای مدت‌زمان پشتیبانی می‌کند.
</ParamField>
<ParamField path="format" type='"mp3" | "wav"'>
  راهنمای قالب خروجی، وقتی ارائه‌دهنده از آن پشتیبانی می‌کند.
</ParamField>
<ParamField path="filename" type="string">راهنمای نام فایل خروجی.</ParamField>

<Note>
همهٔ ارائه‌دهندگان از همهٔ پارامترها پشتیبانی نمی‌کنند. OpenClaw همچنان محدودیت‌های سخت،
مانند تعداد ورودی‌ها، را پیش از ارسال اعتبارسنجی می‌کند. وقتی ارائه‌دهنده از
مدت‌زمان پشتیبانی می‌کند اما حداکثر آن کمتر از مقدار درخواستی است، OpenClaw
مقدار را به نزدیک‌ترین مدت‌زمان پشتیبانی‌شده محدود می‌کند. راهنماهای اختیاری واقعاً پشتیبانی‌نشده،
وقتی ارائه‌دهنده یا مدل انتخاب‌شده نتواند آن‌ها را رعایت کند، با یک هشدار
نادیده گرفته می‌شوند. نتایج ابزار تنظیمات اعمال‌شده را گزارش می‌کنند؛ `details.normalization`
هر نگاشت از مقدار درخواستی به مقدار اعمال‌شده را ثبت می‌کند.
</Note>

مهلت زمانی درخواست ارائه‌دهنده فقط در پیکربندی اپراتور تعریف می‌شود. OpenClaw در صورت پیکربندی از
`agents.defaults.mediaModels.music.timeoutMs` استفاده می‌کند، مقادیر کمتر از 120000ms را به 120000ms
افزایش می‌دهد و در غیر این صورت مهلت پیش‌فرض درخواست ارائه‌دهنده را
300000ms در نظر می‌گیرد.

## رفتار ناهمگام

تولید موسیقی مبتنی بر نشست به‌صورت یک وظیفهٔ پس‌زمینه اجرا می‌شود:

- **وظیفهٔ پس‌زمینه:** `music_generate` یک وظیفهٔ پس‌زمینه ایجاد می‌کند، پاسخ
  آغازشده/وظیفه را فوراً برمی‌گرداند و قطعهٔ نهایی را بعداً در
  یک پیام پیگیری عامل ارسال می‌کند.
- **جلوگیری از تکرار:** تا وقتی یک وظیفه در حالت `queued` یا `running` باشد، فراخوانی‌های بعدی
  `music_generate` در همان نشست به‌جای آغاز تولیدی دیگر،
  وضعیت وظیفه را برمی‌گردانند. برای بررسی صریح از `action: "status"` استفاده کنید.
  یک درخواست منطبق که به‌تازگی تکمیل شده باشد نیز برای 2 دقیقه تکرارزدایی می‌شود.
- **بررسی وضعیت:** `openclaw tasks list` یا `openclaw tasks show <taskId>`
  وضعیت در صف، در حال اجرا و نهایی را بررسی می‌کند.
- **بیدارسازی تکمیل:** OpenClaw یک رویداد تکمیل داخلی را دوباره
  به همان نشست تزریق می‌کند تا مدل بتواند پیام پیگیری قابل‌مشاهده برای کاربر را
  خودش بنویسد.
- **راهنمای پرامپت:** گردش‌های بعدی کاربر/دستی در همان نشست، وقتی وظیفهٔ موسیقی
  از قبل در حال اجرا است، یک راهنمای کوچک زمان اجرا دریافت می‌کنند تا مدل
  کورکورانه دوباره `music_generate` را فراخوانی نکند.
- **جایگزین بدون نشست:** زمینه‌های مستقیم/محلی بدون نشست واقعی عامل،
  به‌صورت درون‌خطی اجرا می‌شوند و نتیجهٔ نهایی صوتی را در همان گردش برمی‌گردانند.

### چرخهٔ عمر وظیفه

وظیفهٔ موسیقی همان وضعیت‌های رجیستری عمومی وظایف را ارائه می‌کند (برای ماشین حالت
کامل، شامل `timed_out`، `cancelled` و `lost`، به
[وظایف پس‌زمینه](/fa/automation/tasks#task-lifecycle) مراجعه کنید). بیشتر اجراهای موسیقی
از این وضعیت‌ها عبور می‌کنند:

| وضعیت       | معنا                                                                                           |
| ----------- | ---------------------------------------------------------------------------------------------- |
| `queued`    | وظیفه ایجاد شده و منتظر پذیرش از سوی ارائه‌دهنده است.                                         |
| `running`   | ارائه‌دهنده در حال پردازش است (معمولاً 30 ثانیه تا 3 دقیقه، بسته به ارائه‌دهنده و مدت‌زمان). |
| `succeeded` | قطعه آماده است؛ عامل بیدار می‌شود و آن را در مکالمه ارسال می‌کند.                              |
| `failed`    | خطا یا پایان مهلت ارائه‌دهنده؛ عامل با جزئیات خطا بیدار می‌شود.                               |

وضعیت را از CLI بررسی کنید:

```bash
openclaw tasks list
openclaw tasks show <taskId>
openclaw tasks cancel <taskId>
```

## پیکربندی

### انتخاب مدل

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
        fallbacks: ["fal/fal-ai/minimax-music/v2.6", "minimax/music-2.6"],
      },
    },
  },
}
```

### ترتیب انتخاب ارائه‌دهنده

OpenClaw ارائه‌دهندگان را به این ترتیب امتحان می‌کند:

1. پارامتر `model` از فراخوانی ابزار (اگر عامل یکی را مشخص کند).
2. `musicGenerationModel.primary` از پیکربندی.
3. `musicGenerationModel.fallbacks` به‌ترتیب.
4. تشخیص خودکار فقط با استفاده از پیش‌فرض‌های ارائه‌دهندهٔ مبتنی بر احراز هویت:
   - ابتدا ارائه‌دهندهٔ فعلی مدل متنی پیش‌فرض، اگر تولید موسیقی را نیز
     ارائه دهد؛
   - سایر ارائه‌دهندگان ثبت‌شدهٔ تولید موسیقی، به‌ترتیب الفبایی
     شناسهٔ ارائه‌دهنده.

اگر یک ارائه‌دهنده شکست بخورد، گزینهٔ بعدی به‌طور خودکار امتحان می‌شود. اگر همه
شکست بخورند، خطا شامل جزئیات هر تلاش خواهد بود.

جایگزینی خودکار میان ارائه‌دهندگان احراز هویت‌شده همیشه فعال است. مقدار
`model` در هر فراخوانی همچنان مرجع نهایی است.

## نکات ارائه‌دهندگان

<AccordionGroup>
  <Accordion title="ComfyUI">
    مبتنی بر گردش‌کار است و به گراف پیکربندی‌شده و نگاشت Node
    برای فیلدهای پرامپت/خروجی وابسته است. Plugin همراه `comfy` از طریق
    رجیستری ارائه‌دهنده تولید موسیقی به ابزار مشترک `music_generate`
    متصل می‌شود.
  </Accordion>
  <Accordion title="fal">
    از نقاط پایانی مدل fal از طریق مسیر مشترک احراز هویت ارائه‌دهنده استفاده می‌کند.
    ارائه‌دهنده همراه به‌طور پیش‌فرض از `fal-ai/minimax-music/v2.6` استفاده می‌کند و
    `fal-ai/ace-step/prompt-to-audio` و
    `fal-ai/stable-audio-25/text-to-audio` را نیز برای درخواست‌های تبدیل پرامپت به صدا ارائه می‌دهد.
    حالت اشعار و بی‌کلام فقط مخصوص مدل MiniMax است؛ دو مدل دیگر
    فقط از پرامپت پشتیبانی می‌کنند.
  </Accordion>
  <Accordion title="Google (Lyria 3)">
    از تولید دسته‌ای Lyria 3 استفاده می‌کند. جریان همراه فعلی از
    پرامپت، متن اختیاری اشعار و تصاویر مرجع اختیاری پشتیبانی می‌کند. مدل
    پیش‌فرض `lyria-3-clip-preview` فقط خروجی mp3 تولید می‌کند؛ مدل
    `lyria-3-pro-preview` از wav نیز پشتیبانی می‌کند.
  </Accordion>
  <Accordion title="MiniMax">
    از نقطه پایانی دسته‌ای `music_generation` استفاده می‌کند. از پرامپت، اشعار
    اختیاری، حالت بی‌کلام و خروجی mp3 از طریق احراز هویت با کلید API
    `minimax` یا OAuth ‏`minimax-portal` پشتیبانی می‌کند. همچنین مدل‌های
    `music-2.6-free`، `music-cover` و `music-cover-free` را ارائه می‌دهد.
  </Accordion>
  <Accordion title="OpenRouter">
    از خروجی صوتی تکمیل‌های چت OpenRouter با پخش جریانی فعال استفاده می‌کند.
    ارائه‌دهنده همراه به‌طور پیش‌فرض از `google/lyria-3-pro-preview` استفاده می‌کند و
    `openrouter/google/lyria-3-clip-preview` را نیز ارائه می‌دهد.
  </Accordion>
</AccordionGroup>

## انتخاب مسیر مناسب

- **مبتنی بر ارائه‌دهنده مشترک**، زمانی که انتخاب مدل، جایگزینی
  ارائه‌دهنده در صورت خرابی و جریان داخلی ناهمگام وظیفه/وضعیت را می‌خواهید.
- **مسیر Plugin ‏(ComfyUI)**، زمانی که به گراف گردش‌کار سفارشی یا
  ارائه‌دهنده‌ای نیاز دارید که بخشی از قابلیت همراه و مشترک موسیقی نیست.

اگر رفتار ویژه ComfyUI را اشکال‌زدایی می‌کنید، به
[ComfyUI](/fa/providers/comfy) مراجعه کنید. اگر رفتار ارائه‌دهنده مشترک را
اشکال‌زدایی می‌کنید، از [fal](/fa/providers/fal)، [Google (Gemini)](/fa/providers/google)،
[MiniMax](/fa/providers/minimax) یا [OpenRouter](/fa/providers/openrouter) شروع کنید.

## حالت‌های قابلیت ارائه‌دهنده

قرارداد مشترک تولید موسیقی از اعلان صریح حالت‌ها پشتیبانی می‌کند:

- `generate` برای تولید فقط با پرامپت.
- `edit` هنگامی که درخواست شامل یک یا چند تصویر مرجع است.

پیاده‌سازی‌های جدید ارائه‌دهنده باید بلوک‌های صریح حالت را ترجیح دهند:

```typescript
capabilities: {
  generate: {
    maxTracks: 1,
    supportsLyrics: true,
    supportsFormat: true,
  },
  edit: {
    enabled: true,
    maxTracks: 1,
    maxInputImages: 1,
    supportsFormat: true,
  },
}
```

فیلدهای مسطح قدیمی مانند `maxInputImages`، `supportsLyrics` و
`supportsFormat` برای اعلام پشتیبانی از ویرایش **کافی نیستند**. ارائه‌دهندگان
باید `generate` و `edit` را صراحتاً اعلام کنند تا آزمون‌های زنده،
آزمون‌های قرارداد و ابزار مشترک `music_generate` بتوانند پشتیبانی از حالت را
به‌صورت قطعی اعتبارسنجی کنند.

## آزمون‌های زنده

پوشش زنده اختیاری برای ارائه‌دهندگان همراه و مشترک (fal، Google، MiniMax،
OpenRouter):

```bash
OPENCLAW_LIVE_TEST=1 pnpm test:live -- extensions/music-generation-providers.live.test.ts
```

پوسته معادل مخزن که همان فایل آزمون را اجرا می‌کند:

```bash
pnpm test:live:media:music
```

این فایل زنده به‌طور پیش‌فرض متغیرهای محیطی از پیش صادرشده ارائه‌دهنده را بر
پروفایل‌های احراز هویت ذخیره‌شده مقدم می‌داند و وقتی ارائه‌دهنده حالت ویرایش را
فعال می‌کند، پوشش `generate` و `edit` اعلام‌شده را اجرا می‌کند.
پوشش فعلی:

- `google`: ‏`generate` به‌علاوه `edit`
- `fal`: فقط `generate`
- `minimax`: فقط `generate`
- `openrouter`: ‏`generate` به‌علاوه `edit`
- `comfy`: پوشش زنده جداگانه Comfy، خارج از پیمایش ارائه‌دهندگان مشترک

پوشش زنده اختیاری برای مسیر همراه موسیقی ComfyUI:

```bash
OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
```

فایل زنده Comfy در صورت پیکربندی آن بخش‌ها، گردش‌کارهای تصویر و ویدیوی comfy را
نیز پوشش می‌دهد.

## مرتبط

- [وظایف پس‌زمینه](/fa/automation/tasks) — رهگیری وظیفه برای اجراهای جداشده `music_generate`
- [ComfyUI](/fa/providers/comfy)
- [مرجع پیکربندی](/fa/gateway/config-agents#agent-defaults) — پیکربندی `musicGenerationModel`
- [Google (Gemini)](/fa/providers/google)
- [MiniMax](/fa/providers/minimax)
- [مدل‌ها](/fa/concepts/models) — پیکربندی مدل و جایگزینی در صورت خرابی
- [نمای کلی ابزارها](/fa/tools)
