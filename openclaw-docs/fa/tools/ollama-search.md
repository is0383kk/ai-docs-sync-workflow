---
read_when:
    - می‌خواهید از Ollama برای web_search استفاده کنید
    - یک ارائه‌دهندهٔ web_search بدون نیاز به کلید می‌خواهید
    - می‌خواهید از Ollama Web Search میزبانی‌شده با OLLAMA_API_KEY استفاده کنید
    - به راهنمای راه‌اندازی جست‌وجوی وب Ollama نیاز دارید
summary: جست‌وجوی وب Ollama از طریق میزبان محلی Ollama یا API میزبانی‌شده Ollama
title: جست‌وجوی وب Ollama
x-i18n:
    generated_at: "2026-07-27T15:55:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: edbbd887841339ab4c0c62ab7682a22fe99434a788957a91989fce6942187e9a
    source_path: tools/ollama-search.md
    workflow: 16
---

OpenClaw از **Ollama Web Search** به‌عنوان ارائه‌دهندهٔ همراه `web_search` پشتیبانی می‌کند
و عنوان‌ها، نشانی‌های URL و قطعه‌متن‌ها را از API جست‌وجوی وب Ollama برمی‌گرداند.

Ollama محلی/خودمیزبان به‌طور پیش‌فرض به کلید API نیاز ندارد؛ به یک میزبان
Ollama قابل‌دسترسی به‌همراه `ollama signin` نیاز دارد. جست‌وجوی میزبانی‌شدهٔ مستقیم (بدون Ollama محلی) به
`baseUrl: "https://ollama.com"` و یک `OLLAMA_API_KEY` واقعی نیاز دارد.

## راه‌اندازی

<Steps>
  <Step title="راه‌اندازی Ollama">
    مطمئن شوید Ollama نصب و در حال اجرا است.
  </Step>
  <Step title="ورود به حساب">
    ```bash
    ollama signin
    ```
  </Step>
  <Step title="انتخاب Ollama Web Search">
    ```bash
    openclaw configure --section web
    ```

    **Ollama Web Search** را به‌عنوان ارائه‌دهنده انتخاب کنید.

  </Step>
</Steps>

اگر از قبل برای مدل‌ها از Ollama استفاده می‌کنید، Ollama Web Search از همان
میزبان پیکربندی‌شده استفاده می‌کند.

<Note>
  OpenClaw هرگز Ollama Web Search را به‌طور خودکار به‌جای یک ارائه‌دهندهٔ
  دارای اعتبارنامه با اولویت بالاتر انتخاب نمی‌کند؛ باید آن را صراحتاً با
  `tools.web.search.provider: "ollama"` انتخاب کنید.
</Note>

## پیکربندی

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

بازنویسی اختیاری میزبان، فقط در محدودهٔ جست‌وجوی وب:

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

یا از میزبانی که از قبل برای ارائه‌دهندهٔ مدل Ollama پیکربندی شده است استفاده کنید:

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

`models.providers.ollama.baseUrl` کلید معیار است؛ ارائه‌دهندهٔ جست‌وجوی وب
برای سازگاری با نمونه‌های پیکربندی به سبک OpenAI SDK، `baseURL` را نیز در آنجا
می‌پذیرد. اگر هیچ‌چیز تنظیم نشده باشد، مقدار پیش‌فرض OpenClaw
`http://127.0.0.1:11434` است.

Ollama Web Search میزبانی‌شدهٔ مستقیم (بدون Ollama محلی):

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

## احراز هویت و مسیریابی درخواست

- هیچ فیلد کلید API ویژهٔ جست‌وجوی وب وجود ندارد؛ ارائه‌دهنده هنگام محافظت‌شدن
  میزبان پیکربندی‌شده با احراز هویت، از `models.providers.ollama.apiKey` (یا احراز هویت منطبق ارائه‌دهنده که از متغیر محیطی تأمین می‌شود)
  دوباره استفاده می‌کند.
- ترتیب تفکیک میزبان: `plugins.entries.ollama.config.webSearch.baseUrl` ←
  `models.providers.ollama.baseUrl` (یا `baseURL`) ← `http://127.0.0.1:11434`.
- اگر میزبان تفکیک‌شده `https://ollama.com` باشد، OpenClaw
  مستقیماً `https://ollama.com/api/web_search` را با کلید API به‌عنوان
  احراز هویت bearer فراخوانی می‌کند.
- در غیر این صورت، OpenClaw ابتدا نقطهٔ پایانی پروکسی محلی
  `/api/experimental/web_search` را فراخوانی می‌کند (که درخواست را امضا کرده و به Ollama
  Cloud ارسال می‌کند)، سپس به `/api/web_search` روی همان میزبان بازمی‌گردد. اگر هر دو ناموفق باشند
  و `OLLAMA_API_KEY` تنظیم شده باشد، یک‌بار دیگر با آن کلید
  روی `https://ollama.com/api/web_search` تلاش می‌کند — بدون ارسال آن به
  میزبان محلی.
- اگر Ollama قابل‌دسترسی نباشد یا ورود به حساب انجام نشده باشد، OpenClaw هنگام راه‌اندازی هشدار می‌دهد، اما
  مانع انتخاب ارائه‌دهنده نمی‌شود.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [Ollama](/fa/providers/ollama) -- راه‌اندازی مدل Ollama و حالت‌های ابری/محلی
