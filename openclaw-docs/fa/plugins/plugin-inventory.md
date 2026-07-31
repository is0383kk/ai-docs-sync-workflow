---
read_when:
    - در حال تصمیم‌گیری هستید که آیا یک Plugin همراه با بستهٔ اصلی npm عرضه شود یا جداگانه نصب شود
    - در حال به‌روزرسانی فرادادهٔ بستهٔ Plugin همراه یا خودکارسازی انتشار هستید
    - به فهرست مرجع Plugin‌های داخلی در برابر خارجی نیاز دارید
summary: فهرست تولیدشدهٔ Pluginهای OpenClaw که در هسته عرضه شده‌اند، به‌صورت خارجی منتشر شده‌اند یا فقط در کد منبع نگه‌داری می‌شوند
title: فهرست Pluginها
x-i18n:
    generated_at: "2026-07-27T14:21:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2d835087afbe9d75f883c3db9739f914bedab5ac87a9c20b69c248304b61c594
    source_path: plugins/plugin-inventory.md
    workflow: 16
---

# فهرست Plugin‌ها

این صفحه از `extensions/*/package.json`، `openclaw.plugin.json`،
و موارد استثنای بستهٔ اصلی npm یعنی `files` تولید می‌شود. برای تولید دوبارهٔ آن از دستور زیر استفاده کنید:

```bash
pnpm plugins:inventory:gen
```

## تعریف‌ها

- **بستهٔ اصلی npm:** در بستهٔ npm با نام `openclaw` تعبیه شده و بدون نصب جداگانهٔ Plugin در دسترس است.
- **بستهٔ خارجی رسمی:** Plugin نگه‌داری‌شده توسط OpenClaw که از بستهٔ اصلی npm حذف شده، در این فهرست رسمی باقی مانده و در صورت نیاز از طریق ClawHub و/یا npm نصب می‌شود.
- **فقط دریافت کد منبع:** Plugin محلی مخزن که از مصنوعات منتشرشدهٔ npm حذف شده و به‌عنوان بسته‌ای قابل‌نصب معرفی نمی‌شود.

دریافت‌های کد منبع با نصب‌های npm متفاوت‌اند: پس از `pnpm install`، Plugin‌های
همراه از `extensions/<id>` بارگذاری می‌شوند تا ویرایش‌های محلی و وابستگی‌های فضای کاری
محلی بسته در دسترس باشند.

## نصب یک Plugin

برای تشخیص نیاز به نصب، از مسیر نصب هر مدخل استفاده کنید. Plugin‌هایی
که عبارت `included in OpenClaw` را دارند، از قبل در بستهٔ اصلی موجودند.
بسته‌های خارجی رسمی به یک‌بار نصب و سپس راه‌اندازی مجدد Gateway نیاز دارند.

برای مثال، Discord یک بستهٔ خارجی رسمی است:

```bash
openclaw plugins install @openclaw/discord
openclaw gateway restart
openclaw plugins inspect discord --runtime --json
```

در دورهٔ گذار راه‌اندازی، مشخصات معمول بسته بدون پیشوند همچنان از npm نصب می‌شوند.
هنگامی که به منبعی صریح نیاز دارید، از `clawhub:@openclaw/discord` یا `npm:@openclaw/discord`
استفاده کنید. پس از نصب، برای افزودن اطلاعات اعتبارسنجی و پیکربندی کانال، راهنمای راه‌اندازی
Plugin، مانند [Discord](/fa/channels/discord)، را دنبال کنید. برای فرمان‌های به‌روزرسانی، حذف نصب و انتشار،
به [مدیریت Plugin‌ها](/fa/plugins/manage-plugins) مراجعه کنید.

هر مدخل، بسته، مسیر توزیع و توضیحات را فهرست می‌کند.

## بستهٔ اصلی npm

70 Plugin

- **[admin-http-rpc](/fa/plugins/reference/admin-http-rpc)** (`@openclaw/admin-http-rpc`) - در OpenClaw گنجانده شده است. نقطهٔ پایانی RPC مدیریتی HTTP در OpenClaw.

- **[alibaba](/fa/plugins/reference/alibaba)** (`@openclaw/alibaba-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ تولید ویدئو را اضافه می‌کند.

- **[anthropic](/fa/plugins/reference/anthropic)** (`@openclaw/anthropic-provider`) - در OpenClaw گنجانده شده است. مدل‌های Anthropic،‏ Claude CLI و فهرست بومی نشست‌های Claude.

- **[azure-speech](/fa/plugins/reference/azure-speech)** (`@openclaw/azure-speech`) - در OpenClaw گنجانده شده است. تبدیل متن به گفتار Azure AI Speech‏ (MP3، یادداشت‌های صوتی بومی Ogg/Opus و PCM برای تلفن).

- **[bonjour](/fa/plugins/reference/bonjour)** (`@openclaw/bonjour`) - در OpenClaw گنجانده شده است. Gateway محلی OpenClaw را از طریق Bonjour/mDNS معرفی می‌کند.

- **[browser](/fa/plugins/reference/browser)** (`@openclaw/browser-plugin`) - در OpenClaw گنجانده شده است. ابزارهای قابل‌فراخوانی توسط عامل را اضافه می‌کند.

- **[byteplus](/fa/plugins/reference/byteplus)** (`@openclaw/byteplus-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل BytePlus و BytePlus Plan را به OpenClaw اضافه می‌کند.

- **[canvas](/fa/plugins/reference/canvas)** (`@openclaw/canvas-plugin`) - در OpenClaw گنجانده شده است. سطوح آزمایشی کنترل Canvas و رندر A2UI برای Nodeهای جفت‌شده.

- **[clawrouter](/fa/plugins/reference/clawrouter)** (`@openclaw/clawrouter`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل ClawRouter را به OpenClaw اضافه می‌کند.

- **[cohere](/fa/plugins/reference/cohere)** (`@openclaw/cohere-provider`) - در OpenClaw گنجانده شده است؛ npm؛ ClawHub: `clawhub:@openclaw/cohere-provider`.‏ Plugin ارائه‌دهندهٔ Cohere برای OpenClaw.

- **[comfy](/fa/plugins/reference/comfy)** (`@openclaw/comfy-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل ComfyUI را به OpenClaw اضافه می‌کند.

- **[copilot-proxy](/fa/plugins/reference/copilot-proxy)** (`@openclaw/copilot-proxy`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Copilot Proxy را به OpenClaw اضافه می‌کند.

- **[crabbox](/fa/plugins/reference/crabbox)** (`@openclaw/crabbox-provider`) - در OpenClaw گنجانده شده است. ارائه‌دهندهٔ کارگر ابری مبتنی بر Crabbox CLI.

- **[cua-computer](/fa/plugins/reference/cua-computer)** (`@openclaw/cua-computer`) - در OpenClaw گنجانده شده است. کنترل آزمایشی رایانه با cua-driver برای میزبان‌های Node ویندوز و لینوکس.

- **[deepgram](/fa/plugins/reference/deepgram)** (`@openclaw/deepgram-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ درک رسانه را اضافه می‌کند. پشتیبانی از ارائه‌دهندهٔ رونویسی بی‌درنگ را اضافه می‌کند.

- **[document-extract](/fa/plugins/reference/document-extract)** (`@openclaw/document-extract-plugin`) - در OpenClaw گنجانده شده است. متن و تصاویر جایگزین صفحات را از پیوست‌های سند محلی استخراج می‌کند.

- **[duckduckgo](/fa/plugins/reference/duckduckgo)** (`@openclaw/duckduckgo-plugin`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ جست‌وجوی وب را اضافه می‌کند.

- **[elevenlabs](/fa/plugins/reference/elevenlabs)** (`@openclaw/elevenlabs-speech`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ درک رسانه را اضافه می‌کند. پشتیبانی از ارائه‌دهندهٔ رونویسی بی‌درنگ را اضافه می‌کند. پشتیبانی از ارائه‌دهندهٔ تبدیل متن به گفتار را اضافه می‌کند.

- **[fal](/fa/plugins/reference/fal)** (`@openclaw/fal-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل fal را به OpenClaw اضافه می‌کند.

- **[file-transfer](/fa/plugins/reference/file-transfer)** (`@openclaw/file-transfer`) - در OpenClaw گنجانده شده است. فایل‌ها را با فرمان‌های اختصاصی Node در Nodeهای جفت‌شده دریافت، فهرست و ذخیره می‌کند. با استفاده از base64 روی node.invoke برای فایل‌های دودویی تا سقف 16 MB، محدودیت برش خروجی استاندارد bash را دور می‌زند.

- **[github-copilot](/fa/plugins/reference/github-copilot)** (`@openclaw/github-copilot-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل GitHub Copilot را به OpenClaw اضافه می‌کند.

- **[google](/fa/plugins/reference/google)** (`@openclaw/google-plugin`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل Google،‏ Google Gemini CLI و Google Vertex را به OpenClaw اضافه می‌کند.

- **[huggingface](/fa/plugins/reference/huggingface)** (`@openclaw/huggingface-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Hugging Face را به OpenClaw اضافه می‌کند.

- **[imessage](/fa/plugins/reference/imessage)** (`@openclaw/imessage`) - در OpenClaw گنجانده شده است. سطح کانال iMessage را برای ارسال و دریافت پیام‌های OpenClaw اضافه می‌کند.

- **[linux-canvas](/fa/plugins/reference/linux-canvas)** (`@openclaw/linux-canvas`) - در OpenClaw گنجانده شده است. پل رندر Canvas برای برنامهٔ دسکتاپ لینوکس OpenClaw.

- **[linux-node](/fa/plugins/reference/linux-node)** (`@openclaw/linux-node`) - در OpenClaw گنجانده شده است. اعلان‌های دسکتاپ، تصویربرداری دوربین و موقعیت مکانی برای میزبان‌های Node لینوکس.

- **[litellm](/fa/plugins/reference/litellm)** (`@openclaw/litellm-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل LiteLLM را به OpenClaw اضافه می‌کند.

- **[llm-task](/fa/plugins/reference/llm-task)** (`@openclaw/llm-task`) - در OpenClaw گنجانده شده است. ابزار عمومی LLM فقط مبتنی بر JSON برای وظایف ساختاریافته که از گردش‌های کاری قابل‌فراخوانی است.

- **[lmstudio](/fa/plugins/reference/lmstudio)** (`@openclaw/lmstudio-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل LM Studio را به OpenClaw اضافه می‌کند.

- **[logbook](/fa/plugins/reference/logbook)** (`@openclaw/logbook`) - در OpenClaw گنجانده شده است. دفتر کار خودکار: در بازه‌های زمانی مشخص از صفحهٔ نمایش یک Node جفت‌شده عکس می‌گیرد و آن‌ها را به خط زمانی قابل‌بازبینی از روز شما تبدیل می‌کند.

- **[memory-core](/fa/plugins/reference/memory-core)** (`@openclaw/memory-core`) - در OpenClaw گنجانده شده است. ابزارهای قابل‌فراخوانی توسط عامل را اضافه می‌کند.

- **[memory-wiki](/fa/plugins/reference/memory-wiki)** (`@openclaw/memory-wiki`) - در OpenClaw گنجانده شده است. کامپایلر ویکی پایدار و مخزن دانش سازگار با Obsidian برای OpenClaw.

- **[meta](/fa/plugins/reference/meta)** (`@openclaw/meta-provider`) - در OpenClaw گنجانده شده است؛ npm؛ ClawHub: `clawhub:@openclaw/meta-provider`. پشتیبانی از ارائه‌دهندهٔ مدل Meta را به OpenClaw اضافه می‌کند.

- **[microsoft](/fa/plugins/reference/microsoft)** (`@openclaw/microsoft-speech`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ تبدیل متن به گفتار را اضافه می‌کند.

- **[microsoft-foundry](/fa/plugins/reference/microsoft-foundry)** (`@openclaw/microsoft-foundry`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Microsoft Foundry را به OpenClaw اضافه می‌کند.

- **[migrate-claude](/fa/plugins/reference/migrate-claude)** (`@openclaw/migrate-claude`) - در OpenClaw گنجانده شده است. دستورالعمل‌های Claude Code و Claude Desktop، سرورهای MCP، مهارت‌ها و پیکربندی ایمن را به OpenClaw وارد می‌کند.

- **[migrate-hermes](/fa/plugins/reference/migrate-hermes)** (`@openclaw/migrate-hermes`) - در OpenClaw گنجانده شده است. پیکربندی، حافظه‌ها، مهارت‌ها و اطلاعات اعتبارسنجی پشتیبانی‌شدهٔ Hermes را به OpenClaw وارد می‌کند.

- **[minimax](/fa/plugins/reference/minimax)** (`@openclaw/minimax-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل MiniMax و MiniMax Portal را به OpenClaw اضافه می‌کند.

- **[mistral](/fa/plugins/reference/mistral)** (`@openclaw/mistral-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Mistral را به OpenClaw اضافه می‌کند.

- **[novita](/fa/plugins/reference/novita)** (`@openclaw/novita-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل Novita،‏ Novita AI و Novitaai را به OpenClaw اضافه می‌کند.

- **[nvidia](/fa/plugins/reference/nvidia)** (`@openclaw/nvidia-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل NVIDIA را به OpenClaw اضافه می‌کند.

- **[oc-path](/fa/plugins/reference/oc-path)** (`@openclaw/oc-path`) - در OpenClaw گنجانده شده است. CLI مسیر openclaw را برای آدرس‌دهی فایل‌های فضای کاری با oc:// اضافه می‌کند.

- **[ollama](/fa/plugins/reference/ollama)** (`@openclaw/ollama-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل Ollama و Ollama Cloud را به OpenClaw اضافه می‌کند.

- **[onepassword](/fa/plugins/reference/onepassword)** (`@openclaw/onepassword`) - در OpenClaw گنجانده شده است. کارگزار گزینش‌شدهٔ اسرار 1Password با خط‌مشی تأیید و سابقهٔ حسابرسی SQLite.

- **[open-prose](/fa/plugins/reference/open-prose)** (`@openclaw/open-prose`) - در OpenClaw گنجانده شده است. بستهٔ مهارت OpenProse VM همراه با فرمان اسلش /prose.

- **[openai](/fa/plugins/reference/openai)** (`@openclaw/openai-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل OpenAI را به OpenClaw اضافه می‌کند.

- **[opencode](/fa/plugins/reference/opencode)** (`@openclaw/opencode-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل OpenCode را به OpenClaw اضافه می‌کند.

- **[opencode-go](/fa/plugins/reference/opencode-go)** (`@openclaw/opencode-go-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل OpenCode Go را به OpenClaw اضافه می‌کند.

- **[openrouter](/fa/plugins/reference/openrouter)** (`@openclaw/openrouter-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل OpenRouter را به OpenClaw اضافه می‌کند.

- **[policy](/fa/plugins/reference/policy)** (`@openclaw/policy`) - در OpenClaw گنجانده شده است. بررسی‌های doctor مبتنی بر خط‌مشی را برای انطباق فضای کاری اضافه می‌کند.

- **[reef](/fa/plugins/reference/reef)** (`@openclaw/reef`) - در OpenClaw گنجانده شده است. کانال claw محافظت‌شده با رمزنگاری سرتاسری.

- **[runway](/fa/plugins/reference/runway)** (`@openclaw/runway-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ تولید ویدئو را اضافه می‌کند.

- **[senseaudio](/fa/plugins/reference/senseaudio)** (`@openclaw/senseaudio-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ درک رسانه را اضافه می‌کند.

- **[sglang](/fa/plugins/reference/sglang)** (`@openclaw/sglang-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل SGLang را به OpenClaw اضافه می‌کند.

- **[synthetic](/fa/plugins/reference/synthetic)** (`@openclaw/synthetic-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Synthetic را به OpenClaw اضافه می‌کند.

- **[teams-meetings](/fa/plugins/reference/teams-meetings)** (`@openclaw/teams-meetings`) - در OpenClaw گنجانده شده است. به‌عنوان مهمان مرورگر Chrome به جلسه‌های Microsoft Teams می‌پیوندد.

- **[telegram](/fa/plugins/reference/telegram)** (`@openclaw/telegram`) - در OpenClaw گنجانده شده است. سطح کانال Telegram را برای ارسال و دریافت پیام‌های OpenClaw اضافه می‌کند.

- **[together](/fa/plugins/reference/together)** (`@openclaw/together-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ مدل Together را به OpenClaw اضافه می‌کند.

- **[tts-local-cli](/fa/plugins/reference/tts-local-cli)** (`@openclaw/tts-local-cli`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندهٔ تبدیل متن به گفتار را اضافه می‌کند.

- **[vault](/fa/plugins/reference/vault)** (`@openclaw/vault`) - در OpenClaw گنجانده شده است. یکپارچه‌سازی ارائه‌دهنده SecretRef برای HashiCorp Vault.

- **[vllm](/fa/plugins/reference/vllm)** (`@openclaw/vllm-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهنده مدل vLLM را به OpenClaw می‌افزاید.

- **[volcengine](/fa/plugins/reference/volcengine)** (`@openclaw/volcengine-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل Volcengine و Volcengine Plan را به OpenClaw می‌افزاید.

- **[voyage](/fa/plugins/reference/voyage)** (`@openclaw/voyage-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهنده تعبیه‌سازی حافظه را می‌افزاید.

- **[vydra](/fa/plugins/reference/vydra)** (`@openclaw/vydra-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهنده مدل Vydra را به OpenClaw می‌افزاید.

- **[web-readability](/fa/plugins/reference/web-readability)** (`@openclaw/web-readability-plugin`) - در OpenClaw گنجانده شده است. محتوای خوانای مقاله را از پاسخ‌های محلی واکشی وب HTML استخراج می‌کند.

- **[webhooks](/fa/plugins/reference/webhooks)** (`@openclaw/webhooks`) - در OpenClaw گنجانده شده است. Webhookهای ورودی احراز هویت‌شده که اتوماسیون خارجی را به TaskFlowهای OpenClaw متصل می‌کنند.

- **[workboard](/fa/plugins/reference/workboard)** (`@openclaw/workboard`) - در OpenClaw گنجانده شده است. تابلوی کار داشبورد برای مسائل و نشست‌های تحت مالکیت عامل.

- **[xai](/fa/plugins/reference/xai)** (`@openclaw/xai-plugin`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهنده مدل xAI را به OpenClaw می‌افزاید.

- **[xiaomi](/fa/plugins/reference/xiaomi)** (`@openclaw/xiaomi-provider`) - در OpenClaw گنجانده شده است. پشتیبانی از ارائه‌دهندگان مدل Xiaomi و Xiaomi Token Plan را به OpenClaw می‌افزاید.

- **[zoom-meetings](/plugins/reference/zoom-meetings)** (`@openclaw/zoom-meetings`) - در OpenClaw گنجانده شده است. به‌عنوان مهمان مرورگر Chrome به جلسه‌های Zoom می‌پیوندد.

## بسته‌های خارجی رسمی

72 پلاگین

- **[acpx](/fa/plugins/reference/acpx)** (`@openclaw/acpx`) - npm؛ ClawHub. بک‌اند زمان اجرای ACP برای OpenClaw با مدیریت نشست و انتقال تحت مالکیت پلاگین.

- **[amazon-bedrock](/fa/plugins/reference/amazon-bedrock)** (`@openclaw/amazon-bedrock-provider`) - npm؛ ClawHub. پلاگین ارائه‌دهنده Amazon Bedrock برای OpenClaw با پشتیبانی از کشف مدل، تعبیه‌سازی‌ها و حفاظ‌ها.

- **[amazon-bedrock-mantle](/fa/plugins/reference/amazon-bedrock-mantle)** (`@openclaw/amazon-bedrock-mantle-provider`) - npm؛ ClawHub. پلاگین ارائه‌دهنده Amazon Bedrock Mantle برای OpenClaw جهت مسیریابی مدل سازگار با OpenAI.

- **[anthropic-vertex](/fa/plugins/reference/anthropic-vertex)** (`@openclaw/anthropic-vertex-provider`) - npm؛ ClawHub. پلاگین ارائه‌دهنده Anthropic Vertex برای OpenClaw جهت استفاده از مدل‌های Claude در Google Vertex AI.

- **[arcee](/fa/plugins/reference/arcee)** (`@openclaw/arcee-provider`) - npm؛ ClawHub: `clawhub:@openclaw/arcee-provider`. پشتیبانی از ارائه‌دهنده مدل Arcee را به OpenClaw می‌افزاید.

- **[baseten](/plugins/reference/baseten)** (`@openclaw/baseten-provider`) - npm؛ ClawHub: `clawhub:@openclaw/baseten-provider`. پلاگین ارائه‌دهنده Baseten برای OpenClaw.

- **[brave](/fa/plugins/reference/brave)** (`@openclaw/brave-plugin`) - npm؛ ClawHub. پلاگین ارائه‌دهنده Brave Search برای جست‌وجوی وب در OpenClaw.

- **[cerebras](/fa/plugins/reference/cerebras)** (`@openclaw/cerebras-provider`) - npm؛ ClawHub: `clawhub:@openclaw/cerebras-provider`. پشتیبانی از ارائه‌دهنده مدل Cerebras را به OpenClaw می‌افزاید.

- **[chutes](/fa/plugins/reference/chutes)** (`@openclaw/chutes-provider`) - npm؛ ClawHub: `clawhub:@openclaw/chutes-provider`. پشتیبانی از ارائه‌دهنده مدل Chutes را به OpenClaw می‌افزاید.

- **[clickclack](/fa/plugins/reference/clickclack)** (`@openclaw/clickclack`) - npm؛ ClawHub: `clawhub:@openclaw/clickclack`. سطح کانال Clickclack را برای ارسال و دریافت پیام‌های OpenClaw می‌افزاید.

- **[cloudflare-ai-gateway](/fa/plugins/reference/cloudflare-ai-gateway)** (`@openclaw/cloudflare-ai-gateway-provider`) - npm؛ ClawHub: `clawhub:@openclaw/cloudflare-ai-gateway-provider`. پشتیبانی از ارائه‌دهنده مدل Cloudflare AI Gateway را به OpenClaw می‌افزاید.

- **[codex](/fa/plugins/reference/codex)** (`@openclaw/codex`) - npm؛ ClawHub. چارچوب app-server برای Codex و کاتالوگ بومی نشست‌ها.

- **[copilot](/fa/plugins/reference/copilot)** (`@openclaw/copilot`) - npm؛ ClawHub: `clawhub:@openclaw/copilot`. زمان اجرای عامل GitHub Copilot را ثبت می‌کند.

- **[deepinfra](/fa/plugins/reference/deepinfra)** (`@openclaw/deepinfra-provider`) - npm؛ ClawHub: `clawhub:@openclaw/deepinfra-provider`. پشتیبانی از ارائه‌دهنده مدل DeepInfra را به OpenClaw می‌افزاید.

- **[deepseek](/fa/plugins/reference/deepseek)** (`@openclaw/deepseek-provider`) - npm؛ ClawHub: `clawhub:@openclaw/deepseek-provider`. پشتیبانی از ارائه‌دهنده مدل DeepSeek را به OpenClaw می‌افزاید.

- **[diagnostics-otel](/fa/plugins/reference/diagnostics-otel)** (`@openclaw/diagnostics-otel`) - npm؛ ClawHub: `clawhub:@openclaw/diagnostics-otel`. صادرکننده OpenTelemetry برای عیب‌یابی OpenClaw جهت معیارها، ردگیری‌ها و گزارش‌ها.

- **[diagnostics-prometheus](/fa/plugins/reference/diagnostics-prometheus)** (`@openclaw/diagnostics-prometheus`) - npm؛ ClawHub: `clawhub:@openclaw/diagnostics-prometheus`. صادرکننده Prometheus برای عیب‌یابی OpenClaw جهت معیارهای زمان اجرا.

- **[diffs](/fa/plugins/reference/diffs)** (`@openclaw/diffs`) - npm؛ ClawHub. پلاگین نمایشگر فقط‌خواندنی تفاوت‌ها و رندرکننده فایل برای عامل‌ها در OpenClaw.

- **[diffs-language-pack](/fa/plugins/reference/diffs-language-pack)** (`@openclaw/diffs-language-pack`) - npm؛ ClawHub: `clawhub:@openclaw/diffs-language-pack`. برجسته‌سازی نحو را برای زبان‌هایی خارج از مجموعه پیش‌فرض نمایشگر تفاوت‌ها می‌افزاید.

- **[discord](/fa/plugins/reference/discord)** (`@openclaw/discord`) - npm؛ ClawHub. پلاگین کانال Discord برای OpenClaw جهت کانال‌ها، پیام‌های مستقیم، فرمان‌ها و رویدادهای برنامه.

- **[exa](/fa/plugins/reference/exa)** (`@openclaw/exa-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/exa-plugin`. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را می‌افزاید.

- **[featherless](/fa/plugins/reference/featherless)** (`@openclaw/featherless-provider`) - npm؛ ClawHub: `clawhub:@openclaw/featherless-provider`. پلاگین ارائه‌دهنده Featherless AI برای OpenClaw.

- **[feishu](/fa/plugins/reference/feishu)** (`@openclaw/feishu`) - npm؛ ClawHub. پلاگین کانال Feishu/Lark برای OpenClaw جهت گفت‌وگوها و ابزارهای محیط کار (با نگه‌داری جامعه توسط @m1heng).

- **[firecrawl](/fa/plugins/reference/firecrawl)** (`@openclaw/firecrawl-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/firecrawl-plugin`. ابزارهای قابل فراخوانی توسط عامل را می‌افزاید. پشتیبانی از ارائه‌دهنده واکشی وب را می‌افزاید. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را می‌افزاید.

- **[fireworks](/fa/plugins/reference/fireworks)** (`@openclaw/fireworks-provider`) - npm؛ ClawHub: `clawhub:@openclaw/fireworks-provider`. پشتیبانی از ارائه‌دهنده مدل Fireworks را به OpenClaw می‌افزاید.

- **[gmi](/fa/plugins/reference/gmi)** (`@openclaw/gmi-provider`) - npm؛ ClawHub: `clawhub:@openclaw/gmi-provider`. پلاگین ارائه‌دهنده GMI Cloud برای OpenClaw.

- **[google-meet](/fa/plugins/reference/google-meet)** (`@openclaw/google-meet`) - npm؛ ClawHub. پلاگین شرکت‌کننده Google Meet برای OpenClaw جهت پیوستن به تماس‌ها از طریق انتقال‌های Chrome یا Twilio.

- **[googlechat](/fa/plugins/reference/googlechat)** (`@openclaw/googlechat`) - npm؛ ClawHub. پلاگین کانال Google Chat برای OpenClaw جهت فضاها و پیام‌های مستقیم.

- **[gradium](/fa/plugins/reference/gradium)** (`@openclaw/gradium-speech`) - npm؛ ClawHub: `clawhub:@openclaw/gradium-speech`. پشتیبانی از ارائه‌دهنده تبدیل متن به گفتار را می‌افزاید.

- **[groq](/fa/plugins/reference/groq)** (`@openclaw/groq-provider`) - npm؛ ClawHub: `clawhub:@openclaw/groq-provider`. پشتیبانی از ارائه‌دهنده مدل Groq را به OpenClaw می‌افزاید.

- **[inworld](/fa/plugins/reference/inworld)** (`@openclaw/inworld-speech`) - npm؛ ClawHub: `clawhub:@openclaw/inworld-speech`. تبدیل جریانی متن به گفتار Inworld ‏(MP3، OGG_OPUS، PCM تلفنی).

- **[irc](/fa/plugins/reference/irc)** (`@openclaw/irc`) - npm؛ ClawHub: `clawhub:@openclaw/irc`. سطح کانال IRC را برای ارسال و دریافت پیام‌های OpenClaw می‌افزاید.

- **[kilocode](/fa/plugins/reference/kilocode)** (`@openclaw/kilocode-provider`) - npm؛ ClawHub: `clawhub:@openclaw/kilocode-provider`. پشتیبانی از ارائه‌دهنده مدل Kilocode را به OpenClaw می‌افزاید.

- **[kimi](/fa/plugins/reference/kimi)** (`@openclaw/kimi-provider`) - npm؛ ClawHub: `clawhub:@openclaw/kimi-provider`. پشتیبانی از ارائه‌دهندگان مدل Kimi و Kimi Coding را به OpenClaw می‌افزاید.

- **[line](/fa/plugins/reference/line)** (`@openclaw/line`) - npm؛ ClawHub. پلاگین کانال LINE برای OpenClaw جهت گفت‌وگوهای LINE Bot API.

- **[llama-cpp](/fa/plugins/reference/llama-cpp)** (`@openclaw/llama-cpp-provider`) - npm؛ ClawHub. استنتاج متن و تعبیه‌سازی محلی GGUF از طریق node-llama-cpp.

- **[lobster](/fa/plugins/reference/lobster)** (`@openclaw/lobster`) - npm؛ ClawHub. پلاگین ابزار گردش کار Lobster برای پایپ‌لاین‌های نوع‌دار و تأییدهای قابل‌ازسرگیری.

- **[longcat](/fa/plugins/reference/longcat)** (`@openclaw/longcat-provider`) - npm؛ ClawHub: `clawhub:@openclaw/longcat-provider`. پلاگین ارائه‌دهنده LongCat برای OpenClaw.

- **[matrix](/fa/plugins/reference/matrix)** (`@openclaw/matrix`) - ClawHub: `clawhub:@openclaw/matrix`؛ npm. پلاگین کانال Matrix برای OpenClaw جهت اتاق‌ها و پیام‌های مستقیم.

- **[mattermost](/fa/plugins/reference/mattermost)** (`@openclaw/mattermost`) - npm؛ ClawHub: `clawhub:@openclaw/mattermost`. سطح کانال Mattermost را برای ارسال و دریافت پیام‌های OpenClaw می‌افزاید.

- **[memory-lancedb](/fa/plugins/reference/memory-lancedb)** (`@openclaw/memory-lancedb`) - npm؛ ClawHub. پلاگین حافظه بلندمدت OpenClaw با پشتیبانی LanceDB و قابلیت‌های یادآوری خودکار، ثبت خودکار و جست‌وجوی برداری.

- **[moonshot](/fa/plugins/reference/moonshot)** (`@openclaw/moonshot-provider`) - npm؛ ClawHub: `clawhub:@openclaw/moonshot-provider`. پشتیبانی از ارائه‌دهنده مدل Moonshot را به OpenClaw می‌افزاید.

- **[msteams](/fa/plugins/reference/msteams)** (`@openclaw/msteams`) - npm؛ ClawHub. پلاگین کانال Microsoft Teams برای OpenClaw جهت مکالمه‌های ربات.

- **[mxc](/fa/plugins/reference/mxc)** (`@openclaw/mxc-sandbox`) - npm؛ ClawHub. اجرای ابزار در محیط ایزوله سطح سیستم‌عامل از طریق MXC: فرمان‌ها را در یک Windows ProcessContainer با فایل‌های پیکربندی‌شده خط‌مشی MXC اجرا می‌کند.

- **[nextcloud-talk](/fa/plugins/reference/nextcloud-talk)** (`@openclaw/nextcloud-talk`) - npm؛ ClawHub. پلاگین کانال Nextcloud Talk برای OpenClaw جهت مکالمه‌ها.

- **[nostr](/fa/plugins/reference/nostr)** (`@openclaw/nostr`) - npm؛ ClawHub. پلاگین کانال Nostr برای OpenClaw جهت پیام‌های مستقیم رمزگذاری‌شده NIP-04.

- **[openshell](/fa/plugins/reference/openshell)** (`@openclaw/openshell-sandbox`) - npm؛ ClawHub. بک‌اند محیط ایزوله OpenClaw برای NVIDIA OpenShell CLI با فضاهای کاری محلی همسان‌سازی‌شده و اجرای فرمان از طریق SSH.

- **[parallel](/fa/tools/parallel-search)** (`@openclaw/parallel-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/parallel-plugin`. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را می‌افزاید.

- **[perplexity](/fa/plugins/reference/perplexity)** (`@openclaw/perplexity-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/perplexity-plugin`. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را می‌افزاید.

- **[pixverse](/fa/plugins/reference/pixverse)** (`@openclaw/pixverse-provider`) - npm؛ ClawHub: `clawhub:@openclaw/pixverse-provider`. پلاگین ارائه‌دهنده تولید ویدئوی PixVerse برای OpenClaw.

- **[qianfan](/fa/plugins/reference/qianfan)** (`@openclaw/qianfan-provider`) - npm؛ ClawHub: `clawhub:@openclaw/qianfan-provider`. پشتیبانی از ارائه‌دهنده مدل Qianfan را به OpenClaw می‌افزاید.

- **[qqbot](/fa/plugins/reference/qqbot)** (`@openclaw/qqbot`) - npm؛ ClawHub. پلاگین کانال QQ Bot برای OpenClaw جهت گردش کارهای گروهی و پیام مستقیم.

- **[qwen](/fa/plugins/reference/qwen)** (`@openclaw/qwen-provider`) - npm؛ ClawHub: `clawhub:@openclaw/qwen-provider`. پشتیبانی از ارائه‌دهندگان مدل Qwen، Qwen Cloud، Model Studio، DashScope، Qwen Token Plan و Bailian Token Plan را به OpenClaw می‌افزاید.

- **[raft](/fa/plugins/reference/raft)** (`@openclaw/raft`) - npm؛ ClawHub. پلاگین کانال Raft برای OpenClaw جهت پل‌های امن بیدارسازی CLI.

- **[searxng](/fa/plugins/reference/searxng)** (`@openclaw/searxng-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/searxng-plugin`. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را می‌افزاید.

- **[signal](/fa/plugins/reference/signal)** (`@openclaw/signal`) - npm؛ ClawHub: `clawhub:@openclaw/signal`. سطح کانال Signal را برای ارسال و دریافت پیام‌های OpenClaw می‌افزاید.

- **[slack](/fa/plugins/reference/slack)** (`@openclaw/slack`) - npm؛ ClawHub. پلاگین کانال Slack برای OpenClaw جهت کانال‌ها، پیام‌های مستقیم، فرمان‌ها و رویدادهای برنامه.

- **[sms](/fa/plugins/reference/sms)** (`@openclaw/sms`) - npm؛ ClawHub: `clawhub:@openclaw/sms`. Plugin کانال پیامک Twilio برای پیام‌های متنی OpenClaw.

- **[stepfun](/fa/plugins/reference/stepfun)** (`@openclaw/stepfun-provider`) - npm؛ ClawHub: `clawhub:@openclaw/stepfun-provider`. پشتیبانی از ارائه‌دهندگان مدل StepFun و StepFun Plan را به OpenClaw اضافه می‌کند.

- **[synology-chat](/fa/plugins/reference/synology-chat)** (`@openclaw/synology-chat`) - npm؛ ClawHub. Plugin کانال Synology Chat برای کانال‌ها و پیام‌های مستقیم OpenClaw.

- **[tavily](/fa/plugins/reference/tavily)** (`@openclaw/tavily-plugin`) - npm؛ ClawHub: `clawhub:@openclaw/tavily-plugin`. ابزارهای قابل فراخوانی توسط عامل را اضافه می‌کند. پشتیبانی از ارائه‌دهنده جست‌وجوی وب را اضافه می‌کند.

- **[tencent](/fa/plugins/reference/tencent)** (`@openclaw/tencent-provider`) - npm؛ ClawHub: `clawhub:@openclaw/tencent-provider`. پشتیبانی از ارائه‌دهندگان مدل Tencent TokenHub و Tencent Tokenplan را به OpenClaw اضافه می‌کند.

- **[tlon](/fa/plugins/reference/tlon)** (`@openclaw/tlon`) - npm؛ ClawHub. Plugin کانال Tlon/Urbit در OpenClaw برای گردش‌کارهای گفت‌وگو.

- **[tokenjuice](/fa/plugins/reference/tokenjuice)** (`@openclaw/tokenjuice`) - npm؛ ClawHub: `clawhub:@openclaw/tokenjuice`. نتایج ابزارهای exec و bash را با کاهش‌دهنده‌های Tokenjuice فشرده می‌کند.

- **[twitch](/fa/plugins/reference/twitch)** (`@openclaw/twitch`) - npm؛ ClawHub. Plugin کانال Twitch در OpenClaw برای گردش‌کارهای گفت‌وگو و نظارت.

- **[venice](/fa/plugins/reference/venice)** (`@openclaw/venice-provider`) - npm؛ ClawHub: `clawhub:@openclaw/venice-provider`. پشتیبانی از ارائه‌دهنده مدل Venice را به OpenClaw اضافه می‌کند.

- **[vercel-ai-gateway](/fa/plugins/reference/vercel-ai-gateway)** (`@openclaw/vercel-ai-gateway-provider`) - npm؛ ClawHub: `clawhub:@openclaw/vercel-ai-gateway-provider`. پشتیبانی از ارائه‌دهنده مدل Vercel AI Gateway را به OpenClaw اضافه می‌کند.

- **[voice-call](/fa/plugins/reference/voice-call)** (`@openclaw/voice-call`) - npm؛ ClawHub. Plugin تماس صوتی OpenClaw برای تماس‌های تلفنی Twilio، Telnyx و Plivo.

- **[whatsapp](/fa/plugins/reference/whatsapp)** (`@openclaw/whatsapp`) - ClawHub: `clawhub:@openclaw/whatsapp`؛ npm. Plugin کانال WhatsApp در OpenClaw برای گفت‌وگوهای WhatsApp Web.

- **[zai](/fa/plugins/reference/zai)** (`@openclaw/zai-provider`) - npm؛ ClawHub: `clawhub:@openclaw/zai-provider`. پشتیبانی از ارائه‌دهنده مدل Z.AI را به OpenClaw اضافه می‌کند.

- **[zalo](/fa/plugins/reference/zalo)** (`@openclaw/zalo`) - npm؛ ClawHub. Plugin کانال Zalo در OpenClaw برای گفت‌وگوهای ربات و Webhook.

- **[zalouser](/fa/plugins/reference/zalouser)** (`@openclaw/zalouser`) - npm؛ ClawHub. Plugin حساب شخصی Zalo در OpenClaw از طریق یکپارچه‌سازی بومی zca-js.

## فقط دریافت کد منبع

2 Plugin

- **[qa-channel](/fa/plugins/reference/qa-channel)** (`@openclaw/qa-channel`) - فقط دریافت کد منبع. سطح QA Channel را برای ارسال و دریافت پیام‌های OpenClaw اضافه می‌کند.

- **[qa-lab](/fa/plugins/reference/qa-lab)** (`@openclaw/qa-lab`) - فقط دریافت کد منبع. Plugin آزمایشگاه QA در OpenClaw با رابط کاربری خصوصی اشکال‌زدایی و اجراکننده سناریو.
