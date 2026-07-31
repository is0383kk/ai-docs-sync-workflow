---
read_when:
    - افزودن یا تغییر فرمان‌های `openclaw infer`
    - طراحی خودکارسازی پایدار قابلیت‌ها در حالت بدون رابط گرافیکی
summary: CLI مبتنی بر استنتاج اولیه برای گردش‌کارهای مدل، تصویر، صدا، TTS، ویدئو، وب و تعبیه‌سازی با پشتیبانی ارائه‌دهنده
title: CLI استنتاج
x-i18n:
    generated_at: "2026-07-27T13:53:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3147bb516a08e12c4eacd6bd527af62049ecae25b5fde9439da6a4431c147b07
    source_path: cli/infer.md
    workflow: 16
---

`openclaw infer` سطح هدلس متعارف برای استنتاج مبتنی بر ارائه‌دهنده است. این سطح، خانواده‌های قابلیت (`model`، `image`، `audio`، `tts`، `video`، `web`، `embedding`) را ارائه می‌کند، نه نام‌های خام RPC در Gateway یا شناسه‌های ابزار عامل. `openclaw capability ...` نام مستعاری برای همان درخت فرمان است.

دلایل ترجیح آن به یک پوشش‌دهندهٔ یک‌بارهٔ ارائه‌دهنده:

- از ارائه‌دهندگان و مدل‌هایی که از قبل در OpenClaw پیکربندی شده‌اند، دوباره استفاده می‌کند.
- پوشش پایدار `--json` برای اسکریپت‌ها و خودکارسازی عامل‌محور (به [خروجی JSON](#json-output) مراجعه کنید).
- برای بیشتر زیرفرمان‌ها مسیر محلی معمول را بدون Gateway اجرا می‌کند.
- برای بررسی‌های سرتاسری ارائه‌دهنده، پیش از ارسال درخواست به ارائه‌دهنده، CLI عرضه‌شده، بارگذاری پیکربندی، تفکیک عامل پیش‌فرض، فعال‌سازی Plugin همراه و زمان‌اجرای مشترک قابلیت را به کار می‌گیرد.

## تبدیل infer به یک مهارت

این متن را در یک عامل کپی و جای‌گذاری کنید:

```text
صفحهٔ https://docs.openclaw.ai/cli/infer را بخوان، سپس مهارتی ایجاد کن که گردش‌کارهای متداول من را به `openclaw infer` هدایت کند.
بر اجرای مدل، تولید تصویر، تولید ویدئو، رونویسی صوت، TTS، جست‌وجوی وب و تعبیه‌ها تمرکز کن.
```

یک مهارت مناسب مبتنی بر infer، مقصودهای متداول کاربر را به زیرفرمان درست نگاشت می‌کند، برای هر گردش‌کار چند نمونهٔ متعارف دارد، `openclaw infer ...` را به گزینه‌های سطح پایین‌تر ترجیح می‌دهد و کل سطح infer را دوباره در بدنهٔ مهارت مستند نمی‌کند.

## درخت فرمان

```text
 openclaw infer
  list
  inspect

  model
    run
    list
    inspect
    providers
    auth login
    auth logout
    auth status

  image
    generate
    edit
    describe
    describe-many
    providers

  audio
    transcribe
    providers

  tts
    convert
    voices
    providers
    personas
    status
    enable
    disable
    set-provider
    set-persona

  video
    generate
    describe
    providers

  web
    search
    fetch
    providers

  embedding
    create
    providers
```

`infer list` / `infer inspect --name <capability>` این درخت را به‌شکل داده نمایش می‌دهند (شناسهٔ قابلیت، انتقال‌ها، توضیحات).

## کارهای متداول

| کار                           | فرمان                                                                                         | یادداشت‌ها                                                    |
| ----------------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| اجرای یک پرامپت متنی/مدل      | `openclaw infer model run --prompt "..." --json`                                                                            | به‌طور پیش‌فرض محلی                                           |
| اجرای پرامپت مدل روی تصاویر   | `openclaw infer model run --prompt "Describe this" --file ./image.png --model provider/model`                                                                            | برای چند تصویر، `--file` را تکرار کنید              |
| تولید تصویر                   | `openclaw infer image generate --prompt "..." --json`                                                                            | هنگام شروع از یک فایل موجود، از `image edit` استفاده کنید |
| توصیف فایل یا URL تصویر       | `openclaw infer image describe --file ./image.png --prompt "..." --json`                                                                            | `--model` باید یک `<provider/model>` دارای قابلیت تصویر باشد |
| رونویسی صوت                   | `openclaw infer audio transcribe --file ./memo.m4a --json`                                                                            | `--model` باید `<provider/model>` باشد               |
| ترکیب گفتار                   | `openclaw infer tts convert --text "..." --output ./speech.mp3 --json`                                                                            | `tts status` فقط از طریق Gateway اجرا می‌شود            |
| تولید ویدئو                   | `openclaw infer video generate --prompt "..." --json`                                                                            | از راهنمایی‌های ارائه‌دهنده مانند `--resolution` پشتیبانی می‌کند |
| توصیف فایل ویدئو              | `openclaw infer video describe --file ./clip.mp4 --json`                                                                            | `--model` باید `<provider/model>` باشد               |
| جست‌وجوی وب                   | `openclaw infer web search --query "..." --json`                                                                            |                                                               |
| واکشی صفحهٔ وب                | `openclaw infer web fetch --url https://example.com --json`                                                                            |                                                               |
| ایجاد تعبیه‌ها                | `openclaw infer embedding create --text "..." --json`                                                                            |                                                               |

## رفتار

- وقتی خروجی ورودی فرمان یا اسکریپت دیگری است، از `--json` استفاده کنید؛ در غیر این صورت از خروجی متنی استفاده کنید.
- برای تثبیت یک بک‌اند مشخص، از `--provider` یا `--model provider/model` استفاده کنید.
- برای بازنویسی یک‌بارهٔ تفکر/استدلال، از `model run --thinking <level>` استفاده کنید: `off`، `minimal`، `low`، `medium`، `high`، `adaptive`، `xhigh` یا `max`.
- برای `image describe`، `audio transcribe` و `video describe`، `--model` باید از قالب `<provider/model>` استفاده کند.
- برای `image describe`، `--file` مسیرهای محلی و URLهای HTTP(S) را می‌پذیرد؛ URLهای راه‌دور از سیاست معمول SSRF واکشی رسانه عبور می‌کنند.
- فرمان‌های اجرای بدون حالت (`model run`، `image *`، `audio *`، `video *`، `web *`، `embedding *`) به‌طور پیش‌فرض محلی هستند. فرمان‌های حالت مدیریت‌شده توسط Gateway (`tts status`) به‌طور پیش‌فرض از Gateway استفاده می‌کنند.
- مسیر محلی هرگز به در حال اجرا بودن Gateway نیاز ندارد.
- `model run` محلی، تکمیل یک‌باره و سبک ارائه‌دهنده است: مدل و احراز هویت عامل پیکربندی‌شده را تفکیک می‌کند، اما نوبت عامل گفت‌وگو را آغاز نمی‌کند، ابزارها را بارگذاری نمی‌کند و سرورهای MCP همراه را باز نمی‌کند.
- `model run --file` فایل‌های تصویر را با نوع MIME تشخیص‌داده‌شدهٔ خودکار به پرامپت پیوست می‌کند؛ برای چند تصویر، `--file` را تکرار کنید. فایل‌های غیرتصویری رد می‌شوند — در عوض از `infer audio transcribe` یا `infer video describe` استفاده کنید.
- `model run --gateway` مسیریابی Gateway، احراز هویت ذخیره‌شده، انتخاب ارائه‌دهنده و زمان‌اجرای تعبیه‌شده را به کار می‌گیرد، اما همچنان یک کاوش خام مدل باقی می‌ماند: بدون رونوشت نشست قبلی، زمینهٔ راه‌اندازی/AGENTS، ابزارها یا سرورهای MCP همراه.
- `model run --gateway --model <provider/model>` به اعتبارنامهٔ Gateway متعلق به اپراتور مورداعتماد نیاز دارد، زیرا از Gateway می‌خواهد یک بازنویسی یک‌بارهٔ ارائه‌دهنده/مدل را اجرا کند.

## مدل

استنتاج متن و بازرسی مدل/ارائه‌دهنده.

```bash
openclaw infer model run --prompt "Reply with exactly: smoke-ok" --json
openclaw infer model run --prompt "Summarize this changelog entry" --model openai/gpt-5.4 --json
openclaw infer model run --prompt "Describe this image in one sentence" --file ./photo.jpg --model google/gemini-2.5-flash --json
openclaw infer model run --prompt "Use more reasoning here" --thinking high --json
openclaw infer model providers --json
openclaw infer model inspect --model gpt-5.6-sol --json
```

از ارجاع‌های کامل `<provider/model>` همراه با `--local` استفاده کنید تا یک ارائه‌دهنده را بدون راه‌اندازی Gateway یا بارگذاری سطح ابزار عامل، آزمایش سریع کنید:

```bash
openclaw infer model run --local --model anthropic/claude-sonnet-4-6 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model cerebras/zai-glm-4.7 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model google/gemini-2.5-flash --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model groq/llama-3.1-8b-instant --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model mistral/mistral-medium-3-5 --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model mistral/mistral-small-latest --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model openai/gpt-5.6-luna --prompt "Reply with exactly: pong" --json
openclaw infer model run --local --model ollama/qwen2.5vl:7b --prompt "Describe this image." --file ./photo.jpg --json
```

یادداشت‌ها:

- `model run` محلی، محدودترین آزمایش سریع CLI برای سلامت ارائه‌دهنده/مدل/احراز هویت است: برای ارائه‌دهندگان غیر از ChatGPT-Codex فقط پرامپت ارائه‌شده را ارسال می‌کند.
- `model run --model <provider/model>` محلی می‌تواند ردیف‌های دقیق کاتالوگ ایستای همراه را (همان ردیف‌هایی که `openclaw models list --all` نمایش می‌دهد) پیش از نوشته‌شدن آن ارائه‌دهنده در پیکربندی تفکیک کند. احراز هویت ارائه‌دهنده همچنان الزامی است؛ اعتبارنامه‌های موجودنبودن به‌صورت خطای احراز هویت شکست می‌خورند، نه `Unknown model`.
- برای کاوش‌های استدلال Mistral Medium 3.5، دما را تنظیم‌نشده/پیش‌فرض بگذارید. Mistral مقدار `reasoning_effort="high"` را با `temperature: 0` رد می‌کند؛ از دمای پیش‌فرض یا مقداری غیرصفر مانند `0.7` استفاده کنید.
- کاوش‌های محلی OAuth مربوط به OpenAI ChatGPT/Codex (API ‏`openai-chatgpt-responses`) یک دستور سیستمی حداقلی اضافه می‌کنند تا انتقال بتواند فیلد الزامی `instructions` خود را پر کند — بدون زمینهٔ کامل عامل، ابزارها، حافظه یا رونوشت نشست.
- `model run --file` محتوای تصویر را مستقیماً به پیام واحد کاربر پیوست می‌کند. قالب‌های متداول (PNG، JPEG، WebP) هنگامی کار می‌کنند که نوع MIME به‌شکل `image/*` تشخیص داده شود؛ فایل‌های پشتیبانی‌نشده یا ناشناخته پیش از فراخوانی ارائه‌دهنده شکست می‌خورند. هنگامی که به‌جای کاوش مستقیم مدل چندوجهی، مسیریابی و مسیرهای جایگزین مدل تصویر OpenClaw را می‌خواهید، از `infer image describe` استفاده کنید.
- مدل انتخاب‌شده باید از ورودی تصویر پشتیبانی کند؛ مدل‌های فقط‌متنی ممکن است درخواست را در لایهٔ ارائه‌دهنده رد کنند.
- `model run --prompt` باید دارای متن غیرسفیدفاصله باشد؛ پرامپت‌های خالی پیش از هر فراخوانی ارائه‌دهنده یا Gateway رد می‌شوند.
- `model run` محلی هنگامی که ارائه‌دهنده هیچ خروجی متنی بازنگرداند، با کد غیرصفر خارج می‌شود تا ارائه‌دهندگان دسترس‌ناپذیر و تکمیل‌های خالی شبیه کاوش موفق به نظر نرسند.
- برای آزمایش مسیریابی Gateway یا راه‌اندازی زمان‌اجرای عامل، در حالی که ورودی مدل خام باقی می‌ماند، از `model run --gateway` استفاده کنید. برای زمینهٔ کامل عامل، ابزارها، حافظه و رونوشت نشست از `openclaw agent` یا یک سطح گفت‌وگو استفاده کنید.
- `--thinking adaptive` به `medium` در سطح زمان‌اجرای تکمیل نگاشت می‌شود؛ `--thinking max` برای مدل‌های OpenAI که از حداکثر تلاش بومی پشتیبانی می‌کنند به `max` و در غیر این صورت به `xhigh` نگاشت می‌شود.
- `model auth login`، `model auth logout` و `model auth status` وضعیت احراز هویت ذخیره‌شدهٔ ارائه‌دهنده را مدیریت می‌کنند.

## تصویر

تولید، ویرایش و توصیف.

```bash
openclaw infer image generate --prompt "friendly lobster illustration" --json
openclaw infer image generate --prompt "cinematic product photo of headphones" --json
openclaw infer image generate --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "simple red circle sticker on a transparent background" --json
openclaw infer image generate --model openai/gpt-image-2 --quality low --openai-moderation low --prompt "low-cost draft poster" --json
openclaw infer image generate --prompt "slow image backend" --timeout-ms 180000 --json
openclaw infer image edit --file ./logo.png --model openai/gpt-image-1.5 --output-format png --background transparent --prompt "keep the logo, remove the background" --json
openclaw infer image edit --file ./poster.png --prompt "make this a vertical story ad" --size 2160x3840 --aspect-ratio 9:16 --resolution 4K --json
openclaw infer image describe --file ./photo.jpg --json
openclaw infer image describe --file https://example.com/photo.png --json
openclaw infer image describe --file ./receipt.jpg --prompt "Extract the merchant, date, and total" --json
openclaw infer image describe-many --file ./before.png --file ./after.png --prompt "Compare the screenshots and list visible UI changes" --json
openclaw infer image describe --file ./ui-screenshot.png --model openai/gpt-5.4-mini --json
openclaw infer image describe --file ./photo.jpg --model ollama/qwen2.5vl:7b --prompt "Describe the image in one sentence" --timeout-ms 300000 --json
```

یادداشت‌ها:

- هنگام شروع از فایل‌های ورودی موجود، از `image edit` استفاده کنید؛ `--size`، `--aspect-ratio` یا `--resolution` در ارائه‌دهندگان/مدل‌هایی که از آن‌ها پشتیبانی می‌کنند، راهنمایی‌های هندسی اضافه می‌کنند.
- `--output-format png --background transparent` همراه با `--model openai/gpt-image-1.5` خروجی PNG با پس‌زمینهٔ شفاف از OpenAI ارائه می‌دهد؛ `--openai-background` نام مستعار ویژهٔ OpenAI برای همین راهنمایی است. ارائه‌دهندگانی که پشتیبانی از پس‌زمینه را اعلام نمی‌کنند، آن را به‌عنوان بازنویسی نادیده‌گرفته‌شده گزارش می‌کنند (به `ignoredOverrides` در [پوش JSON](#json-output) مراجعه کنید).
- `--quality low|medium|high|auto` برای ارائه‌دهندگانی که از راهنمایی‌های کیفیت تصویر پشتیبانی می‌کنند، از جمله OpenAI، کار می‌کند. OpenAI همچنین `--openai-moderation low|auto` را می‌پذیرد.
- `image providers --json` فهرست می‌کند که کدام ارائه‌دهندگان تصویر همراه، قابل کشف، پیکربندی‌شده و انتخاب‌شده هستند و هرکدام چه قابلیت‌های تولید/ویرایشی ارائه می‌دهند.
- `image generate --model <provider/model> --json` محدودترین آزمون دود زنده برای تغییرات تولید تصویر است:

  ```bash
  openclaw infer image providers --json
  openclaw infer image generate \
    --model google/gemini-3.1-flash-image \
    --prompt "تصویر آزمایشی تخت و مینیمال: یک مربع آبی روی پس‌زمینهٔ سفید، بدون متن." \
    --output ./openclaw-infer-image-smoke.png \
    --json
  ```

  پاسخ، `ok`، `provider`، `model`، `attempts` و مسیرهای خروجی نوشته‌شده را گزارش می‌کند. وقتی `--output` تنظیم شده باشد، پسوند نهایی ممکن است از نوع MIME بازگردانده‌شده توسط ارائه‌دهنده پیروی کند.

- برای `image describe` و `image describe-many`، از `--prompt` برای دستورالعملی ویژهٔ وظیفه (OCR، مقایسه، بررسی رابط کاربری، شرح مختصر) استفاده کنید.
- برای مدل‌های بینایی محلی کُند یا راه‌اندازی سرد Ollama، از `--timeout-ms` استفاده کنید.
- برای `image describe`، ابتدا یک `--model` صریح (که باید یک `<provider/model>` دارای قابلیت تصویر باشد) اجرا می‌شود؛ سپس اگر آن فراخوانی شکست بخورد، `agents.defaults.imageModel.fallbacks` پیکربندی‌شده امتحان می‌شود. خطاهای آماده‌سازی ورودی (فایل مفقود، URL پشتیبانی‌نشده) پیش از هر تلاش بازگشتی باعث شکست می‌شوند و مدل باید در کاتالوگ مدل یا پیکربندی ارائه‌دهنده دارای قابلیت تصویر باشد.
- برای مدل‌های بینایی محلی Ollama، ابتدا مدل را دریافت کنید و `OLLAMA_API_KEY` را روی هر مقدار جای‌نگهدار، برای مثال `ollama-local`، تنظیم کنید. به [Ollama](/fa/providers/ollama#vision-and-image-description) مراجعه کنید.

## صوت

رونویسی فایل (نه مدیریت نشست بلادرنگ).

```bash
openclaw infer audio transcribe --file ./memo.m4a --json
openclaw infer audio transcribe --file ./team-sync.m4a --language en --prompt "روی نام‌ها و موارد اقدام تمرکز کن" --json
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

`--model` باید `<provider/model>` باشد.

## TTS

ترکیب گفتار و وضعیت ارائه‌دهنده/شخصیت TTS.

```bash
openclaw infer tts convert --text "سلام از openclaw" --output ./hello.mp3 --json
openclaw infer tts convert --text "ساخت شما کامل شد" --output ./build-complete.mp3 --json
openclaw infer tts providers --json
openclaw infer tts personas --json
openclaw infer tts status --json
```

نکته‌ها:

- `tts status` فقط از `--gateway` پشتیبانی می‌کند (وضعیت TTS مدیریت‌شده توسط Gateway را بازتاب می‌دهد).
- برای بررسی و پیکربندی رفتار TTS، از `tts providers`، `tts voices`، `tts personas`، `tts set-provider` و `tts set-persona` استفاده کنید.

## ویدئو

تولید و توصیف.

```bash
openclaw infer video generate --prompt "غروب سینمایی بر فراز اقیانوس" --json
openclaw infer video generate --prompt "نمای آهستهٔ پهپاد بر فراز دریاچه‌ای جنگلی" --resolution 768P --duration 6 --json
openclaw infer video describe --file ./clip.mp4 --json
openclaw infer video describe --file ./clip.mp4 --model openai/gpt-5.4-mini --json
```

نکته‌ها:

- `video generate` مقادیر `--size`، `--aspect-ratio`، `--resolution`، `--duration`، `--audio`، `--watermark` و `--timeout-ms` را می‌پذیرد که به زمان‌اجرای تولید ویدئو ارسال می‌شوند.
- `--model` برای `video describe` باید `<provider/model>` باشد.

## وب

جست‌وجو و واکشی.

```bash
openclaw infer web search --query "مستندات OpenClaw" --json
openclaw infer web search --query "ارائه‌دهندگان وب infer در OpenClaw" --json
openclaw infer web fetch --url https://docs.openclaw.ai/cli/infer --json
openclaw infer web providers --json
```

`web providers` ارائه‌دهندگان موجود، پیکربندی‌شده و انتخاب‌شده برای جست‌وجو و واکشی را فهرست می‌کند.

## تعبیه‌سازی

ساخت بردار و بررسی ارائه‌دهندهٔ تعبیه‌سازی.

```bash
openclaw infer embedding create --text "خرچنگ مهربان" --json
openclaw infer embedding create --text "تیکت پشتیبانی مشتری: ارسال با تأخیر" --model openai/text-embedding-3-large --json
openclaw infer embedding providers --json
```

## خروجی JSON

فرمان‌های Infer خروجی JSON را در یک پوش مشترک یکدست می‌کنند:

```json
{
  "ok": true,
  "capability": "image.generate",
  "transport": "local",
  "provider": "openai",
  "model": "gpt-image-2",
  "attempts": [],
  "outputs": []
}
```

فیلدهای پایدار سطح بالا:

- `ok`
- `capability`
- `transport`
- `provider`
- `model`
- `attempts`
- `inputs` (پیوست‌های تصویری ارسال‌شده همراه درخواست، در صورت کاربرد)
- `outputs`
- `ignoredOverrides` (کلیدهای راهنمایی که ارائه‌دهنده از آن‌ها پشتیبانی نمی‌کند، در صورت کاربرد)
- `error`

برای فرمان‌های رسانهٔ تولیدشده، `outputs` شامل فایل‌هایی است که OpenClaw نوشته است. برای خودکارسازی، به‌جای تجزیهٔ stdout خوانا برای انسان، از `path`، `mimeType`، `size` و هر بُعد ویژهٔ رسانه در آن آرایه استفاده کنید.

## خطاهای رایج

```bash
# نادرست
openclaw infer media image generate --prompt "خرچنگ مهربان"

# درست
openclaw infer image generate --prompt "خرچنگ مهربان"
```

```bash
# نادرست
openclaw infer audio transcribe --file ./memo.m4a --model whisper-1 --json

# درست
openclaw infer audio transcribe --file ./memo.m4a --model openai/whisper-1 --json
```

## مرتبط

- [مرجع CLI](/fa/cli)
- [مدل‌ها](/fa/concepts/models)
