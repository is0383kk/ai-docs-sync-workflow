---
read_when:
    - می‌خواهید از Cloudflare AI Gateway با OpenClaw استفاده کنید
    - به شناسه حساب، شناسه Gateway یا متغیر محیطی کلید API نیاز دارید
summary: راه‌اندازی Cloudflare AI Gateway (احراز هویت + انتخاب مدل)
title: Gateway هوش مصنوعی Cloudflare
x-i18n:
    generated_at: "2026-07-27T17:02:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 02c7785616e7aee645bb3fc41ef6a3585e1f2f9d886fab1a06231e497effd045
    source_path: providers/cloudflare-ai-gateway.md
    workflow: 16
---

[Cloudflare AI Gateway](https://developers.cloudflare.com/ai-gateway/) در جلوی APIهای ارائه‌دهندگان قرار می‌گیرد و قابلیت‌های تحلیل، کش‌کردن و کنترل را اضافه می‌کند. برای Anthropic، ‏OpenClaw از طریق نقطه پایانی Gateway شما از Anthropic Messages API استفاده می‌کند.

| ویژگی         | مقدار                                                                                    |
| ------------- | ---------------------------------------------------------------------------------------- |
| ارائه‌دهنده   | `cloudflare-ai-gateway`                                                                  |
| Plugin        | بسته رسمی خارجی (`@openclaw/cloudflare-ai-gateway-provider`)                   |
| نشانی پایه    | `https://gateway.ai.cloudflare.com/v1/<account_id>/<gateway_id>/anthropic`               |
| مدل پیش‌فرض   | `cloudflare-ai-gateway/claude-sonnet-4-6`                                                |
| کلید API      | `CLOUDFLARE_AI_GATEWAY_API_KEY` (کلید API ارائه‌دهنده شما برای درخواست‌هایی که از طریق Gateway ارسال می‌شوند) |

<Note>
برای مدل‌های Anthropic که از طریق Cloudflare AI Gateway مسیریابی می‌شوند، از **کلید API ‏Anthropic** خود به‌عنوان کلید ارائه‌دهنده استفاده کنید.
</Note>

وقتی قابلیت تفکر برای مدل‌های Anthropic Messages فعال باشد، ‏OpenClaw پیش از ارسال بار داده از طریق Cloudflare AI Gateway، نوبت‌های پیش‌پرشده پایانی دستیار را حذف می‌کند.
Anthropic پیش‌پرکردن پاسخ همراه با تفکر توسعه‌یافته را رد می‌کند، درحالی‌که
پیش‌پرکردن عادی بدون تفکر همچنان در دسترس است.

## نصب Plugin

Plugin رسمی را نصب کنید، سپس Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/cloudflare-ai-gateway-provider
openclaw gateway restart
```

## شروع به کار

<Steps>
  <Step title="تنظیم کلید API ارائه‌دهنده و جزئیات Gateway">
    فرایند راه‌اندازی اولیه را اجرا و گزینه احراز هویت Cloudflare AI Gateway را انتخاب کنید:

    ```bash
    openclaw onboard --auth-choice cloudflare-ai-gateway-api-key
    ```

    در این مرحله شناسه حساب، شناسه Gateway و کلید API شما درخواست می‌شود.

  </Step>
  <Step title="تنظیم مدل پیش‌فرض">
    مدل را به پیکربندی OpenClaw خود اضافه کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "cloudflare-ai-gateway/claude-sonnet-4-6" },
        },
      },
    }
    ```

  </Step>
  <Step title="تأیید دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider cloudflare-ai-gateway
    ```
  </Step>
</Steps>

## نمونه غیرتعاملی

برای راه‌اندازی‌های اسکریپتی یا CI، همه مقادیر را در خط فرمان وارد کنید:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice cloudflare-ai-gateway-api-key \
  --cloudflare-ai-gateway-account-id "your-account-id" \
  --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
  --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY"
```

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="Gatewayهای احرازهویت‌شده">
    اگر احراز هویت Gateway را در Cloudflare فعال کرده‌اید، هدر `cf-aig-authorization` را اضافه کنید. این هدر **علاوه بر** کلید API ارائه‌دهنده شما است.

    ```json5
    {
      models: {
        providers: {
          "cloudflare-ai-gateway": {
            headers: {
              "cf-aig-authorization": "Bearer <cloudflare-ai-gateway-token>",
            },
          },
        },
      },
    }
    ```

    <Tip>
    هدر `cf-aig-authorization` احراز هویت را با خود Cloudflare Gateway انجام می‌دهد، درحالی‌که کلید API ارائه‌دهنده (برای مثال، کلید Anthropic شما) احراز هویت را با ارائه‌دهنده بالادستی انجام می‌دهد.
    </Tip>

  </Accordion>

  <Accordion title="نکته محیطی">
    اگر Gateway به‌صورت دیمن (launchd/systemd) اجرا می‌شود، مطمئن شوید `CLOUDFLARE_AI_GATEWAY_API_KEY` برای آن فرایند در دسترس است.

    <Warning>
    کلیدی که فقط در یک پوسته تعاملی صادر شده باشد، برای دیمن launchd/systemd مفید نخواهد بود، مگر اینکه آن محیط نیز به آنجا وارد شود. کلید را در `~/.openclaw/.env` یا از طریق `env.shellEnv` تنظیم کنید تا فرایند Gateway بتواند آن را بخواند.
    </Warning>

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    عیب‌یابی عمومی و پرسش‌های متداول.
  </Card>
</CardGroup>
