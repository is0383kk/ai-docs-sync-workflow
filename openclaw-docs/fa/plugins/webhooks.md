---
read_when:
    - می‌خواهید TaskFlowها را از یک سیستم خارجی راه‌اندازی یا هدایت کنید
    - در حال پیکربندی Plugin همراه Webhookها هستید
summary: 'Plugin وب‌هوک‌ها: ورودی احراز هویت‌شده TaskFlow برای خودکارسازی خارجی مورد اعتماد'
title: Plugin وب‌هوک‌ها
x-i18n:
    generated_at: "2026-07-27T16:02:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77e455450d6183635c76a1e8002feeb287deb4ff242dbd555ef9d0f2b21ce5f6
    source_path: plugins/webhooks.md
    workflow: 16
---

Plugin وب‌هوک‌ها مسیرهای HTTP احراز هویت‌شده‌ای اضافه می‌کند تا یک سامانه خارجی قابل‌اعتماد
(Zapier، n8n، یک کار CI یا یک سرویس داخلی) بتواند TaskFlowهای مدیریت‌شده
OpenClaw را از طریق HTTP ایجاد و هدایت کند، بدون آنکه نیاز به نوشتن یک Plugin سفارشی باشد.

این Plugin درون فرایند Gateway اجرا می‌شود. برای یک Gateway راه دور، آن را روی
همان میزبان نصب و پیکربندی کنید، سپس Gateway را راه‌اندازی مجدد کنید. این Plugin
بدون هیچ مسیر پیکربندی‌شده‌ای ارائه می‌شود، بنابراین تا زمانی که دست‌کم یک مسیر اضافه نکنید، هیچ عملی انجام نمی‌دهد.

## پیکربندی مسیرها

پیکربندی را زیر `plugins.entries.webhooks.config` تنظیم کنید:

```json5
{
  plugins: {
    entries: {
      webhooks: {
        enabled: true,
        config: {
          routes: {
            zapier: {
              path: "/plugins/webhooks/zapier",
              sessionKey: "agent:main:main",
              secret: {
                source: "env",
                provider: "default",
                id: "OPENCLAW_WEBHOOK_SECRET",
              },
              controllerId: "webhooks/zapier",
              description: "پل TaskFlow برای Zapier",
            },
          },
        },
      },
    },
  },
}
```

فیلدهای مسیر:

| فیلد           | الزامی | پیش‌فرض                      | توضیحات                                                         |
| -------------- | ------ | ----------------------------- | --------------------------------------------------------------- |
| `enabled`      | خیر    | `true`                        |                                                                 |
| `path`         | خیر    | `/plugins/webhooks/<routeId>` | باید در میان مسیرها یکتا باشد.                                  |
| `sessionKey`   | بله    | -                             | نشستی که مالک TaskFlowهای متصل است.                             |
| `secret`       | بله    | -                             | رشته ساده یا یک SecretRef (در ادامه).                           |
| `controllerId` | خیر    | `webhooks/<routeId>`          | به‌عنوان کنترل‌گر پیش‌فرض `create_flow` استفاده می‌شود. |
| `description`  | خیر    | -                             | فقط یادداشت اپراتور.                                            |

`secret` یک رشته ساده یا SecretRef را می‌پذیرد: `{ source: "env" | "file" | "exec", provider: "default", id: "..." }`.

SecretRefها در تصویر لحظه‌ای پیکربندی هنگام راه‌اندازی Gateway تفکیک می‌شوند. هنگامی که
secret یک مسیر قابل تفکیک نباشد، Gateway به کار خود ادامه می‌دهد و دقیقاً همان مسیر
ثبت‌شده اما غیرفعال باقی می‌ماند: درخواست‌ها یک خطای عمومی احراز هویت دریافت می‌کنند (`401`).
سایر مسیرها همچنان در دسترس می‌مانند. منبع SecretRef را اصلاح کنید، سپس برای فعال‌سازی
تصویر لحظه‌ای جدید، Gateway را بازخوانی یا راه‌اندازی مجدد کنید. مقادیر SecretRef هرگز
در مسیر عمومی درخواست تفکیک نمی‌شوند.

## مدل امنیتی

هر مسیر با اختیار TaskFlow مربوط به `sessionKey` پیکربندی‌شده خود عمل می‌کند: این مسیر
می‌تواند هر TaskFlow متعلق به آن نشست را بررسی و تغییر دهد. دسترسی به TaskFlow
همیشه از طریق `api.runtime.tasks.managedFlows.bindSession(...)` انجام می‌شود، بنابراین یک
مسیر هرگز نمی‌تواند خارج از نشست متصل به خود عمل کند. برای محدودکردن دامنه آسیب:

- برای هر مسیر از یک secret قوی و یکتا استفاده کنید.
- یک SecretRef را به secret متن ساده درون‌خطی ترجیح دهید.
- مسیرها را به محدودترین نشستی متصل کنید که برای گردش‌کار مناسب است.
- فقط مسیر وب‌هوک مشخصی را که نیاز دارید در معرض دسترسی قرار دهید.

ترتیب رسیدگی به درخواست برای هر مسیر: بررسی متد HTTP (فقط `POST`) و
`Content-Type: application/json`، سپس محدودسازی نرخ با پنجره ثابت (120
درخواست در هر پنجره 60 ثانیه‌ای برای هر کلید مسیر+IP کلاینت، با حداکثر 4,096 کلید
ردیابی‌شده)، سپس محدودسازی درخواست‌های در حال اجرا (8 درخواست هم‌زمان برای هر کلید، با حداکثر
4,096 کلید ردیابی‌شده)، سپس احراز هویت با secret مشترک، و پس از آن خواندن بدنه JSON با محدودیت
256 KB / 15 ثانیه. درخواست‌هایی که در یک بررسی زودتر رد شوند، هرگز به
بررسی‌های بعدی نمی‌رسند.

## قالب درخواست

درخواست‌های `POST` را همراه با `Content-Type: application/json` و یکی از
`Authorization: Bearer <secret>` یا `x-openclaw-webhook-secret: <secret>` ارسال کنید:

```bash
curl -X POST https://gateway.example.com/plugins/webhooks/zapier \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SHARED_SECRET' \
  -d '{"action":"create_flow","goal":"بازبینی صف ورودی"}'
```

## کنش‌های پشتیبانی‌شده

| کنش               | هدف                                                                  |
| ------------------ | -------------------------------------------------------------------- |
| `create_flow`      | ایجاد یک TaskFlow مدیریت‌شده برای نشست مسیر.                         |
| `get_flow`         | دریافت یک TaskFlow بر اساس شناسه.                                    |
| `list_flows`       | فهرست‌کردن TaskFlowهای نشست مسیر.                                    |
| `find_latest_flow` | دریافت TaskFlowای که اخیراً به‌روزرسانی شده است.                     |
| `resolve_flow`     | یافتن یک TaskFlow بر اساس توکن مات.                                  |
| `get_task_summary` | دریافت خلاصه وظیفه برای یک TaskFlow.                                 |
| `set_waiting`      | علامت‌گذاری TaskFlow به‌عنوان منتظر، با داده‌های اختیاری حالت/انتظار. |
| `resume_flow`      | ازسرگیری یک TaskFlow منتظر/مسدودشده.                                 |
| `finish_flow`      | علامت‌گذاری TaskFlow به‌عنوان پایان‌یافته.                            |
| `fail_flow`        | علامت‌گذاری TaskFlow به‌عنوان ناموفق.                                |
| `request_cancel`   | درخواست لغو مشارکتی.                                                  |
| `cancel_flow`      | لغو یک TaskFlow (اگر فرزندان همچنان فعال باشند، ممکن است `202` برگرداند). |
| `run_task`         | ایجاد یک وظیفه فرزند مدیریت‌شده درون یک TaskFlow موجود.               |

کنش‌های تغییردهنده (`set_waiting`، `resume_flow`، `finish_flow`، `fail_flow`،
`request_cancel`) برای هم‌زمانی خوش‌بینانه به `flowId` و `expectedRevision`
نیاز دارند؛ یک بازبینی قدیمی `409 revision_conflict` برمی‌گرداند.

### `create_flow`

```json
{
  "action": "create_flow",
  "goal": "بازبینی صف ورودی",
  "status": "queued",
  "notifyPolicy": "done_only"
}
```

### `run_task`

مقادیر مجاز `runtime`: `subagent`، `acp`. مقادیر `startedAt`، `lastEventAt` و
`progressSummary` فقط زمانی معتبرند که `status` برابر با `"running"` باشد؛ ارسال آن‌ها
با هر وضعیت دیگری `400 invalid_request` برمی‌گرداند.

```json
{
  "action": "run_task",
  "flowId": "flow_123",
  "runtime": "acp",
  "childSessionKey": "agent:main:acp:worker",
  "task": "بررسی دسته بعدی پیام‌ها"
}
```

## ساختار پاسخ

```json
{
  "ok": true,
  "routeId": "zapier",
  "result": {}
}
```

```json
{
  "ok": false,
  "routeId": "zapier",
  "code": "not_found",
  "error": "TaskFlow یافت نشد.",
  "result": {}
}
```

نماهای جریان و وظیفه هرگز شامل فراداده مالک/نشست نیستند، بنابراین پاسخ‌ها نمی‌توانند
`sessionKey` متصل به مسیر را افشا کنند. مقادیر `code` شامل `not_found`،
`not_managed`، `revision_conflict`، `persist_failed`، `cancel_requested`،
`cancel_pending`، `terminal`، `invalid_request`، `request_rejected` و
کدهای جایگزین ویژه هر کنش (`mutation_rejected`، `create_rejected`،
`task_not_created`، `cancel_rejected`) هستند که وقتی یک تغییر به دلیلی
رد شود که کدهای نام‌گذاری‌شده بالا آن را پوشش نمی‌دهند، استفاده می‌شوند.

## مرتبط

- [هوک‌ها](/fa/automation/hooks) - هوک‌های داخلی رویدادمحور در مقایسه با این پل TaskFlow مبتنی بر HTTP
- [وب‌هوک‌های Gateway (پیکربندی `hooks.*`)](/fa/automation/cron-jobs#webhooks) - قابلیت جداگانه نقطه پایانی عمومی HTTP در Gateway؛ با مسیرهای این Plugin یکسان نیست
- [SDK زمان اجرای Plugin](/fa/plugins/sdk-runtime)
- [وب‌هوک‌های CLI](/fa/cli/webhooks)
