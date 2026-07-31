---
read_when:
    - می‌خواهید از انتقال‌دهنده‌های مدل OpenClaw در برنامه‌ای دیگر دوباره استفاده کنید
    - در حال تغییر packages/ai یا پورت‌های میزبان انتقال AI هستید
    - در حال بررسی این هستید که انتشار OpenClaw علاوه بر بستهٔ ریشه، چه چیزهایی را در npm منتشر می‌کند
summary: 'بسته npm ‏@openclaw/ai: انتقال‌دهنده‌های مدلِ قابل‌استفادهٔ مجدد، محیط‌های اجرای ایزوله و درگاه‌های سیاست میزبان'
title: بستهٔ @openclaw/ai
x-i18n:
    generated_at: "2026-07-27T16:09:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 610057caae0a9bbf9f74074cda75fc40c0b9aa9d3441f8263151f08f1a3f35a8
    source_path: reference/openclaw-ai.md
    workflow: 16
---

`@openclaw/ai` شکل کتابخانه‌ای قابل‌انتشارِ لایه اجرای مدل OpenClaw است:
قراردادهای پیام/ابزار/جریان مستقل از ارائه‌دهنده، اعتبارسنجی، عیب‌یابی،
جریان‌های رویداد، یک رجیستری زمان‌اجرای ایزوله، و آداپتورهای بارگذاری تنبل برای هشت
خانواده API داخلی (Anthropic Messages، OpenAI Completions، OpenAI
Responses، Azure OpenAI Responses، ChatGPT/Codex Responses، Google Generative
AI، Google Vertex، Mistral Conversations).

این کتابخانه در هر انتشار، در کنار بسته ریشه `openclaw` و با همان
نسخه تثبیت‌شده منتشر می‌شود و `npm-shrinkwrap.json` مختص خود را دارد تا درخت
وابستگی‌های انتقالی آن هنگام نصب قفل شود. نصب `openclaw`، نسخه منطبق
`@openclaw/ai` را به‌طور خودکار نصب می‌کند؛ مصرف‌کنندگان کتابخانه می‌توانند
بدون هیچ کد برنامه OpenClaw، مستقیماً به آن وابسته باشند.

## شروع سریع

```js
import { createLlmRuntime } from "@openclaw/ai";
import { registerBuiltInApiProviders } from "@openclaw/ai/providers";

const runtime = createLlmRuntime();
registerBuiltInApiProviders(runtime.registry);

const stream = runtime.streamSimple(model, { messages }, { apiKey });
for await (const event of stream) {
  if (event.type === "text_delta") process.stdout.write(event.delta);
}
const result = await stream.result();
```

نسخه‌ای قابل‌اجرا در `examples/ai-chat` مخزن قرار دارد.

## قرارداد طراحی

- **به‌طور پیش‌فرض محدود به نمونه است.** واردکردن بسته هیچ‌چیز را
  به‌صورت سراسری ثبت نمی‌کند. `createApiRegistry()` / `createLlmRuntime()` نمونه‌های
  ایزوله برمی‌گردانند؛ `registerBuiltInApiProviders(registry)` یک رجیستری را برای استفاده از
  انتقال‌های داخلی فعال می‌کند. ماژول‌های SDK ارائه‌دهنده در نخستین استفاده به‌صورت تنبل بارگذاری می‌شوند.
- **سیاست میزبان تزریق می‌شود، نه اینکه همراه بسته باشد.** محافظت از fetch درخواست
  (برای مثال، سیاست SSRF)، پنهان‌سازی اسرار در متن بازپخش نتیجه ابزار، پیش‌فرض‌های
  ابزار سخت‌گیرانه OpenAI و ثبت عیب‌یابی، پورت‌های `AiTransportHost` هستند
  که با `configureAiTransportHost` پیکربندی می‌شوند. پیش‌فرض‌های کتابخانه غیرفعال‌اند؛
  OpenClaw پیاده‌سازی‌های واقعی خود را در نمای جریان نصب می‌کند.
- **یک هویت واحد برای جریان رویداد.** `@openclaw/ai/event-stream` سازنده متعارف
  `EventStream` است که میان هسته OpenClaw، agent-core و مصرف‌کنندگان
  خارجی مشترک است.
- **زیرمسیرهای `internal/*` جزو API نیستند.** آن‌ها برای خود برنامه
  OpenClaw وجود دارند و هیچ تضمین semver ندارند.
- شناسه‌های ارائه‌دهنده، اطلاعات اعتبارسنجی، کاتالوگ‌های مدل، تلاش‌های مجدد و جایگزینی در صورت خرابی،
  همچنان دغدغه‌های برنامه هستند. OpenClaw این موارد را پیرامون این بسته لایه‌بندی می‌کند؛
  مصرف‌کننده کتابخانه یک شیء `Model` و گزینه‌ها را مستقیماً ارائه می‌دهد.

## خروجی‌های زیرمسیر

| زیرمسیر          | محتویات                                                                       |
| ---------------- | ------------------------------------------------------------------------------ |
| `.`              | قراردادها، `createApiRegistry`، `createLlmRuntime`، `configureAiTransportHost` |
| `./providers`    | `registerBuiltInApiProviders`، `resetApiProviders`                             |
| `./types`        | نوع‌های مدل/پیام/ابزار/جریان                                                |
| `./validation`   | اعتبارسنجی آرگومان‌های ابزار                                                       |
| `./diagnostics`  | قراردادهای عیب‌یابی                                                          |
| `./event-stream` | پیاده‌سازی مشترک `EventStream`                                            |
| `./internal/*`   | داخلی OpenClaw، بدون تضمین semver                                         |
