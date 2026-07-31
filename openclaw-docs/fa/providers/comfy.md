---
read_when:
    - می‌خواهید از گردش‌کارهای محلی ComfyUI با OpenClaw استفاده کنید
    - می‌خواهید از Comfy Cloud با گردش‌کارهای تصویر، ویدئو یا موسیقی استفاده کنید
    - به کلیدهای پیکربندی Plugin همراه comfy نیاز دارید
summary: راه‌اندازی تولید تصویر، ویدئو و موسیقی با گردش‌کار ComfyUI در OpenClaw
title: ComfyUI
x-i18n:
    generated_at: "2026-07-27T14:32:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74150d202a422de8e0f4b2b82d5d12bd42eb46991e8ef688832208e1a2ff7793
    source_path: providers/comfy.md
    workflow: 16
---

OpenClaw یک Plugin داخلی `comfy` برای اجرای ComfyUI مبتنی بر گردش‌کار ارائه می‌کند. این
Plugin کاملاً مبتنی بر گردش‌کار است: OpenClaw کنترل‌های عمومی `size`،
`aspectRatio`، `resolution`، `durationSeconds` یا کنترل‌های سبک TTS را روی
گراف شما نگاشت نمی‌کند.

| ویژگی        | جزئیات                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| ارائه‌دهنده  | `comfy`                                                                          |
| مدل          | `comfy/workflow`                                                                 |
| ابزارهای مشترک | `image_generate`، `video_generate`، `music_generate`                             |
| احراز هویت   | برای ComfyUI محلی هیچ‌کدام؛ برای Comfy Cloud، `COMFY_API_KEY` یا `COMFY_CLOUD_API_KEY` |
| API          | ComfyUI `/prompt` / `/history` / `/view`؛ Comfy Cloud `/api/*`                   |

## قابلیت‌های پشتیبانی‌شده

- تولید و ویرایش تصویر از یک JSON گردش‌کار (ویرایش 1 تصویر مرجع بارگذاری‌شده می‌پذیرد)
- تولید ویدئو از یک JSON گردش‌کار، متن‌به‌ویدئو یا تصویر‌به‌ویدئو (1 تصویر مرجع)
- تولید موسیقی/صدا از طریق ابزار مشترک `music_generate`، با 1 تصویر مرجع اختیاری
- دریافت خروجی از Node پیکربندی‌شده، یا از همه Nodeهای خروجی منطبق در صورت پیکربندی‌نشدن آن

## شروع به کار

بین اجرای ComfyUI روی دستگاه خود و استفاده از Comfy Cloud یکی را انتخاب کنید.

<Tabs>
  <Tab title="محلی">
    **مناسب برای:** اجرای نمونه ComfyUI خودتان روی دستگاه یا LAN.

    <Steps>
      <Step title="ComfyUI را به‌صورت محلی راه‌اندازی کنید">
        مطمئن شوید نمونه محلی ComfyUI در حال اجرا است (مقدار پیش‌فرض `http://127.0.0.1:8188` است).
      </Step>
      <Step title="JSON گردش‌کار خود را آماده کنید">
        یک فایل JSON گردش‌کار ComfyUI صادر یا ایجاد کنید. شناسه‌های Node مربوط به Node ورودی پرامپت و Node خروجی‌ای را که می‌خواهید OpenClaw از آن بخواند، یادداشت کنید.
      </Step>
      <Step title="ارائه‌دهنده را پیکربندی کنید">
        `mode: "local"` را تنظیم کنید و مسیر فایل گردش‌کار خود را بدهید. نمونه حداقلی تصویر:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "local",
                  baseUrl: "http://127.0.0.1:8188",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```
      </Step>
      <Step title="مدل پیش‌فرض را تنظیم کنید">
        OpenClaw را برای قابلیتی که پیکربندی کرده‌اید به مدل `comfy/workflow` هدایت کنید:

        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="بررسی کنید">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Comfy Cloud">
    **مناسب برای:** اجرای گردش‌کارها در Comfy Cloud بدون مدیریت منابع GPU محلی.

    <Steps>
      <Step title="یک کلید API دریافت کنید">
        در [comfy.org](https://comfy.org) ثبت‌نام کنید و از داشبورد حساب خود یک کلید API بسازید.
      </Step>
      <Step title="کلید API را تنظیم کنید">
        کلید خود را با یکی از روش‌های زیر ارائه کنید:

        ```bash
        # پرچم راه‌اندازی اولیه
        openclaw onboard --comfy-api-key "your-key"

        # متغیر محیطی (ترجیحی برای سرویس‌های پس‌زمینه)
        export COMFY_API_KEY="your-key"

        # متغیر محیطی جایگزین
        export COMFY_CLOUD_API_KEY="your-key"

        # یا به‌صورت درون‌خطی در پیکربندی
        openclaw config set plugins.entries.comfy.config.apiKey "your-key"
        ```
      </Step>
      <Step title="JSON گردش‌کار خود را آماده کنید">
        یک فایل JSON گردش‌کار ComfyUI صادر یا ایجاد کنید. شناسه‌های Node مربوط به Node ورودی پرامپت و Node خروجی را یادداشت کنید.
      </Step>
      <Step title="ارائه‌دهنده را پیکربندی کنید">
        `mode: "cloud"` را تنظیم کنید و مسیر فایل گردش‌کار خود را بدهید:

        ```json5
        {
          plugins: {
            entries: {
              comfy: {
                config: {
                  mode: "cloud",
                  image: {
                    workflowPath: "./workflows/flux-api.json",
                    promptNodeId: "6",
                    outputNodeId: "9",
                  },
                },
              },
            },
          },
        }
        ```

        <Tip>
        در حالت ابری، مقدار پیش‌فرض `baseUrl` برابر با `https://cloud.comfy.org` است. `baseUrl` را فقط برای یک نقطه پایانی ابری سفارشی تنظیم کنید.
        </Tip>
      </Step>
      <Step title="مدل پیش‌فرض را تنظیم کنید">
        ```json5
        {
          agents: {
            defaults: {
              imageGenerationModel: {
                primary: "comfy/workflow",
              },
            },
          },
        }
        ```
      </Step>
      <Step title="بررسی کنید">
        ```bash
        openclaw models list --provider comfy
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## پیکربندی

Comfy از تنظیمات اتصال مشترک سطح بالا به‌همراه بخش‌های گردش‌کار جداگانه برای هر قابلیت (`image`، `video`، `music`) پشتیبانی می‌کند:

```json5
{
  plugins: {
    entries: {
      comfy: {
        config: {
          mode: "local",
          baseUrl: "http://127.0.0.1:8188",
          image: {
            workflowPath: "./workflows/flux-api.json",
            promptNodeId: "6",
            outputNodeId: "9",
          },
          video: {
            workflowPath: "./workflows/video-api.json",
            promptNodeId: "12",
            outputNodeId: "21",
          },
          music: {
            workflowPath: "./workflows/music-api.json",
            promptNodeId: "3",
            outputNodeId: "18",
          },
        },
      },
    },
  },
}
```

### کلیدهای مشترک

| کلید                  | نوع                    | توضیحات                                                                               |
| --------------------- | ---------------------- | ------------------------------------------------------------------------------------- |
| `mode`                | `"local"` یا `"cloud"` | حالت اتصال. مقدار پیش‌فرض `"local"` است.                                               |
| `baseUrl`             | رشته                   | مقدار پیش‌فرض برای حالت محلی `http://127.0.0.1:8188` و برای حالت ابری `https://cloud.comfy.org` است. |
| `apiKey`              | رشته                   | کلید درون‌خطی اختیاری، به‌جای متغیرهای محیطی `COMFY_API_KEY` / `COMFY_CLOUD_API_KEY`. |
| `allowPrivateNetwork` | بولی                    | اجازه استفاده از `baseUrl` خصوصی/LAN در حالت ابری یا یک FQDN محلی با DNS خصوصی.              |

<Note>
در حالت `local`، مقادیر صریح IP حلقه‌بازگشتی/خصوصی و نام‌های سرویس تک‌برچسبی مانند `http://comfyui:8188` بدون `allowPrivateNetwork` کار می‌کنند. FQDNهای DNS خصوصی با ظاهر عمومی مانند `https://comfy.local.example.com` به `allowPrivateNetwork: true` نیاز دارند. اعتماد به مبدأ خصوصی همچنان به طرح، نام میزبان و پورت پیکربندی‌شده محدود می‌ماند؛ تغییرمسیرهای محلی نمی‌توانند نام میزبان پیکربندی‌شده را ترک کنند، درحالی‌که تغییرمسیرهای ابری به CDNهای عمومی با سیاست پیش‌فرض SSRF بررسی می‌شوند.
</Note>

### کلیدهای هر قابلیت

این کلیدها درون بخش‌های `image`، `video` یا `music` اعمال می‌شوند:

| کلید                         | الزامی | پیش‌فرض | توضیحات                                                                      |
| ---------------------------- | ------ | -------- | ---------------------------------------------------------------------------- |
| `workflow` یا `workflowPath` | بله    | --       | JSON گردش‌کار درون‌خطی، یا مسیر فایل JSON گردش‌کار ComfyUI.                 |
| `promptNodeId`               | بله    | --       | شناسه Node که پرامپت متنی را دریافت می‌کند.                                 |
| `promptInputName`            | خیر     | `"text"` | نام ورودی در Node پرامپت.                                                    |
| `outputNodeId`               | خیر     | --       | شناسه Node برای خواندن خروجی. اگر حذف شود، همه Nodeهای خروجی منطبق استفاده می‌شوند. |
| `pollIntervalMs`             | خیر     | `1500`   | فاصله نظرسنجی برحسب میلی‌ثانیه برای تکمیل کار.                              |
| `timeoutMs`                  | خیر     | `300000` | مهلت زمانی اجرای گردش‌کار برحسب میلی‌ثانیه.                                 |

بخش‌های `image` و `video` همچنین از یک Node ورودی تصویر مرجع پشتیبانی می‌کنند:

| کلید                  | الزامی                               | پیش‌فرض  | توضیحات                                           |
| --------------------- | ------------------------------------ | --------- | --------------------------------------------------- |
| `inputImageNodeId`    | بله (هنگام ارسال تصویر مرجع)        | --        | شناسه Node که تصویر مرجع بارگذاری‌شده را دریافت می‌کند. |
| `inputImageInputName` | خیر                                  | `"image"` | نام ورودی در Node تصویر.                          |

`apiKey` یک رشته تحت‌اللفظی یا یک شیء [ارجاع راز](/fa/gateway/configuration-reference#secrets) را می‌پذیرد.

## جزئیات گردش‌کار

<AccordionGroup>
  <Accordion title="گردش‌کارهای تصویر">
    مدل پیش‌فرض تصویر را روی `comfy/workflow` تنظیم کنید:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    **نمونه ویرایش با تصویر مرجع:**

    برای فعال‌کردن ویرایش تصویر با یک تصویر مرجع بارگذاری‌شده، `inputImageNodeId` را به پیکربندی تصویر خود اضافه کنید:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              image: {
                workflowPath: "./workflows/edit-api.json",
                promptNodeId: "6",
                inputImageNodeId: "7",
                inputImageInputName: "image",
                outputNodeId: "9",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="گردش‌کارهای ویدئو">
    مدل پیش‌فرض ویدئو را روی `comfy/workflow` تنظیم کنید:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "comfy/workflow",
          },
        },
      },
    }
    ```

    گردش‌کارهای ویدئویی Comfy از طریق گراف پیکربندی‌شده از متن‌به‌ویدئو و تصویر‌به‌ویدئو پشتیبانی می‌کنند.

    <Note>
    OpenClaw ویدئوهای ورودی را به گردش‌کارهای Comfy ارسال نمی‌کند. فقط پرامپت‌های متنی و تصاویر مرجع تکی به‌عنوان ورودی پشتیبانی می‌شوند.
    </Note>

  </Accordion>

  <Accordion title="گردش‌کارهای موسیقی">
    Plugin داخلی یک ارائه‌دهنده تولید موسیقی برای خروجی‌های صوتی یا موسیقی تعریف‌شده توسط گردش‌کار ثبت می‌کند که از طریق ابزار مشترک `music_generate` ارائه می‌شود. این ابزار یک تصویر مرجع اختیاری (حداکثر 1) می‌پذیرد:

    ```text
    /tool music_generate prompt="حلقه سینث امبینت گرم با بافت نرم نوار"
    ```

    از بخش پیکربندی `music` برای تعیین JSON گردش‌کار صوتی و Node خروجی خود استفاده کنید.

  </Accordion>

  <Accordion title="سازگاری با نسخه‌های پیشین">
    پیکربندی موجود تصویر در سطح بالا (بدون بخش تودرتوی `image`) همچنان کار می‌کند:

    ```json5
    {
      plugins: {
        entries: {
          comfy: {
            config: {
              workflowPath: "./workflows/flux-api.json",
              promptNodeId: "6",
              outputNodeId: "9",
            },
          },
        },
      },
    }
    ```

    OpenClaw آن ساختار قدیمی را به‌عنوان پیکربندی گردش‌کار تصویر در نظر می‌گیرد. نیازی نیست فوراً مهاجرت کنید، اما بخش‌های تودرتوی `image` / `video` / `music` برای راه‌اندازی‌های جدید توصیه می‌شوند. اگر فقط از تولید تصویر استفاده می‌کنید، پیکربندی مسطح قدیمی و بخش تودرتوی جدید `image` از نظر عملکرد معادل‌اند.

  </Accordion>

  <Accordion title="آزمایش‌های زنده">
    پوشش آزمایش زنده اختیاری برای Plugin همراه وجود دارد:

    ```bash
    OPENCLAW_LIVE_TEST=1 COMFY_LIVE_TEST=1 pnpm test:live -- extensions/comfy/comfy.live.test.ts
    ```

    آزمایش زنده موارد منفرد تصویر، ویدئو یا موسیقی را رد می‌کند، مگر اینکه بخش متناظر گردش‌کار Comfy پیکربندی شده باشد.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="تولید تصویر" href="/fa/tools/image-generation" icon="image">
    پیکربندی و استفاده از ابزار تولید تصویر.
  </Card>
  <Card title="تولید ویدئو" href="/fa/tools/video-generation" icon="video">
    پیکربندی و استفاده از ابزار تولید ویدئو.
  </Card>
  <Card title="تولید موسیقی" href="/fa/tools/music-generation" icon="music">
    راه‌اندازی ابزار تولید موسیقی و صدا.
  </Card>
  <Card title="فهرست ارائه‌دهندگان" href="/fa/providers/index" icon="layers">
    نمای کلی همه ارائه‌دهندگان و ارجاع‌های مدل.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    مرجع کامل پیکربندی، شامل پیش‌فرض‌های عامل.
  </Card>
</CardGroup>
