---
read_when:
    - می‌خواهید OpenClaw را از طریق یک پروکسی LiteLLM مسیریابی کنید
    - به ردیابی هزینه، ثبت گزارش‌ها یا مسیریابی مدل از طریق LiteLLM نیاز دارید
summary: اجرای OpenClaw از طریق LiteLLM Proxy برای دسترسی یکپارچه به مدل‌ها و پیگیری هزینه‌ها
title: LiteLLM
x-i18n:
    generated_at: "2026-07-27T17:03:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22451f0eefcf991a602409701fc752f97600a67752c67304137c7f17f3dd1a16
    source_path: providers/litellm.md
    workflow: 16
---

[LiteLLM](https://litellm.ai) یک Gateway متن‌باز برای LLM است که API یکپارچه‌ای برای بیش از 100 ارائه‌دهندهٔ مدل فراهم می‌کند.
OpenClaw را از طریق LiteLLM مسیریابی کنید تا بدون تغییر پیکربندی OpenClaw، ردیابی متمرکز هزینه، ثبت رویدادها، کلیدهای مجازی با
محدودیت هزینه و جایگزینی خودکار بک‌اند را در اختیار داشته باشید.

## شروع سریع

<Tabs>
  <Tab title="راه‌اندازی اولیه (توصیه‌شده)">
    ```bash
    openclaw onboard --auth-choice litellm-api-key
    ```

    برای راه‌اندازی غیرتعاملی در برابر یک پراکسی راه دور، URL پراکسی را به‌صراحت وارد کنید:

    ```bash
    openclaw onboard --non-interactive --accept-risk --auth-choice litellm-api-key \
      --litellm-api-key "$LITELLM_API_KEY" --custom-base-url "https://litellm.example/v1"
    ```

  </Tab>

  <Tab title="راه‌اندازی دستی">
    <Steps>
      <Step title="راه‌اندازی پراکسی LiteLLM">
        ```bash
        pip install 'litellm[proxy]'
        litellm --model claude-opus-4-6
        ```
      </Step>
      <Step title="اتصال OpenClaw به LiteLLM">
        ```bash
        export LITELLM_API_KEY="your-litellm-key"
        openclaw
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## پیکربندی

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "claude-opus-4-6",
            name: "Claude Opus 4.6",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 200000,
            maxTokens: 64000,
          },
          {
            id: "gpt-4o",
            name: "GPT-4o",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "litellm/claude-opus-4-6" },
    },
  },
}
```

مدل پیش‌فرضی که راه‌اندازی اولیه می‌نویسد، `litellm/claude-opus-4-6` است.

## تولید تصویر

LiteLLM می‌تواند ابزار `image_generate` را از طریق مسیرهای سازگار با OpenAI یعنی `/images/generations` و
`/images/edits` پشتیبانی کند. مدل پیش‌فرض تصویر `gpt-image-2` است؛ مدل دیگری را در
`agents.defaults.mediaModels.image` پیکربندی کنید:

```json5
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
      },
    },
  },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "litellm/gpt-image-2",
        timeoutMs: 180_000,
      },
    },
  },
}
```

URLهای حلقهٔ بازگشتی LiteLLM ‏(`http://localhost:4000`، `127.0.0.1`، `::1`، `host.docker.internal`) بدون
نادیده‌گیری سراسری شبکهٔ خصوصی کار می‌کنند. برای پراکسی میزبانی‌شده در LAN،
`models.providers.litellm.request.allowPrivateNetwork: true` را تنظیم کنید، زیرا کلید API به آن میزبان ارسال می‌شود.

## پیشرفته

<AccordionGroup>
  <Accordion title="کلیدهای مجازی">
    برای OpenClaw یک کلید اختصاصی با محدودیت هزینه ایجاد کنید:

    ```bash
    curl -X POST "http://localhost:4000/key/generate" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY" \
      -H "Content-Type: application/json" \
      -d '{
        "key_alias": "openclaw",
        "max_budget": 50.00,
        "budget_duration": "monthly"
      }'
    ```

    از کلید ایجادشده به‌عنوان `LITELLM_API_KEY` استفاده کنید.

  </Accordion>

  <Accordion title="مسیریابی مدل">
    LiteLLM می‌تواند درخواست‌های مدل را به بک‌اندهای مختلف مسیریابی کند. آن را در `config.yaml` مربوط به LiteLLM پیکربندی کنید:

    ```yaml
    model_list:
      - model_name: claude-opus-4-6
        litellm_params:
          model: claude-opus-4-6
          api_key: os.environ/ANTHROPIC_API_KEY

      - model_name: gpt-4o
        litellm_params:
          model: gpt-4o
          api_key: os.environ/OPENAI_API_KEY
    ```

    OpenClaw همچنان `claude-opus-4-6` را درخواست می‌کند؛ LiteLLM مسیریابی را انجام می‌دهد.

  </Accordion>

  <Accordion title="مشاهدهٔ میزان استفاده">
    ```bash
    # اطلاعات کلید
    curl "http://localhost:4000/key/info" \
      -H "Authorization: Bearer sk-litellm-key"

    # گزارش‌های هزینه
    curl "http://localhost:4000/spend/logs" \
      -H "Authorization: Bearer $LITELLM_MASTER_KEY"
    ```

  </Accordion>

  <Accordion title="نکاتی دربارهٔ رفتار پراکسی">
    - LiteLLM به‌طور پیش‌فرض روی `http://localhost:4000` اجرا می‌شود.
    - OpenClaw از طریق نقطهٔ پایانی سازگار با OpenAI و به‌سبک پراکسی LiteLLM یعنی `/v1` متصل می‌شود.
    - شکل‌دهی درخواست مختص OpenAI بومی، از طریق URL پایهٔ پیکربندی‌شدهٔ LiteLLM اعمال نمی‌شود:
      بدون `service_tier`، بدون Responses ‏`store`، بدون راهنمایی‌های کش پرامپت و بدون شکل‌دهی بار دادهٔ
      تلاش استدلال OpenAI.
    - هدرهای پنهان انتساب OpenClaw ‏(`originator`، `version`، `User-Agent`) فقط به
      نقاط پایانی تأییدشدهٔ بومی OpenAI ارسال می‌شوند؛ بنابراین به URL پایهٔ سفارشی LiteLLM تزریق نمی‌شوند.
  </Accordion>
</AccordionGroup>

<Note>
برای پیکربندی عمومی ارائه‌دهنده و رفتار جایگزینی خودکار، به [ارائه‌دهندگان مدل](/fa/concepts/model-providers) مراجعه کنید.
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="مستندات LiteLLM" href="https://docs.litellm.ai" icon="book">
    مستندات رسمی LiteLLM و مرجع API.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    نمای کلی همهٔ ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی خودکار.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی.
  </Card>
  <Card title="مدل‌ها" href="/fa/concepts/models" icon="brain">
    نحوهٔ انتخاب و پیکربندی مدل‌ها.
  </Card>
</CardGroup>
