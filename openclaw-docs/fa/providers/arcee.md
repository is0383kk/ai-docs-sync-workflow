---
read_when:
    - می‌خواهید از Arcee AI با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API یا گزینه احراز هویت CLI نیاز دارید
summary: راه‌اندازی Arcee AI (احراز هویت + انتخاب مدل)
title: Arcee AI
x-i18n:
    generated_at: "2026-07-27T16:03:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a4c2fc7b8d86dd0d2a300dfc48951657cbcfcd9250016f52c1804777b2966e11
    source_path: providers/arcee.md
    workflow: 16
---

[Arcee AI](https://arcee.ai) خانواده مدل‌های ترکیب خبرگان Trinity را از طریق یک API سازگار با OpenAI ارائه می‌دهد. همه مدل‌های Trinity تحت مجوز Apache 2.0 هستند. Arcee یک Plugin رسمی OpenClaw است و همراه هسته ارائه نمی‌شود؛ بنابراین پیش از راه‌اندازی اولیه باید نصب شود.

از طریق پلتفرم Arcee یا [OpenRouter](/fa/providers/openrouter) مستقیماً به مدل‌های Arcee دسترسی پیدا کنید.

| ویژگی | مقدار                                                                                 |
| -------- | ------------------------------------------------------------------------------------- |
| ارائه‌دهنده | `arcee`                                                                               |
| احراز هویت     | `ARCEEAI_API_KEY` (مستقیم) یا `OPENROUTER_API_KEY` (از طریق OpenRouter)                   |
| API      | سازگار با OpenAI                                                                     |
| نشانی پایه | `https://api.arcee.ai/api/v1` (مستقیم) یا `https://openrouter.ai/api/v1` (OpenRouter) |

## نصب Plugin

```bash
openclaw plugins install @openclaw/arcee-provider
openclaw gateway restart
```

## شروع به کار

<Tabs>
  <Tab title="مستقیم (پلتفرم Arcee)">
    <Steps>
      <Step title="دریافت کلید API">
        یک کلید API در [Arcee AI](https://chat.arcee.ai/) ایجاد کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        ```bash
        openclaw onboard --auth-choice arceeai-api-key
        ```
      </Step>
      <Step title="تنظیم مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```
      </Step>
    </Steps>
  </Tab>

  <Tab title="از طریق OpenRouter">
    <Steps>
      <Step title="دریافت کلید API">
        یک کلید API در [OpenRouter](https://openrouter.ai/keys) ایجاد کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        ```bash
        openclaw onboard --auth-choice arceeai-openrouter
        ```
      </Step>
      <Step title="تنظیم مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "arcee/trinity-large-thinking" },
            },
          },
        }
        ```

        همان ارجاع‌های مدل برای هر دو پیکربندی مستقیم و OpenRouter کار می‌کنند.
      </Step>
    </Steps>

  </Tab>
</Tabs>

## راه‌اندازی غیرتعاملی

<Tabs>
  <Tab title="مستقیم (پلتفرم Arcee)">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-api-key \
      --arceeai-api-key "$ARCEEAI_API_KEY"
    ```
  </Tab>

  <Tab title="از طریق OpenRouter">
    ```bash
    openclaw onboard --non-interactive \
      --mode local \
      --auth-choice arceeai-openrouter \
      --openrouter-api-key "$OPENROUTER_API_KEY"
    ```
  </Tab>
</Tabs>

## فهرست مدل‌های مستقیم Arcee

| ارجاع مدل                      | نام                   | ورودی | زمینه | حداکثر خروجی | هزینه (ورودی/خروجی به‌ازای هر 1M) | ابزارها | یادداشت‌ها                                     |
| ------------------------------ | ---------------------- | ----- | ------- | ---------- | -------------------- | ----- | ----------------------------------------- |
| `arcee/trinity-large-thinking` | Trinity Large Thinking | متن  | 256K    | 80K        | $0.25 / $0.90        | خیر    | مدل پیش‌فرض؛ تفکر توسعه‌یافته          |
| `arcee/trinity-large-preview`  | Trinity Large Preview  | متن  | 128K    | 16K        | $0.25 / $1.00        | بله   | همه‌منظوره؛ 400B پارامتر، 13B فعال  |
| `arcee/trinity-mini`           | Trinity Mini 26B       | متن  | 128K    | 80K        | $0.045 / $0.15       | بله   | سریع و مقرون‌به‌صرفه؛ فراخوانی تابع |

<Tip>
پیش‌تنظیم راه‌اندازی اولیه، `arcee/trinity-large-thinking` را به‌عنوان مدل پیش‌فرض تنظیم می‌کند.
</Tip>

## فهرست مدل‌های OpenRouter

راه‌اندازی اولیه OpenRouter، مدل‌های `arcee/trinity-large-preview` و `arcee/trinity-large-thinking` را در دسترس قرار می‌دهد. OpenClaw این ارجاع‌های مدل دارای شناسه ارائه‌دهنده را در پیکربندی نگه می‌دارد و شناسه‌های زمان اجرای متعارف `arcee-ai/*` متعلق به OpenRouter را ارسال می‌کند. Trinity Mini دیگر از طریق OpenRouter ارائه نمی‌شود؛ برای این مدل از API مستقیم Arcee استفاده کنید.

## قابلیت‌های پشتیبانی‌شده

| قابلیت                                       | پشتیبانی                                    |
| --------------------------------------------- | -------------------------------------------- |
| پخش جریانی                                     | بله                                          |
| استفاده از ابزار / فراخوانی تابع                   | بله (Trinity Mini، Trinity Large Preview)    |
| خروجی ساختاریافته (حالت JSON و شِمای JSON) | بله                                          |
| تفکر توسعه‌یافته                             | بله (Trinity Large Thinking؛ ابزارها غیرفعال‌اند) |

<AccordionGroup>
  <Accordion title="نکته محیطی">
    اگر Gateway به‌صورت دیمن (launchd/systemd) اجرا می‌شود، مطمئن شوید `ARCEEAI_API_KEY`
    (یا `OPENROUTER_API_KEY`) برای آن فرایند در دسترس است؛ برای نمونه در
    `~/.openclaw/.env` یا از طریق `env.shellEnv`.
  </Accordion>

  <Accordion title="مسیریابی OpenRouter">
    OpenRouter از همان ارجاع مدل `arcee/trinity-large-thinking` در OpenClaw استفاده می‌کند.
    OpenClaw آن را با شناسه متعارف زمان اجرای `arcee-ai/trinity-large-thinking`
    در OpenRouter مسیریابی می‌کند. برای جزئیات پیکربندی مختص OpenRouter به
    [مستندات ارائه‌دهنده OpenRouter](/fa/providers/openrouter) مراجعه کنید.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="OpenRouter" href="/fa/providers/openrouter" icon="shuffle">
    با یک کلید API به مدل‌های Arcee و بسیاری مدل‌های دیگر دسترسی پیدا کنید.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
</CardGroup>
