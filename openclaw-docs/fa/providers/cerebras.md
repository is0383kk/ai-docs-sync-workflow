---
read_when:
    - می‌خواهید از Cerebras با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API سرویس Cerebras یا گزینه احراز هویت CLI نیاز دارید
summary: راه‌اندازی Cerebras (احراز هویت + انتخاب مدل)
title: Cerebras
x-i18n:
    generated_at: "2026-07-27T15:48:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 716eef83155ef80d9aa61bd55ed83e3e38ad22720ae055bce7eb9c2cbfb6cf41
    source_path: providers/cerebras.md
    workflow: 16
---

[Cerebras](https://www.cerebras.ai) استنتاج پرسرعتِ سازگار با OpenAI را روی سخت‌افزار سفارشی استنتاج ارائه می‌دهد. این Plugin با یک کاتالوگ ثابت شامل دو مدل عرضه می‌شود (بدون کشف زنده).

| ویژگی           | مقدار                                                     |
| --------------- | --------------------------------------------------------- |
| شناسه ارائه‌دهنده | `cerebras`                                                |
| Plugin          | بسته خارجی رسمی (`@openclaw/cerebras-provider`) |
| متغیر محیطی احراز هویت | `CEREBRAS_API_KEY`                                        |
| پرچم راه‌اندازی اولیه | `--auth-choice cerebras-api-key`                          |
| پرچم مستقیم CLI | `--cerebras-api-key <key>`                                |
| API             | سازگار با OpenAI (`openai-completions`)                  |
| نشانی پایه      | `https://api.cerebras.ai/v1`                              |
| مدل پیش‌فرض     | `cerebras/zai-glm-4.7`                                    |

## نصب Plugin

```bash
openclaw plugins install @openclaw/cerebras-provider
openclaw gateway restart
```

## شروع به کار

<Steps>
  <Step title="دریافت کلید API">
    یک کلید API در [کنسول ابری Cerebras](https://cloud.cerebras.ai) ایجاد کنید.
  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    <CodeGroup>

```bash راه‌اندازی اولیه
openclaw onboard --auth-choice cerebras-api-key
```

```bash پرچم مستقیم
openclaw onboard --non-interactive \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

```bash فقط متغیر محیطی
export CEREBRAS_API_KEY=csk-...
```

    </CodeGroup>

  </Step>
  <Step title="بررسی دردسترس‌بودن مدل‌ها">
    ```bash
    openclaw models list --provider cerebras
    ```

    هر دو مدل ثابت را فهرست می‌کند. اگر `CEREBRAS_API_KEY` حل‌نشده باشد، `openclaw models status --json` اعتبارنامه مفقود را زیر `auth.unusableProfiles` گزارش می‌کند.

  </Step>
</Steps>

## راه‌اندازی غیرتعاملی

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cerebras-api-key \
  --cerebras-api-key "$CEREBRAS_API_KEY"
```

## کاتالوگ داخلی

هر دو مدل دارای پنجره زمینه 128k و حداکثر 8,192 توکن خروجی هستند.

| ارجاع مدل               | نام          | استدلال | یادداشت‌ها                                  |
| ----------------------- | ------------ | --------- | -------------------------------------- |
| `cerebras/zai-glm-4.7`  | Z.ai GLM 4.7 | بله       | مدل پیش‌فرض؛ مدل استدلالی پیش‌نمایش |
| `cerebras/gpt-oss-120b` | GPT OSS 120B | بله       | مدل استدلالی عملیاتی             |

## پیکربندی دستی

بیشتر راه‌اندازی‌ها فقط به کلید API نیاز دارند. برای بازنویسی فراداده مدل یا اجرا در `mode: "merge"` در برابر کاتالوگ ثابت، از پیکربندی صریح `models.providers.cerebras` استفاده کنید:

```json5
{
  env: { CEREBRAS_API_KEY: "csk-..." },
  agents: {
    defaults: {
      model: { primary: "cerebras/zai-glm-4.7" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      cerebras: {
        baseUrl: "https://api.cerebras.ai/v1",
        apiKey: "${CEREBRAS_API_KEY}",
        api: "openai-completions",
        models: [
          { id: "zai-glm-4.7", name: "Z.ai GLM 4.7" },
          { id: "gpt-oss-120b", name: "GPT OSS 120B" },
        ],
      },
    },
  },
}
```

<Note>
اگر Gateway به‌صورت یک دیمون (launchd، systemd، Docker) اجرا می‌شود، مطمئن شوید `CEREBRAS_API_KEY` برای آن فرایند در دسترس است — برای مثال در `~/.openclaw/.env` یا از طریق `env.shellEnv`. کلیدی که فقط در یک پوسته تعاملی صادر شده باشد، به یک سرویس مدیریت‌شده کمکی نمی‌کند، مگر اینکه محیط به‌طور جداگانه وارد شود.
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="حالت‌های تفکر" href="/fa/tools/thinking" icon="brain">
    سطوح تلاش استدلالی برای دو مدل Cerebras دارای قابلیت استدلال.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    پیش‌فرض‌های عامل و پیکربندی مدل.
  </Card>
  <Card title="پرسش‌های متداول مدل‌ها" href="/fa/help/faq-models" icon="circle-question">
    نمایه‌های احراز هویت، تعویض مدل‌ها و رفع خطاهای «بدون نمایه».
  </Card>
</CardGroup>
