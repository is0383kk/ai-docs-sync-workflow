---
read_when:
    - می‌خواهید OpenClaw را با یک سرور محلی vLLM اجرا کنید
    - می‌خواهید نقاط پایانی سازگار با OpenAI در مسیر `/v1` را با مدل‌های خودتان داشته باشید
summary: اجرای OpenClaw با vLLM (سرور محلی سازگار با OpenAI)
title: vLLM
x-i18n:
    generated_at: "2026-07-27T14:33:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM مدل‌های متن‌باز (و برخی مدل‌های سفارشی) را از طریق یک API HTTP **سازگار با OpenAI** ارائه می‌کند. OpenClaw با استفاده از API ‏`openai-completions` متصل می‌شود و در صورت اعلام موافقت از طریق `VLLM_API_KEY`، می‌تواند مدل‌ها را به‌طور **خودکار شناسایی** کند.

| ویژگی            | مقدار                                      |
| ---------------- | ------------------------------------------ |
| شناسه ارائه‌دهنده | `vllm`                                     |
| API              | `openai-completions` (سازگار با OpenAI)   |
| احراز هویت       | متغیر محیطی `VLLM_API_KEY`        |
| URL پایه پیش‌فرض | `http://127.0.0.1:8000/v1`                 |
| استفاده از استریم | پشتیبانی می‌شود (`stream_options.include_usage`) |

## شروع به کار

<Steps>
  <Step title="راه‌اندازی vLLM با یک سرور سازگار با OpenAI">
    URL پایه باید نقطه‌های پایانی `/v1` را ارائه کند (`/v1/models`، `/v1/chat/completions`). ‏vLLM معمولاً در نشانی زیر اجرا می‌شود:

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="تنظیم متغیر محیطی کلید API">
    اگر سرور احراز هویت را اجباری نمی‌کند، هر مقدار غیرخالی کار می‌کند:

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="انتخاب مدل">
    آن را با یکی از شناسه‌های مدل vLLM خود جایگزین کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="بررسی در دسترس بودن مدل">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
برای راه‌اندازی غیرتعاملی (CI، اسکریپت‌نویسی)، URL پایه، کلید و مدل را مستقیماً وارد کنید:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## شناسایی مدل (ارائه‌دهنده ضمنی)

وقتی `VLLM_API_KEY` تنظیم شده باشد (یا یک پروفایل احراز هویت وجود داشته باشد) و `models.providers.vllm` تعریف **نشده** باشد، OpenClaw از `GET http://127.0.0.1:8000/v1/models` پرس‌وجو می‌کند و شناسه‌های بازگشتی را به ورودی‌های مدل تبدیل می‌کند.

<Note>
اگر `models.providers.vllm` را صریحاً تنظیم کنید، OpenClaw فقط از مدل‌های اعلام‌شده شما استفاده می‌کند. برای اینکه OpenClaw علاوه بر این، از نقطه پایانی `/models` ارائه‌دهنده پیکربندی‌شده پرس‌وجو کند و همه مدل‌های vLLM اعلام‌شده را دربر بگیرد، `"vllm/*": {}` را به `agents.defaults.models` اضافه کنید.
</Note>

## پیکربندی صریح

هنگامی‌که vLLM روی میزبان یا درگاه دیگری اجرا می‌شود، می‌خواهید `contextWindow`/`maxTokens` را ثابت کنید، سرور به یک کلید API واقعی نیاز دارد، یا به یک نقطه پایانی قابل‌اعتماد loopback، ‏LAN یا Tailscale متصل می‌شوید، آن را صریحاً پیکربندی کنید:

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // اختیاری: افزایش مهلت زمانی درخواست برای مدل‌های محلی کند
        models: [
          {
            id: "your-model-id",
            name: "مدل محلی vLLM",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

برای پویا نگه‌داشتن ارائه‌دهنده بدون فهرست‌کردن همه مدل‌ها، یک نویسه عام به کاتالوگ مدل‌های قابل‌مشاهده اضافه کنید:

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="رفتار به‌سبک پراکسی">
    با vLLM به‌عنوان یک بک‌اند `/v1` سازگار با OpenAI و به‌سبک پراکسی رفتار می‌شود، نه یک نقطه پایانی بومی OpenAI:

    | رفتار                                   | اعمال می‌شود؟                    |
    | --------------------------------------- | -------------------------------- |
    | شکل‌دهی بومی درخواست OpenAI             | خیر                              |
    | `service_tier`                          | ارسال نمی‌شود                    |
    | Responses `store`                       | ارسال نمی‌شود                    |
    | راهنمایی‌های کش پرامپت                   | ارسال نمی‌شود                    |
    | شکل‌دهی محموله سازگاری استدلال OpenAI   | اعمال نمی‌شود                    |
    | سرآیندهای پنهان انتساب OpenClaw         | به URLهای پایه سفارشی تزریق نمی‌شوند |

  </Accordion>

  <Accordion title="کنترل‌های تفکر Qwen">
    برای مدل‌های Qwen، هنگامی‌که سرور آرگومان‌های کلیدی قالب چت Qwen را انتظار دارد، `compat.thinkingFormat: "qwen-chat-template"` را در ردیف مدل تنظیم کنید. این مدل‌ها یک پروفایل دودویی `/think` ‏(`off`، `on`) ارائه می‌کنند، زیرا تفکر قالب چت Qwen یک پرچم روشن/خاموش است، نه یک نردبان شدت به‌سبک OpenAI.

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw، ‏`/think off` را به مورد زیر نگاشت می‌کند:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    سطوح تفکر غیر از `off`، ‏`enable_thinking: true` را ارسال می‌کنند. اگر نقطه پایانی شما در عوض پرچم‌های سطح‌بالای به‌سبک DashScope را انتظار دارد، از `compat.thinkingFormat: "qwen"` استفاده کنید تا `enable_thinking` در ریشه درخواست ارسال شود.

  </Accordion>

  <Accordion title="کنترل‌های تفکر Nemotron 3">
    برای مدل‌های `vllm/nemotron-3-*` که تفکر در آن‌ها خاموش است، Plugin همراه مورد زیر را ارسال می‌کند:

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    برای سفارشی‌سازی این مقادیر، `chat_template_kwargs` را زیر پارامترهای مدل تنظیم کنید. اگر `params.extra_body.chat_template_kwargs` را نیز تنظیم کنید، آن مقدار اولویت دارد، زیرا `extra_body` آخرین بازنویسی بدنه درخواست است.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="فراخوانی ابزارهای Qwen به‌شکل متن نمایش داده می‌شوند">
    ابتدا تأیید کنید که vLLM با تجزیه‌گر فراخوانی ابزار و قالب چت مناسب مدل راه‌اندازی شده است. مستندات vLLM، ‏`hermes` را برای مدل‌های Qwen2.5 و `qwen3_xml` را برای مدل‌های Qwen3-Coder ذکر می‌کند.

    نشانه‌ها: Skills/ابزارها هرگز اجرا نمی‌شوند، دستیار JSON/XML خامی مانند `{"name":"read","arguments":...}` را چاپ می‌کند، یا وقتی OpenClaw ‏`tool_choice: "auto"` را ارسال می‌کند، vLLM یک آرایه خالی `tool_calls` برمی‌گرداند.

    برخی ترکیب‌های Qwen/vLLM فقط زمانی فراخوانی ابزار ساخت‌یافته برمی‌گردانند که درخواست از `tool_choice: "required"` استفاده کند. با `params.extra_body` آن را برای هر مدل اجباری کنید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    شناسه مدل را با شناسه دقیق موجود در `openclaw models list --provider vllm` جایگزین کنید، یا همان بازنویسی را از CLI اعمال کنید:

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    این یک راهکار موقت انتخابی است: هر نوبتی را که ابزار دارد مجبور به انجام یک فراخوانی ابزار می‌کند، بنابراین فقط برای یک ورودی مدل اختصاصی که این رفتار در آن پذیرفتنی است از آن استفاده کنید. آن را به‌عنوان پیش‌فرض سراسری برای همه مدل‌های vLLM تنظیم نکنید و با پراکسی‌ای که متن دلخواه دستیار را به فراخوانی ابزار اجرایی تبدیل می‌کند جفت نکنید.

  </Accordion>

  <Accordion title="URL پایه سفارشی">
    اگر سرور vLLM روی میزبان یا درگاه غیراستاندارد اجرا می‌شود، `baseUrl` را در پیکربندی صریح ارائه‌دهنده تنظیم کنید:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "مدل راه‌دور vLLM",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="پاسخ نخست کند است یا مهلت زمانی سرور راه‌دور تمام می‌شود">
    برای مدل‌های محلی بزرگ، میزبان‌های LAN راه‌دور یا پیوندهای tailnet، مهلت زمانی درخواست را در محدوده ارائه‌دهنده تنظیم کنید:

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "مدل محلی vLLM" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` فقط بر درخواست‌های HTTP مدل vLLM اعمال می‌شود: برقراری اتصال، سرآیندهای پاسخ، استریم بدنه و لغو کلی واکشی محافظت‌شده. همچنین سقف زمان‌سنج نظارتی بی‌کاری/استریم LLM را برای این ارائه‌دهنده از مقدار پیش‌فرض ضمنی حدود ~120s بالاتر می‌برد. این روش را به افزایش `agents.defaults.timeoutSeconds` ترجیح دهید؛ مورد دوم کل اجرای عامل را کنترل می‌کند.

  </Accordion>

  <Accordion title="سرور در دسترس نیست">
    بررسی کنید که سرور vLLM در حال اجرا و قابل‌دسترسی باشد:

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    اگر خطای اتصال مشاهده می‌کنید، میزبان، درگاه و راه‌اندازی vLLM در حالت سرور سازگار با OpenAI را بررسی کنید. OpenClaw برای درخواست‌های محافظت‌شده مدل در نقطه‌های پایانی loopback، ‏LAN و Tailscale دقیقاً به مبدأ پیکربندی‌شده `models.providers.vllm.baseUrl` اعتماد می‌کند. مبدأهای فراداده/link-local بدون اعلام موافقت صریح همچنان مسدود می‌مانند. فقط زمانی `models.providers.vllm.request.allowPrivateNetwork: true` را تنظیم کنید که درخواست‌های vLLM باید به مبدأ خصوصی دیگری برسند، یا برای انصراف از اعتماد به مبدأ دقیق، `false` را تنظیم کنید.

  </Accordion>

  <Accordion title="خطاهای احراز هویت در درخواست‌ها">
    اگر درخواست‌ها با خطاهای احراز هویت ناموفق می‌شوند، یک `VLLM_API_KEY` واقعی و منطبق با پیکربندی سرور تنظیم کنید، یا ارائه‌دهنده را صریحاً زیر `models.providers.vllm` پیکربندی کنید.

    <Tip>
    اگر سرور vLLM احراز هویت را اجباری نمی‌کند، هر مقدار غیرخالی برای `VLLM_API_KEY` به‌عنوان سیگنال اعلام موافقت برای OpenClaw کار می‌کند.
    </Tip>

  </Accordion>

  <Accordion title="هیچ مدلی شناسایی نشد">
    شناسایی خودکار نیازمند تنظیم `VLLM_API_KEY` است. اگر `models.providers.vllm` را تعریف کرده باشید، OpenClaw فقط از مدل‌های اعلام‌شده شما استفاده می‌کند، مگر اینکه `agents.defaults.models` شامل `"vllm/*": {}` باشد.
  </Accordion>

  <Accordion title="ابزارها به‌شکل متن خام نمایش داده می‌شوند">
    اگر یک مدل Qwen به‌جای اجرای یک Skill، نحو ابزار JSON/XML را چاپ می‌کند:

    - ‏vLLM را با تجزیه‌گر/قالب صحیح آن مدل راه‌اندازی کنید.
    - شناسه دقیق مدل را با `openclaw models list --provider vllm` تأیید کنید.
    - فقط اگر `tool_choice: "auto"` همچنان فراخوانی‌های ابزار خالی یا صرفاً متنی برمی‌گرداند، یک بازنویسی اختصاصی `params.extra_body.tool_choice: "required"` برای هر مدل اضافه کنید.

  </Accordion>
</AccordionGroup>

<Warning>
راهنمای بیشتر: [عیب‌یابی](/fa/help/troubleshooting) و [پرسش‌های متداول](/fa/help/faq).
</Warning>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="OpenAI" href="/fa/providers/openai" icon="bolt">
    ارائه‌دهنده بومی OpenAI و رفتار مسیر سازگار با OpenAI.
  </Card>
  <Card title="OAuth و احراز هویت" href="/fa/gateway/authentication" icon="key">
    جزئیات احراز هویت و قواعد استفاده مجدد از اعتبارنامه‌ها.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و روش رفع آن‌ها.
  </Card>
</CardGroup>
