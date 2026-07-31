---
read_when:
    - باید بدانید کدام متغیرهای محیطی بارگذاری می‌شوند و ترتیب بارگذاری آن‌ها چیست
    - در حال اشکال‌زدایی کلیدهای API مفقود در Gateway هستید
    - در حال مستندسازی احراز هویت ارائه‌دهنده یا محیط‌های استقرار هستید
summary: محل بارگذاری متغیرهای محیطی توسط OpenClaw و ترتیب اولویت آن‌ها
title: متغیرهای محیطی
x-i18n:
    generated_at: "2026-07-27T15:18:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw متغیرهای محیطی را از چندین منبع دریافت می‌کند. قاعده این است که **هرگز مقادیر موجود بازنویسی نشوند**.
فایل‌های `.env` فضای کاری منبعی با اعتماد کمتر هستند: OpenClaw پیش از اعمال تقدم، اعتبارنامه‌های ارائه‌دهندگان و کنترل‌های محافظت‌شده زمان اجرا را از `.env` فضای کاری نادیده می‌گیرد.

## تقدم (از بالاترین به پایین‌ترین)

1. **محیط فرایند** (آنچه فرایند Gateway از قبل از پوسته/دیمون والد در اختیار دارد).
2. **`.env` در دایرکتوری کاری فعلی** (پیش‌فرض dotenv؛ بازنویسی نمی‌کند؛ اعتبارنامه‌های ارائه‌دهندگان و کنترل‌های محافظت‌شده زمان اجرا نادیده گرفته می‌شوند).
3. **`.env` سراسری** در `~/.openclaw/.env` (همان `$OPENCLAW_STATE_DIR/.env`؛ برای کلیدهای API ارائه‌دهندگان توصیه می‌شود؛ بازنویسی نمی‌کند).
4. **بلوک `env` پیکربندی** در `~/.openclaw/openclaw.json` (فقط در صورت نبود مقدار اعمال می‌شود).
5. **درون‌ریزی اختیاری پوسته ورود** (`env.shellEnv.enabled` یا `OPENCLAW_LOAD_SHELL_ENV=1`) که فقط برای کلیدهای موردانتظارِ فاقد مقدار اعمال می‌شود.

در نصب‌های جدید Ubuntu که از دایرکتوری وضعیت پیش‌فرض استفاده می‌کنند، OpenClaw همچنین `~/.config/openclaw/gateway.env` را پس از `.env` سراسری به‌عنوان یک مسیر جایگزین سازگاری در نظر می‌گیرد. اگر هر دو فایل وجود داشته باشند و با یکدیگر مغایرت داشته باشند، OpenClaw مقدار `~/.openclaw/.env` را نگه می‌دارد و هشداری چاپ می‌کند.

اگر فایل پیکربندی کاملاً وجود نداشته باشد، مرحله 4 رد می‌شود؛ در صورت فعال بودن، درون‌ریزی پوسته همچنان اجرا می‌شود.

## متغیرهای پشتیبانی‌شده برای اپراتورها

متغیرهای زیر قرارداد محیطی پشتیبانی‌شده برای اپراتورها هستند. متغیرهای مستندنشده `OPENCLAW_*` جزئیات داخلی پیاده‌سازی‌اند و ممکن است بدون اطلاع قبلی حذف شوند.

### مسیرها و نمونه‌ها

| متغیر                 | هدف                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | دایرکتوری خانگی مورداستفاده برای پیش‌فرض‌های مسیر OpenClaw را بازنویسی می‌کند.      |
| `OPENCLAW_STATE_DIR`     | دایرکتوری وضعیت تغییرپذیر را بازنویسی می‌کند.                             |
| `OPENCLAW_CONFIG_PATH`   | مسیر فایل پیکربندی فعال را بازنویسی می‌کند.                             |
| `OPENCLAW_WORKSPACE_DIR` | فضای کاری پیش‌فرض عامل را بازنویسی می‌کند.                             |
| `OPENCLAW_PROFILE`       | یک پروفایل نام‌گذاری‌شده و پیش‌فرض‌های ایزوله آن را انتخاب می‌کند.                 |
| `OPENCLAW_GIT_DIR`       | کد منبع دریافت‌شده مورداستفاده برای به‌روزرسانی‌های کانال توسعه را بازنویسی می‌کند. |
| `OPENCLAW_INCLUDE_ROOTS` | به `$include` اجازه می‌دهد از ریشه‌های بیشتری تفکیک شود.                |

### Gateway و احراز هویت

| متغیر                    | هدف                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | نشانی URL راه دور Gateway مورداستفاده کلاینت‌ها را بازنویسی می‌کند.                |
| `OPENCLAW_GATEWAY_PORT`     | درگاه محلی Gateway را بازنویسی می‌کند.                                |
| `OPENCLAW_GATEWAY_TOKEN`    | احراز هویت توکنی را برای سرورها و کلاینت‌های Gateway فراهم می‌کند.    |
| `OPENCLAW_GATEWAY_PASSWORD` | احراز هویت با گذرواژه را برای سرورها و کلاینت‌های Gateway فراهم می‌کند. |

### اعتبارنامه‌های ارائه‌دهندگان

هسته و Pluginهای همراه ارائه‌دهندگان، متغیرهای زیر را برای اعتبارنامه و انتخاب ارائه‌دهنده تشخیص می‌دهند. هنگامی که به‌جای یک مقدار واحد در سراسر فرایند به اعتبارنامه‌های دارای دامنه نیاز دارید، فیلدهای پیکربندی یا SecretRef هر ارائه‌دهنده را ترجیح دهید.

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY` و `Z_AI_API_KEY`.

Pluginهای شخص ثالث نصب‌شده ممکن است متغیرهای اعتبارنامه دیگری را در مانیفست Plugin خود اعلام کنند؛ آن متغیرها قراردادهای Plugin اعلام‌کننده آن‌ها هستند، نه متغیرهای هسته OpenClaw.

### ثبت گزارش و عیب‌یابی

| متغیر                             | هدف                                                       |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | سطوح گزارش فایل و کنسول را بازنویسی می‌کند.                         |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | عیب‌یابی زمان‌بندی انتقال مدل را فعال می‌کند.                    |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | عیب‌یابی محموله مدلِ سانسورشده را انتخاب می‌کند.                    |
| `OPENCLAW_DEBUG_SSE`                 | زمان‌بندی SSE یا عیب‌یابی مشاهده رویداد را انتخاب می‌کند.                  |
| `OPENCLAW_DEBUG_CODE_MODE`           | عیب‌یابی سطح حالت کد را فعال می‌کند.                         |
| `OPENCLAW_DIAGNOSTICS`               | پرچم‌های عیب‌یابی نام‌گذاری‌شده را فعال می‌کند یا همه پرچم‌ها را با `0` غیرفعال می‌کند. |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | مسیر JSONL را برای عیب‌یابی خط زمانی انتخاب می‌کند.               |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | نمونه‌های حلقه رویداد را به عیب‌یابی خط زمانی اضافه می‌کند.               |

### کلیدهای تغییر وضعیت قابلیت‌ها و زمان اجرا

| متغیر                             | هدف                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | متغیرهای موردانتظارِ فاقد مقدار را از پوسته ورود درون‌ریزی می‌کند.                      |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | مهلت زمانی درون‌ریزی پوسته ورود را تنظیم می‌کند.                                          |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | اسنپ‌شات‌های پوسته exec را با `0` غیرفعال می‌کند.                                       |
| `OPENCLAW_OFFLINE`                   | از بارگیری باینری‌های کمکی سنجاق‌شده عامل جلوگیری می‌کند.                           |
| `OPENCLAW_BROWSER_HEADLESS`          | اجرای مرورگر مدیریت‌شده را به‌اجبار دارای رابط (`0`) یا بدون رابط (`1`) می‌کند.               |
| `OPENCLAW_DISABLE_BONJOUR`           | تبلیغ Bonjour را به‌اجبار روشن (`0`) یا خاموش (`1`) می‌کند.                             |
| `OPENCLAW_NO_AUTO_UPDATE`            | اعمال خودکار به‌روزرسانی‌ها را غیرفعال می‌کند.                                            |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | به‌عنوان یک بازنویسی اضطراری، اتصال‌های `ws://` مربوط به DNS خصوصی مورداعتماد را مجاز می‌کند.     |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | ضمن حفظ قفل‌های مالکیت به‌ازای هر وضعیت، چندین فرایند Gateway را مجاز می‌کند. |
| `OPENCLAW_SKIP_CHANNELS`             | برای عیب‌یابی، Gateway را بدون انتقال‌های کانال راه‌اندازی می‌کند.            |
| `OPENCLAW_THEME`                     | پالت TUI را به‌اجبار روی `light` یا `dark` تنظیم می‌کند.                                  |

## اعتبارنامه‌های ارائه‌دهندگان و `.env` فضای کاری

کلیدهای API ارائه‌دهندگان را فقط در `.env` فضای کاری نگه ندارید. OpenClaw مجموعه بزرگی از کلیدهای اعتبارنامه ارائه‌دهندگان و تغییر مسیر نقطه پایانی را در فایل‌های `.env` فضای کاری مسدود می‌کند؛ از جمله همه متغیرهای محیطی شناخته‌شده احراز هویت ارائه‌دهندگان (برای مثال `GEMINI_API_KEY`، `GOOGLE_API_KEY`، `XAI_API_KEY`، `MISTRAL_API_KEY`، `GROQ_API_KEY`، `DEEPSEEK_API_KEY`، `PERPLEXITY_API_KEY`، `BRAVE_API_KEY`، `TAVILY_API_KEY`، `EXA_API_KEY`، `FIRECRAWL_API_KEY`)، همچنین هر کلیدی که به `_API_HOST`، `_BASE_URL`، `_ENDPOINT` یا `_HOMESERVER` ختم شود، و تمام فضاهای نام `OPENCLAW_*`، `CLAWHUB_*`، `ANTHROPIC_API_KEY_*` و `OPENAI_API_KEY_*`.

در عوض، یکی از این منابع مورداعتماد را برای اعتبارنامه‌های ارائه‌دهندگان استفاده کنید:

- محیط فرایند Gateway، مانند پوسته، واحد launchd/systemd، راز کانتینر یا راز CI.
- فایل dotenv سراسری زمان اجرا در `~/.openclaw/.env` یا `$OPENCLAW_STATE_DIR/.env`.
- بلوک `env` پیکربندی در `~/.openclaw/openclaw.json`.
- درون‌ریزی اختیاری پوسته ورود هنگامی که `env.shellEnv.enabled` یا `OPENCLAW_LOAD_SHELL_ENV=1` فعال است.

اگر پیش‌تر کلیدهای ارائه‌دهندگان یا مقادیر مسیریابی نقطه پایانی را فقط در `.env` فضای کاری ذخیره کرده‌اید، آن‌ها را به یکی از منابع مورداعتماد بالا منتقل کنید. `.env` فضای کاری همچنان می‌تواند متغیرهای عادی پروژه را فراهم کند که اعتبارنامه، تغییر مسیر نقطه پایانی، بازنویسی میزبان یا کنترل‌های زمان اجرای `OPENCLAW_*` نیستند.

برای آگاهی از منطق امنیتی، به [فایل‌های `.env` فضای کاری](/fa/gateway/security#workspace-env-files) مراجعه کنید.

## بلوک `env` پیکربندی

دو روش معادل برای تنظیم متغیرهای محیطی درون‌خطی (هیچ‌یک بازنویسی نمی‌کنند):

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

بلوک `env` پیکربندی فقط مقادیر رشته‌ای صریح را می‌پذیرد. مقادیر
`file:...` را بسط نمی‌دهد؛ برای مثال، `XAI_API_KEY: "file:secrets/xai-api-key.txt"`
دقیقاً با همان رشته به ارائه‌دهندگان ارسال می‌شود.

برای کلیدهای ارائه‌دهندگان مبتنی بر فایل، در فیلد اعتبارنامه‌ای که
از آن پشتیبانی می‌کند از SecretRef استفاده کنید:

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

برای فیلدهای پشتیبانی‌شده به [مدیریت رازها](/fa/gateway/secrets) و
[سطح اعتبارنامه SecretRef](/fa/reference/secretref-credential-surface)
مراجعه کنید.

## درون‌ریزی محیط پوسته

`env.shellEnv` پوسته ورود را اجرا می‌کند و فقط کلیدهای موردانتظارِ **فاقد مقدار** را درون‌ریزی می‌کند:

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

معادل‌های متغیر محیطی:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000` (پیش‌فرض `15000`)

## اسنپ‌شات‌های پوسته exec

در میزبان‌های غیر Windows مربوط به Gateway، فرمان‌های `exec` در bash و zsh به‌طور پیش‌فرض از یک اسنپ‌شات آغازین استفاده می‌کنند.
برای غیرفعال کردن این مسیر، `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` را در محیط فرایند Gateway تنظیم کنید.
مقادیر `false`، `no` و `off` نیز آن را غیرفعال می‌کنند. مقادیر `exec.env` هر فراخوانی نمی‌توانند
اسنپ‌شات‌ها را تغییر وضعیت دهند یا کش اسنپ‌شات را تغییر مسیر دهند.

## متغیرهای محیطی تزریق‌شده در زمان اجرا

OpenClaw همچنین نشانگرهای زمینه را به فرایندهای فرزند ایجادشده تزریق می‌کند:

- `OPENCLAW_SHELL=exec`: برای فرمان‌هایی تنظیم می‌شود که از طریق ابزار `exec` اجرا می‌شوند.
- `OPENCLAW_SHELL=acp-client`: برای `openclaw acp client` هنگام ایجاد فرایند پل ACP تنظیم می‌شود.
- `OPENCLAW_SHELL=tui-local`: برای فرمان‌های پوستهٔ `!` در TUI محلی تنظیم می‌شود.
- `OPENCLAW_CLI=1`: برای فرایندهای فرزندی تنظیم می‌شود که نقطهٔ ورود CLI ایجاد می‌کند.

این‌ها نشانگرهای زمان اجرا هستند (نه پیکربندی الزامی کاربر). می‌توان از آن‌ها در منطق پوسته/پروفایل
برای اعمال قواعد مختص هر زمینه استفاده کرد.

## متغیرهای محیطی رابط کاربری

- `OPENCLAW_THEME=light`: وقتی پس‌زمینهٔ ترمینال روشن است، پالت روشن TUI را به‌اجبار فعال می‌کند.
- `OPENCLAW_THEME=dark`: پالت تیرهٔ TUI را به‌اجبار فعال می‌کند.
- `COLORFGBG`: اگر ترمینال این متغیر را صادر کند، OpenClaw از راهنمای رنگ پس‌زمینه برای انتخاب خودکار پالت TUI استفاده می‌کند.

## جای‌گذاری متغیر محیطی در پیکربندی

می‌توان با استفاده از نحو `${VAR_NAME}` مستقیماً به متغیرهای محیطی در مقادیر رشته‌ای پیکربندی ارجاع داد:

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

برای جزئیات کامل، به [پیکربندی: جای‌گذاری متغیر محیطی](/fa/gateway/configuration-reference#env-var-substitution) مراجعه کنید.

## ارجاع‌های محرمانه در برابر رشته‌های `${ENV}`

OpenClaw از دو الگوی مبتنی بر محیط پشتیبانی می‌کند:

- جای‌گذاری رشتهٔ `${VAR}` در مقادیر پیکربندی.
- اشیای SecretRef‏ (`{ source: "env", provider: "default", id: "VAR" }`) برای فیلدهایی که از ارجاع‌های محرمانه پشتیبانی می‌کنند.

هر دو هنگام فعال‌سازی از محیط فرایند تفکیک می‌شوند. جزئیات SecretRef در [مدیریت اطلاعات محرمانه](/fa/gateway/secrets) مستند شده است.
خود بلوک `env` پیکربندی، SecretRefها یا مقادیر مختصر `file:...`
را تفکیک نمی‌کند.

## متغیرهای محیطی مرتبط با مسیر

| متغیر                 | کاربرد                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | پوشهٔ خانهٔ مورداستفاده برای پیش‌فرض‌های مسیر داخلی OpenClaw‏ (`~/.openclaw/`، پوشه‌های عامل، نشست‌ها، اعتبارنامه‌ها، راه‌اندازی اولیهٔ نصب‌کننده و پرداخت پیش‌فرض توسعه) را بازنویسی می‌کند. هنگام اجرای OpenClaw با یک کاربر اختصاصی سرویس مفید است. |
| `OPENCLAW_STATE_DIR`     | پوشهٔ وضعیت را بازنویسی می‌کند (پیش‌فرض `~/.openclaw`).                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | مسیر فایل پیکربندی را بازنویسی می‌کند (پیش‌فرض `~/.openclaw/openclaw.json`).                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | فهرستی از مسیرهای پوشه‌هایی که دستورالعمل‌های `$include` می‌توانند فایل‌های خارج از پوشهٔ پیکربندی را در آن‌ها تفکیک کنند (پیش‌فرض: هیچ‌کدام — `$include` به پوشهٔ پیکربندی محدود است). علامت مد به مسیر خانه بسط داده می‌شود.                                                         |

## بارگیری ابزارهای کمکی عامل

برای جلوگیری از بارگیری فایل‌های اجرایی کمکی پین‌شدهٔ `fd`
و `ripgrep` توسط OpenClaw، متغیر `OPENCLAW_OFFLINE=1` را تنظیم کنید. ابزارهای کمکی موجود در پوشهٔ ابزارهای OpenClaw
و فایل‌های اجرایی عملیاتی سیستم همچنان قابل‌استفاده‌اند؛ ابزار کمکی مفقود به‌جای آغاز درخواست شبکه،
دردسترس‌نبودن خود را حفظ می‌کند.

## ثبت گزارش

| متغیر                         | کاربرد                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | سطح گزارش را هم برای فایل و هم برای کنسول بازنویسی می‌کند (برای مثال `debug`، `trace`). بر `logging.level` و `logging.consoleLevel` در پیکربندی اولویت دارد. مقادیر نامعتبر با یک هشدار نادیده گرفته می‌شوند. |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | بدون فعال‌کردن گزارش‌های اشکال‌زدایی سراسری، داده‌های تشخیصی هدفمند زمان‌بندی درخواست/پاسخ مدل را در سطح `info` منتشر می‌کند.                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | داده‌های تشخیصی بار مدل: `summary`، `tools` یا `full-redacted`. مقدار `full-redacted` محدود و سانسورشده است، اما ممکن است متن پیام/پرامپت را شامل شود.                                               |
| `OPENCLAW_DEBUG_SSE`             | داده‌های تشخیصی جریانی: `events` برای زمان‌بندی اولین/پایان و `peek` برای گنجاندن پنج رویداد نخست SSE به‌صورت سانسورشده.                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | داده‌های تشخیصی سطح مدل در حالت کد، شامل پنهان‌سازی ابزار ارائه‌دهنده و اعمال مستقیم/کنترل فشرده.                                                                                  |

### `OPENCLAW_HOME`

وقتی تنظیم شود، `OPENCLAW_HOME` جایگزین پوشهٔ خانهٔ سیستم (`$HOME` / `os.homedir()`) برای پیش‌فرض‌های مسیر داخلی OpenClaw می‌شود. این موارد شامل پوشهٔ پیش‌فرض وضعیت، مسیر پیکربندی، پوشه‌های عامل، اعتبارنامه‌ها، فضای کاری راه‌اندازی اولیهٔ نصب‌کننده و پرداخت پیش‌فرض توسعهٔ مورداستفادهٔ `openclaw update --channel dev` است.

**اولویت:** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > مسیر خانهٔ جایگزین Termux‏ `PREFIX` در Android > `os.homedir()`

**نمونه** (LaunchDaemon در macOS):

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` را می‌توان روی مسیری دارای علامت مد نیز تنظیم کرد (برای مثال `~/svc`) که پیش از استفاده، با همان زنجیرهٔ جایگزین خانهٔ سیستم‌عامل بسط داده می‌شود.

متغیرهای صریح مسیر مانند `OPENCLAW_STATE_DIR`، `OPENCLAW_CONFIG_PATH` و `OPENCLAW_GIT_DIR` همچنان اولویت دارند. وظایف مربوط به حساب سیستم‌عامل، مانند شناسایی فایل آغاز به کار پوسته، راه‌اندازی مدیر بسته و بسط `~` میزبان، ممکن است همچنان از پوشهٔ خانهٔ واقعی سیستم استفاده کنند.

## کاربران nvm: شکست‌های TLS در web_fetch

اگر Node.js از طریق **nvm** نصب شده باشد (نه مدیر بستهٔ سیستم)، `fetch()` داخلی از
مخزن CA همراه nvm استفاده می‌کند که ممکن است CAهای ریشهٔ جدید را نداشته باشد (ISRG Root X1/X2 برای Let's Encrypt،
DigiCert Global Root G2 و غیره). این موضوع باعث می‌شود `web_fetch` در بیشتر وب‌سایت‌های HTTPS با خطای `"fetch failed"` مواجه شود.

در Linux، OpenClaw به‌طور خودکار nvm را شناسایی می‌کند و راه‌حل را در محیط واقعی آغاز به کار اعمال می‌کند:

- `openclaw gateway install` مقدار `NODE_EXTRA_CA_CERTS` را در محیط سرویس systemd می‌نویسد
- نقطهٔ ورود CLI‏ `openclaw` پیش از آغاز Node، خود را با تنظیم `NODE_EXTRA_CA_CERTS` دوباره اجرا می‌کند

**راه‌حل دستی (برای نسخه‌های قدیمی‌تر یا اجرای مستقیم `node ...`):**

پیش از آغاز OpenClaw متغیر را صادر کنید:

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

برای این متغیر تنها به نوشتن در `~/.openclaw/.env` اتکا نکنید؛ Node مقدار
`NODE_EXTRA_CA_CERTS` را هنگام آغاز فرایند می‌خواند.

## متغیرهای محیطی قدیمی

OpenClaw فقط متغیرهای محیطی `OPENCLAW_*` را می‌خواند. پیشوندهای قدیمی
`CLAWDBOT_*` و `MOLTBOT_*` از نسخه‌های پیشین، بدون هشدار
نادیده گرفته می‌شوند.

اگر هنگام آغاز، هرکدام همچنان در فرایند Gateway تنظیم شده باشند، OpenClaw یک
هشدار منسوخ‌شدن Node‏ (`OPENCLAW_LEGACY_ENV_VARS`) منتشر می‌کند که
پیشوندهای شناسایی‌شده و تعداد کل را فهرست می‌کند. نام هر مقدار را با جایگزینی
پیشوند قدیمی با `OPENCLAW_` تغییر دهید (برای مثال `CLAWDBOT_GATEWAY_TOKEN` به
`OPENCLAW_GATEWAY_TOKEN`)؛ نام‌های قدیمی هیچ اثری ندارند.

## مرتبط

- [پیکربندی Gateway](/fa/gateway/configuration)
- [پرسش‌های متداول: متغیرهای محیطی و بارگذاری .env](/fa/help/faq#env-vars-and-env-loading)
- [نمای کلی مدل‌ها](/fa/concepts/models)
