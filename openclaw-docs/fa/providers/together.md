---
read_when:
    - می‌خواهید از Together AI با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API یا گزینه احراز هویت CLI نیاز دارید
summary: راه‌اندازی Together AI (احراز هویت + انتخاب مدل)
title: Together AI
x-i18n:
    generated_at: "2026-07-27T16:07:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9b08cae93c1ea7df46e1d2fbe78692f73bb3e56809122f70a56eec8b3dc5d8a4
    source_path: providers/together.md
    workflow: 16
---

[Together AI](https://together.ai) از طریق یک API یکپارچه، دسترسی به مدل‌های متن‌باز پیشرو از جمله Llama، DeepSeek، Kimi و مدل‌های دیگر را فراهم می‌کند.
OpenClaw آن را به‌عنوان ارائه‌دهندهٔ `together` عرضه می‌کند.

| ویژگی | مقدار                         |
| -------- | ----------------------------- |
| ارائه‌دهنده | `together`                    |
| احراز هویت     | `TOGETHER_API_KEY`            |
| API      | سازگار با OpenAI             |
| نشانی پایه | `https://api.together.xyz/v1` |

## شروع به کار

<Steps>
  <Step title="دریافت کلید API">
    یک کلید API در
    [api.together.ai/settings/api-keys](https://api.together.ai/settings/api-keys) ایجاد کنید.
  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    ```bash
    openclaw onboard --auth-choice together-api-key
    ```
  </Step>
  <Step title="تنظیم مدل پیش‌فرض">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "together/meta-llama/Llama-3.3-70B-Instruct-Turbo",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

### نمونهٔ غیرتعاملی

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice together-api-key \
  --together-api-key "$TOGETHER_API_KEY"
```

<Note>
راه‌اندازی اولیه، `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` را به‌عنوان
مدل پیش‌فرض تنظیم می‌کند.
</Note>

## کاتالوگ داخلی

هزینه بر حسب دلار آمریکا به‌ازای هر یک میلیون توکن است.

| ارجاع مدل                                          | نام                         | ورودی       | زمینه | حداکثر خروجی | هزینه (ورودی/خروجی) | توضیحات               |
| -------------------------------------------------- | ---------------------------- | ----------- | ------- | ---------- | ------------- | ------------------- |
| `together/meta-llama/Llama-3.3-70B-Instruct-Turbo` | Llama 3.3 70B Instruct Turbo | متن        | 131,072 | 8,192      | 0.88 / 0.88   | مدل پیش‌فرض       |
| `together/moonshotai/Kimi-K2.6`                    | Kimi K2.6 FP4                | متن، تصویر | 262,144 | 32,768     | 1.20 / 4.50   | مدل استدلالی     |
| `together/deepseek-ai/DeepSeek-V4-Pro`             | DeepSeek V4 Pro              | متن        | 512,000 | 8,192      | 2.10 / 4.40   | مدل استدلالی     |
| `together/Qwen/Qwen2.5-7B-Instruct-Turbo`          | Qwen2.5 7B Instruct Turbo    | متن        | 32,768  | 8,192      | 0.30 / 0.30   | سریع، بدون استدلال |
| `together/zai-org/GLM-5.1`                         | GLM 5.1 FP4                  | متن        | 202,752 | 8,192      | 1.40 / 4.40   | مدل استدلالی     |

## تولید ویدئو

Plugin داخلی `together` همچنین تولید ویدئو را از طریق
ابزار مشترک `video_generate` ثبت می‌کند.

| ویژگی             | مقدار                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------- |
| مدل پیش‌فرض ویدئو  | `Wan-AI/Wan2.2-T2V-A14B`                                                                  |
| مدل‌های دیگر         | `Wan-AI/Wan2.2-I2V-A14B`، `minimax/hailuo-02`، `kwaivgI/kling-2.1-master`                 |
| حالت‌ها                | متن‌به‌ویدئو؛ تصویر‌به‌ویدئو فقط با `Wan-AI/Wan2.2-I2V-A14B` (یک تصویر مرجع) |
| مدت‌زمان             | 1 تا 10 ثانیه                                                                              |
| پارامترهای پشتیبانی‌شده | `size` (به‌صورت `<width>x<height>` تجزیه می‌شود)؛ `aspectRatio`/`resolution` خوانده نمی‌شوند            |

برای استفاده از Together به‌عنوان ارائه‌دهندهٔ پیش‌فرض ویدئو:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "together/Wan-AI/Wan2.2-T2V-A14B",
      },
    },
  },
}
```

<Tip>
برای پارامترهای ابزار مشترک، انتخاب ارائه‌دهنده و رفتار انتقال هنگام خرابی، به [تولید ویدئو](/fa/tools/video-generation) مراجعه کنید.
</Tip>

<AccordionGroup>
  <Accordion title="نکتهٔ محیطی">
    اگر Gateway به‌صورت یک دیمن (launchd/systemd) اجرا می‌شود، مطمئن شوید
    `TOGETHER_API_KEY` برای آن فرایند در دسترس است (برای نمونه، در
    `~/.openclaw/.env` یا از طریق `env.shellEnv`).

    <Warning>
    کلیدهایی که فقط در پوستهٔ تعاملی تنظیم شده‌اند، برای فرایندهای Gateway
    تحت مدیریت دیمن قابل مشاهده نیستند. برای دسترس‌پذیری پایدار، از پیکربندی
    `~/.openclaw/.env` یا `env.shellEnv` استفاده کنید.
    </Warning>

  </Accordion>

  <Accordion title="عیب‌یابی">
    - بررسی کنید کلیدتان کار می‌کند: `openclaw models list --provider together`
    - اگر مدل‌ها نمایش داده نمی‌شوند، تأیید کنید که کلید API در محیط صحیح
      برای فرایند Gateway تنظیم شده است.
    - ارجاع‌های مدل از قالب `together/<model-id>` استفاده می‌کنند.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    قواعد ارائه‌دهنده، ارجاع‌های مدل و رفتار انتقال هنگام خرابی.
  </Card>
  <Card title="تولید ویدئو" href="/fa/tools/video-generation" icon="video">
    پارامترهای ابزار مشترک تولید ویدئو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    طرح‌وارهٔ کامل پیکربندی، شامل تنظیمات ارائه‌دهنده.
  </Card>
  <Card title="Together AI" href="https://together.ai" icon="arrow-up-right-from-square">
    داشبورد، مستندات API و قیمت‌گذاری Together AI.
  </Card>
</CardGroup>
