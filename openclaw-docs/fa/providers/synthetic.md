---
read_when:
    - می‌خواهید از Synthetic به‌عنوان ارائه‌دهنده مدل استفاده کنید
    - به یک کلید API یا پیکربندی URL پایه برای Synthetic نیاز دارید
summary: از API سازگار با Anthropic متعلق به Synthetic در OpenClaw استفاده کنید
title: Synthetic
x-i18n:
    generated_at: "2026-07-27T15:52:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c3f6cc89a7b837f57555d176ce78e62a39095d4ef0765c96b6b7b93ffebd7388
    source_path: providers/synthetic.md
    workflow: 16
---

[Synthetic](https://synthetic.new) نقطه‌های پایانی سازگار با Anthropic را ارائه می‌دهد.
OpenClaw آن را به‌عنوان ارائه‌دهندهٔ `synthetic` همراه خود دارد و از API پیام‌های Anthropic
استفاده می‌کند.

| ویژگی | مقدار                                 |
| -------- | ------------------------------------- |
| ارائه‌دهنده | `synthetic`                           |
| احراز هویت     | `SYNTHETIC_API_KEY`                   |
| API      | پیام‌های Anthropic                    |
| نشانی پایه | `https://api.synthetic.new/anthropic` |

## شروع به کار

<Steps>
  <Step title="دریافت کلید API">
    یک `SYNTHETIC_API_KEY` از حساب Synthetic خود دریافت کنید، یا اجازه دهید فرایند راه‌اندازی اولیه
    آن را از شما درخواست کند.
  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    ```bash
    openclaw onboard --auth-choice synthetic-api-key
    ```
  </Step>
  <Step title="بررسی مدل پیش‌فرض">
    راه‌اندازی اولیه، مدل پیش‌فرض را روی مقدار زیر تنظیم می‌کند:
    ```text
    synthetic/hf:MiniMaxAI/MiniMax-M3
    ```
  </Step>
</Steps>

<Warning>
کلاینت Anthropic در OpenClaw به‌طور خودکار `/v1` را به نشانی پایه می‌افزاید؛ بنابراین از
`https://api.synthetic.new/anthropic` استفاده کنید (نه `/anthropic/v1`). اگر Synthetic
نشانی پایهٔ خود را تغییر داد، `models.providers.synthetic.baseUrl` را بازنویسی کنید.
</Warning>

## نمونهٔ پیکربندی

```json5
{
  env: { SYNTHETIC_API_KEY: "sk-..." },
  agents: {
    defaults: {
      model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
      models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
    },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [
          {
            id: "hf:MiniMaxAI/MiniMax-M3",
            name: "MiniMax M3",
            reasoning: true,
            input: ["text", "image"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 262144,
            maxTokens: 65536,
          },
        ],
      },
    },
  },
}
```

## کاتالوگ داخلی

همهٔ مدل‌های Synthetic از هزینهٔ `0` (ورودی/خروجی/کش) استفاده می‌کنند. برای اطلاع از دسترس‌پذیری سرویس، به
[فهرست مدل‌های فعلی](https://dev.synthetic.new/docs/api/models) Synthetic مراجعه کنید.

| شناسهٔ مدل                                            | پنجرهٔ زمینه | حداکثر توکن | استدلال | ورودی        |
| --------------------------------------------------- | -------------- | ---------- | --------- | ------------ |
| `hf:MiniMaxAI/MiniMax-M3`                           | 262,144        | 65,536     | بله       | متن + تصویر |
| `hf:moonshotai/Kimi-K2.7-Code`                      | 262,144        | 8,192      | بله       | متن + تصویر |
| `hf:nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4` | 262,144        | 8,192      | بله       | متن         |
| `hf:openai/gpt-oss-120b`                            | 131,072        | 8,192      | بله       | متن         |
| `hf:Qwen/Qwen3.6-27B`                               | 262,144        | 81,920     | بله       | متن + تصویر |
| `hf:zai-org/GLM-4.7-Flash`                          | 196,608        | 131,072    | بله       | متن         |
| `hf:zai-org/GLM-5.2`                                | 524,288        | 131,072    | بله       | متن         |

<Tip>
ارجاع‌های مدل از قالب `synthetic/<modelId>` استفاده می‌کنند. برای مشاهدهٔ همهٔ مدل‌های دردسترس در
حساب خود، از `openclaw models list --provider synthetic` استفاده کنید.
</Tip>

<AccordionGroup>
  <Accordion title="فهرست مجاز مدل‌ها">
    اگر فهرست مجاز مدل‌ها (`agents.defaults.modelPolicy.allow`) را فعال می‌کنید، همهٔ
    مدل‌های Synthetic را که قصد استفاده از آن‌ها را دارید اضافه کنید. مدل‌هایی که در فهرست مجاز نیستند
    از عامل پنهان می‌شوند.
  </Accordion>

  <Accordion title="بازنویسی نشانی پایه">
    اگر Synthetic نقطهٔ پایانی API خود را تغییر داد، نشانی پایه را بازنویسی کنید:

    ```json5
    {
      models: {
        providers: {
          synthetic: {
            baseUrl: "https://new-api.synthetic.new/anthropic",
          },
        },
      },
    }
    ```

    OpenClaw همچنان `/v1` را به‌طور خودکار اضافه می‌کند.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    قواعد ارائه‌دهنده، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    طرح‌وارهٔ کامل پیکربندی، شامل تنظیمات ارائه‌دهنده.
  </Card>
  <Card title="Synthetic" href="https://synthetic.new" icon="arrow-up-right-from-square">
    داشبورد و مستندات API مربوط به Synthetic.
  </Card>
</CardGroup>
