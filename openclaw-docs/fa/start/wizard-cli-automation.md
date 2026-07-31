---
read_when:
    - در حال خودکارسازی راه‌اندازی اولیه در اسکریپت‌ها یا CI هستید
    - برای ارائه‌دهندگان مشخص به نمونه‌های غیرتعاملی نیاز دارید
sidebarTitle: CLI automation
summary: راه‌اندازی اسکریپتی اولیه و عامل برای CLI ‏OpenClaw
title: خودکارسازی CLI
x-i18n:
    generated_at: "2026-07-27T15:51:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2a9fd8530379927995641f8033651ff12ada98068f106672e6655a17b8265735
    source_path: start/wizard-cli-automation.md
    workflow: 16
---

برای اسکریپت‌نویسی راه‌اندازی از `openclaw onboard --non-interactive` استفاده کنید. این گزینه به `--accept-risk` نیاز دارد: راه‌اندازی غیرتعاملی می‌تواند اعتبارنامه‌ها و پیکربندی دیمون را بدون درخواست تأیید بنویسد؛ بنابراین این پرچم به‌منزله پذیرش صریح ریسک است.

<Note>
`--json` به‌معنای فعال‌شدن حالت غیرتعاملی نیست. برای اسکریپت‌ها، `--non-interactive --accept-risk` را صریحاً وارد کنید.
</Note>

## نمونه پایه غیرتعاملی

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode plaintext \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-bootstrap \
  --skip-skills
```

برای دریافت خلاصه‌ای قابل‌خواندن توسط ماشین، `--json` را اضافه کنید.

- `--gateway-port` به‌طور پیش‌فرض `18789` است؛ فقط برای بازنویسی آن، این گزینه را وارد کنید.
- `--skip-bootstrap` از ایجاد فایل‌های پیش‌فرض فضای کاری جلوگیری می‌کند و برای خودکارسازی‌هایی است که فضای کاری خود را از پیش آماده می‌کنند.
- `--secret-input-mode ref` به‌جای کلید متن ساده، یک ارجاع مبتنی بر متغیر محیطی (`{ source: "env", provider: "default", id: "<ENV_VAR>" }`) را در نمایه احراز هویت ذخیره می‌کند. در حالت غیرتعاملی `ref`، متغیر محیطی ارائه‌دهنده باید از قبل در محیط فرایند تنظیم شده باشد: واردکردن پرچم کلید درون‌خطی بدون متغیر محیطی متناظر، بلافاصله با خطا متوقف می‌شود.

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref
```

## نمونه‌های ویژه ارائه‌دهندگان

<AccordionGroup>
  <Accordion title="نمونه کلید API ‏Anthropic">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice apiKey \
      --anthropic-api-key "$ANTHROPIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Cloudflare AI Gateway">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice cloudflare-ai-gateway-api-key \
      --cloudflare-ai-gateway-account-id "your-account-id" \
      --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
      --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Gemini">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice gemini-api-key \
      --gemini-api-key "$GEMINI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Mistral">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice mistral-api-key \
      --mistral-api-key "$MISTRAL_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Moonshot">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice moonshot-api-key \
      --moonshot-api-key "$MOONSHOT_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Ollama">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ollama \
      --custom-model-id "qwen3.5:27b" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه OpenCode">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice opencode-zen \
      --opencode-zen-api-key "$OPENCODE_API_KEY" \
      --gateway-bind loopback
    ```
    برای کاتالوگ Go، آن را با `--auth-choice opencode-go --opencode-go-api-key "$OPENCODE_API_KEY"` جایگزین کنید.
  </Accordion>
  <Accordion title="نمونه Synthetic">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice synthetic-api-key \
      --synthetic-api-key "$SYNTHETIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Vercel AI Gateway">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ai-gateway-api-key \
      --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه Z.AI">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice zai-api-key \
      --zai-api-key "$ZAI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="نمونه ارائه‌دهنده سفارشی">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --custom-api-key "$CUSTOM_API_KEY" \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

    `--custom-api-key` اختیاری است؛ برخی نقاط پایانی به احراز هویت نیاز ندارند. اگر حذف شود، فرایند راه‌اندازی `CUSTOM_API_KEY` را در متغیرهای محیطی بررسی می‌کند. `--custom-provider-id` اختیاری است و در صورت حذف، به‌طور خودکار از URL پایه استخراج می‌شود. مقدار پیش‌فرض `--custom-compatibility` برابر با `openai` است (مقادیر دیگر: `openai-responses`، `anthropic`).

    OpenClaw پشتیبانی از ورودی تصویر را از الگوهای شناخته‌شده شناسه مدل‌های بینایی استنباط می‌کند (پسوندهای `gpt-4o`، `claude-3/4`، `gemini`، `-vl`/`vision` و موارد مشابه). برای فعال‌سازی اجباری آن برای یک مدل بینایی ناشناخته، `--custom-image-input` را اضافه کنید؛ یا برای اجبار به حالت فقط‌متنی، `--custom-text-input` را اضافه کنید.

    گونه حالت ارجاع، با ذخیره `apiKey` به‌صورت `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }`:

    ```bash
    export CUSTOM_API_KEY="your-key"
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --secret-input-mode ref \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

  </Accordion>
</AccordionGroup>

احراز هویت با توکن راه‌اندازی Anthropic همچنان پشتیبانی می‌شود، اما وقتی ورود محلی Claude CLI موجود باشد، OpenClaw استفاده مجدد از Claude CLI را ترجیح می‌دهد. برای محیط عملیاتی، کلید API ‏Anthropic را ترجیح دهید.

## افزودن عامل دیگر

`openclaw agents add <name>` عاملی جداگانه با فضای کاری، نشست‌ها و نمایه‌های احراز هویت مخصوص خود ایجاد می‌کند. اجرای آن بدون `--workspace` (و بدون هیچ پرچم دیگری) ویزارد تعاملی را راه‌اندازی می‌کند؛ واردکردن هر یک از `--workspace`، `--model`، `--agent-dir`، `--bind` یا `--non-interactive` آن را به‌صورت غیرتعاملی اجرا می‌کند و سپس به `--workspace` نیاز دارد.

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

کلیدهای پیکربندی که می‌نویسد (ورودی `agents.entries.*` برای شناسه عامل جدید):

- `name`
- `workspace`
- `agentDir`
- `model` (فقط هنگام واردکردن `--model`)

نکته‌ها:

- فضای کاری پیش‌فرض (وقتی `--workspace` در ویزارد تعاملی حذف شده باشد): `~/.openclaw/workspace-<agentId>`.
- `--bind <channel[:accountId]>` تکرارپذیر است؛ برای هدایت پیام‌های ورودی به عامل جدید، اتصال‌ها را اضافه کنید (ویزارد نیز می‌تواند این کار را به‌صورت تعاملی انجام دهد).
- نام عامل به یک شناسه معتبر عامل عادی‌سازی می‌شود؛ `main` رزروشده است.

## مستندات مرتبط

- مرکز راه‌اندازی: [راه‌اندازی (CLI)](/fa/start/wizard)
- مرجع کامل: [مرجع راه‌اندازی CLI](/fa/start/wizard-cli-reference)
- مرجع فرمان: [`openclaw onboard`](/fa/cli/onboard)
