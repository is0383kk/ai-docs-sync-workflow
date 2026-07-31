---
read_when:
    - می‌خواهید از Fireworks با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API مربوط به Fireworks یا شناسه مدل پیش‌فرض نیاز دارید
    - در حال اشکال‌زدایی رفتار thinking-off در Kimi روی Fireworks هستید
summary: راه‌اندازی Fireworks (احراز هویت + انتخاب مدل)
title: Fireworks
x-i18n:
    generated_at: "2026-07-27T15:50:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7720b23b69aa716d2e2903f5644bb74f81ca1c5e753f71d72d4d7a25c0747884
    source_path: providers/fireworks.md
    workflow: 16
---

[Fireworks](https://fireworks.ai) مدل‌های با وزن‌های باز و مسیریابی‌شده را از طریق یک API سازگار با OpenAI ارائه می‌کند. برای استفاده از دو مدل Kimi ازپیش‌فهرست‌شده و هر شناسه مدل یا مسیریاب Fireworks در زمان اجرا، Plugin رسمی ارائه‌دهنده Fireworks را نصب کنید.

| ویژگی        | مقدار                                                  |
| --------------- | ------------------------------------------------------ |
| شناسه ارائه‌دهنده     | `fireworks` (نام مستعار: `fireworks-ai`)                    |
| بسته         | `@openclaw/fireworks-provider`                         |
| متغیر محیطی احراز هویت    | `FIREWORKS_API_KEY`                                    |
| پرچم راه‌اندازی اولیه | `--auth-choice fireworks-api-key`                      |
| پرچم مستقیم CLI | `--fireworks-api-key <key>`                            |
| API             | سازگار با OpenAI (`openai-completions`)               |
| نشانی پایه        | `https://api.fireworks.ai/inference/v1`                |
| مدل پیش‌فرض   | `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` |
| نام مستعار پیش‌فرض   | `Kimi K2.6 Turbo`                                      |

## شروع به کار

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install @openclaw/fireworks-provider
    ```
  </Step>
  <Step title="تنظیم کلید API مربوط به Fireworks">
    <CodeGroup>

```bash راه‌اندازی اولیه
openclaw onboard --auth-choice fireworks-api-key
```

```bash پرچم مستقیم
openclaw onboard --non-interactive \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY"
```

```bash فقط متغیر محیطی
export FIREWORKS_API_KEY=fw-...
```

    </CodeGroup>

    راه‌اندازی اولیه، کلید را برای ارائه‌دهنده `fireworks` در پروفایل‌های احراز هویت شما ذخیره می‌کند و مسیریاب Kimi K2.6 Turbo با **Fire Pass** را به‌عنوان مدل پیش‌فرض تنظیم می‌کند.

  </Step>
  <Step title="اطمینان از دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider fireworks
    ```

    فهرست باید شامل `Kimi K2.6` و `Kimi K2.6 Turbo (Fire Pass)` باشد. اگر `FIREWORKS_API_KEY` حل‌نشده باشد، `openclaw models status --json` اعتبارنامه مفقود را در بخش `auth.unusableProfiles` گزارش می‌کند.

  </Step>
</Steps>

## راه‌اندازی غیرتعاملی

برای نصب‌های اسکریپتی یا CI، همه‌چیز را در خط فرمان وارد کنید:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice fireworks-api-key \
  --fireworks-api-key "$FIREWORKS_API_KEY" \
  --skip-health \
  --accept-risk
```

## کاتالوگ داخلی

| ارجاع مدل                                              | نام                        | ورودی        | زمینه | حداکثر خروجی | تفکر             |
| ------------------------------------------------------ | --------------------------- | ------------ | ------- | ---------- | -------------------- |
| `fireworks/accounts/fireworks/models/kimi-k2p6`        | Kimi K2.6                   | متن + تصویر | 262,144 | 262,144    | اجباراً خاموش           |
| `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo` | Kimi K2.6 Turbo (Fire Pass) | متن + تصویر | 256,000 | 256,000    | اجباراً خاموش (پیش‌فرض) |

<Note>
  OpenClaw همه مدل‌های Kimi در Fireworks را روی `thinking: off` ثابت می‌کند، زیرا Kimi در Fireworks ممکن است زنجیره فکر را در پاسخ قابل‌مشاهده افشا کند، مگر آنکه درخواست صراحتاً تفکر را غیرفعال کند. مسیریابی مستقیم همان مدل از طریق [Moonshot](/fa/providers/moonshot)، خروجی استدلال Kimi را حفظ می‌کند. برای جابه‌جایی میان ارائه‌دهندگان، به [حالت‌های تفکر](/fa/tools/thinking) مراجعه کنید.
</Note>

## شناسه‌های سفارشی مدل Fireworks

OpenClaw در زمان اجرا هر شناسه مدل یا مسیریاب Fireworks را می‌پذیرد. از شناسه دقیق نمایش‌داده‌شده در Fireworks استفاده کنید و پیشوند `fireworks/` را به آن بیفزایید. تفکیک پویا، الگوی Fire Pass را شبیه‌سازی می‌کند (ورودی متن + تصویر، API سازگار با OpenAI و هزینه پیش‌فرض صفر) و وقتی شناسه با الگوی Kimi مطابقت داشته باشد، تفکر را به‌طور خودکار غیرفعال می‌کند. شناسه‌های پویای GLM فقط‌متنی علامت‌گذاری می‌شوند، مگر آنکه یک ورودی مدل سفارشی با ورودی تصویر پیکربندی کنید.

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "fireworks/accounts/fireworks/models/<your-model-id>",
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="نحوه کار پیشوندگذاری شناسه مدل">
    هر ارجاع مدل Fireworks در OpenClaw با `fireworks/` آغاز می‌شود و پس از آن، شناسه دقیق یا مسیر مسیریاب از پلتفرم Fireworks می‌آید. برای مثال:

    - مدل مسیریاب: `fireworks/accounts/fireworks/routers/kimi-k2p6-turbo`
    - مدل مستقیم: `fireworks/accounts/fireworks/models/<model-name>`

    OpenClaw هنگام ساخت درخواست API، پیشوند `fireworks/` را حذف می‌کند و مسیر باقی‌مانده را به‌عنوان فیلد سازگار با OpenAI یعنی `model` به نقطه پایانی Fireworks می‌فرستد.

  </Accordion>

  <Accordion title="چرا تفکر برای Kimi اجباراً خاموش است">
    Fireworks مدل Kimi را بدون کانال استدلال جداگانه ارائه می‌کند؛ بنابراین زنجیره فکر ممکن است در جریان قابل‌مشاهده `content` ظاهر شود. OpenClaw در هر درخواست Kimi به Fireworks، مقدار `thinking: { type: "disabled" }` را ارسال می‌کند و `reasoning`، `reasoning_effort` و `reasoningEffort` را از محموله حذف می‌کند (`extensions/fireworks/stream.ts`). خط‌مشی ارائه‌دهنده (`extensions/fireworks/thinking-policy.ts`) برای شناسه‌های مدل Kimi فقط سطح تفکر `off` را اعلام می‌کند؛ بنابراین تغییرات دستی `/think` و سطوح خط‌مشی ارائه‌دهنده با قرارداد زمان اجرا هم‌راستا می‌مانند.

    برای استفاده سرتاسری از استدلال Kimi، [ارائه‌دهنده Moonshot](/fa/providers/moonshot) را پیکربندی و همان مدل را از طریق آن مسیریابی کنید.

  </Accordion>

  <Accordion title="دردسترس‌بودن محیط برای دیمن">
    اگر Gateway به‌صورت یک سرویس مدیریت‌شده (launchd، systemd، Docker) اجرا می‌شود، کلید Fireworks باید برای آن فرایند قابل‌مشاهده باشد، نه فقط برای پوسته تعاملی شما.

    <Warning>
      کلیدی که فقط در یک پوسته تعاملی صادر شده است، برای دیمن launchd یا systemd مفید نخواهد بود، مگر آنکه آن محیط نیز در آنجا وارد شود. برای آنکه کلید از فرایند Gateway قابل‌خواندن باشد، آن را در `~/.openclaw/.env` یا از طریق `env.shellEnv` تنظیم کنید.
    </Warning>

    OpenClaw هنگام بارگذاری پیکربندی، `~/.openclaw/.env` را بارگذاری می‌کند؛ بنابراین کلیدهای ذخیره‌شده در آن، در هر پلتفرمی به سرویس‌های مدیریت‌شده Gateway می‌رسند. پس از تعویض کلید، Gateway را راه‌اندازی مجدد کنید (یا `openclaw doctor --fix` را دوباره اجرا کنید).

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="حالت‌های تفکر" href="/fa/tools/thinking" icon="brain">
    سطوح `/think`، خط‌مشی‌های ارائه‌دهنده و مسیریابی مدل‌های دارای قابلیت استدلال.
  </Card>
  <Card title="Moonshot" href="/fa/providers/moonshot" icon="moon">
    اجرای Kimi با خروجی تفکر بومی از طریق API اختصاصی Moonshot.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    عیب‌یابی عمومی و پرسش‌های متداول.
  </Card>
</CardGroup>
