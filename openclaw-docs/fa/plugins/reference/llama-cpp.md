---
read_when:
    - در حال نصب، پیکربندی یا ممیزی Plugin llama-cpp هستید
summary: استنتاج متن و تعبیه‌سازی‌های محلی GGUF از طریق node-llama-cpp.
title: Plugin ‏Llama Cpp
x-i18n:
    generated_at: "2026-07-27T14:28:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2756d4b3e00bbe37b4dedec1d54d28bfe6662e8105504317a402293254ce0240
    source_path: plugins/reference/llama-cpp.md
    workflow: 16
---

# Plugin ‏Llama Cpp

استنتاج متن و تعبیه‌سازی‌های محلی GGUF از طریق node-llama-cpp.

## توزیع

- بسته: `@openclaw/llama-cpp-provider`
- مسیر نصب: npm؛ ClawHub

## سطح

ارائه‌دهندگان: `llama-cpp`؛ قراردادها: `embeddingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## مدل متنی پیش‌فرض

در طول راه‌اندازی تعاملی، OpenClaw مدل Gemma 4 E4B IT Q4_K_M را به‌صورت دانلود همراهی با حجم تقریبی 5.0 GB پیشنهاد می‌کند. این پیشنهاد به حداقل 16 GiB حافظهٔ RAM کل نیاز دارد. مدل‌هایی که از قبل در حافظهٔ نهان قرار دارند، همچنان در دستگاه‌های کوچک‌تر شناسایی می‌شوند.

برای استفاده از مدلی دیگر، `params.modelPath` را روی هر GGUF سفارشی تنظیم کنید. مدل‌های سفارشی مشمول الزام RAM دانلود همراه نیستند. در دستگاه‌هایی با حافظهٔ کمتر از مقدار موردنیاز، می‌توانید مدلی کوچک‌تر را نیز از طریق Ollama یا LM Studio اجرا کنید، یا یک ارائه‌دهندهٔ ابری را برگزینید.

<!-- openclaw-plugin-reference:manual-end -->

## مستندات مرتبط

- [llama-cpp](/fa/plugins/llama-cpp)
