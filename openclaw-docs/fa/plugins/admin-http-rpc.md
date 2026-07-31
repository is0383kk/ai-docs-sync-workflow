---
read_when:
    - ساخت ابزارهای میزبان که نمی‌توانند از کلاینت RPC مبتنی بر WebSocket برای Gateway استفاده کنند
    - در دسترس قرار دادن خودکارسازی مدیریت Gateway در پشت یک ورودی خصوصی و مورد اعتماد
    - ممیزی مدل امنیتی دسترسی HTTP به متدهای Gateway
summary: روش‌های منتخب صفحهٔ کنترل Gateway را از طریق Plugin همراه و اختیاری admin-http-rpc در دسترس قرار دهید
title: Plugin فراخوانی رویهٔ راه‌دور HTTP مدیریت
x-i18n:
    generated_at: "2026-07-27T15:49:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0709081efd0ce65cef7edac54df9a71978cbad17e2b25df83ac9075de938376c
    source_path: plugins/admin-http-rpc.md
    workflow: 16
---

Plugin همراهِ `admin-http-rpc` مجموعه‌ای فهرست‌مجاز از متدهای صفحهٔ کنترل Gateway را از طریق HTTP ارائه می‌کند؛ برای خودکارسازی روی میزبان‌های مورداعتمادی که نمی‌توانند اتصال WebSocket به Gateway را باز نگه دارند.

این Plugin همراه OpenClaw ارائه می‌شود، اما به‌طور پیش‌فرض غیرفعال است؛ هنگام غیرفعال‌بودن، مسیر ثبت نمی‌شود. هنگام فعال‌بودن، `POST /api/v1/admin/rpc` را روی همان شنوندهٔ Gateway (`http://<gateway-host>:<port>/api/v1/admin/rpc`) اضافه می‌کند.

آن را فقط برای ابزارهای خصوصی میزبان، خودکارسازی tailnet یا ورودی داخلی مورداعتماد فعال کنید. هرگز این مسیر را مستقیماً در معرض اینترنت عمومی قرار ندهید.

## پیش از فعال‌سازی

RPC مدیریتی HTTP یک سطح کامل صفحهٔ کنترل اپراتور است: هر فراخواننده‌ای که احراز هویت HTTP ‏Gateway را با موفقیت انجام دهد، می‌تواند متدهای فهرست‌مجاز زیر را فراخوانی کند. آن را تنها زمانی فعال کنید که همهٔ موارد زیر برقرار باشند:

- فراخواننده برای راهبری Gateway مورداعتماد است.
- فراخواننده نمی‌تواند از کلاینت RPC ‏WebSocket استفاده کند.
- مسیر فقط روی loopback، یک tailnet یا ورودی خصوصی احرازهویت‌شده قابل دسترسی است.
- متدهای مجاز را بازبینی کرده‌اید و آن‌ها با خودکارسازی موردنظرتان مطابقت دارند.

برای کلاینت‌های OpenClaw و ابزارهای تعاملی که می‌توانند اتصال WebSocket به Gateway را باز نگه دارند، به‌جای آن از RPC ‏WebSocket استفاده کنید.

## فعال‌سازی

Plugin همراه را فعال کنید:

<Tabs>
  <Tab title="CLI">
    ```bash
    openclaw plugins enable admin-http-rpc
    openclaw gateway restart
    ```
  </Tab>
  <Tab title="پیکربندی">
    ```json5
    {
      plugins: {
        entries: {
          "admin-http-rpc": { enabled: true },
        },
      },
    }
    ```
  </Tab>
</Tabs>

مسیر هنگام راه‌اندازی Plugin ثبت می‌شود؛ بنابراین پس از تغییر پیکربندی Plugin، ‏Gateway را راه‌اندازی مجدد کنید.

وقتی دیگر به سطح HTTP نیاز ندارید، آن را غیرفعال کنید:

```bash
openclaw plugins disable admin-http-rpc
openclaw gateway restart
```

## تأیید مسیر

از `health` به‌عنوان کوچک‌ترین درخواست امن استفاده کنید:

```bash
curl -sS http://<gateway-host>:<port>/api/v1/admin/rpc \
  -H 'Authorization: Bearer <gateway-token>' \
  -H 'Content-Type: application/json' \
  -d '{"method":"health","params":{}}'
```

پاسخ موفق دارای `ok: true` است:

```json
{
  "id": "generated-request-id",
  "ok": true,
  "payload": {
    "status": "ok"
  }
}
```

وقتی Plugin غیرفعال است، مسیر `404` را برمی‌گرداند، زیرا ثبت نشده است.

## احراز هویت

مسیر Plugin از احراز هویت HTTP ‏Gateway استفاده می‌کند.

روش‌های رایج احراز هویت:

- احراز هویت با راز مشترک (`gateway.auth.mode="token"` یا `"password"`): `Authorization: Bearer <token-or-password>`
- احراز هویت HTTP مورداعتماد و حامل هویت (`gateway.auth.mode="trusted-proxy"`): مسیر را از پراکسی آگاه از هویتِ پیکربندی‌شده عبور دهید تا سرآیندهای هویت لازم را تزریق کند
- احراز هویت بازِ ورودی خصوصی (`gateway.auth.mode="none"`): به سرآیند احراز هویت نیازی نیست

## مدل امنیتی

با این Plugin مانند یک سطح کامل اپراتوری Gateway رفتار کنید.

- فعال‌کردن Plugin عمداً دسترسی به متدهای RPC مدیریتی فهرست‌مجاز را در `/api/v1/admin/rpc` فراهم می‌کند.
- Plugin قرارداد مانیفست رزروشدهٔ `contracts.gatewayMethodDispatch: ["authenticated-request"]` را اعلام می‌کند که به مسیر HTTP احرازهویت‌شده توسط Gateway اجازه می‌دهد متدهای صفحهٔ کنترل را درون همان فرایند توزیع کند. این یک محیط ایزوله نیست: قرارداد از استفادهٔ تصادفی از کمک‌تابع‌های رزروشدهٔ SDK جلوگیری می‌کند، اما Pluginهای مورداعتماد همچنان در فرایند Gateway اجرا می‌شوند.
- احراز هویت bearer با راز مشترک (حالت‌های `token`/`password`) مالکیت راز اپراتوری Gateway را اثبات می‌کند؛ سرآیندهای محدودتر `x-openclaw-scopes` در آن مسیر نادیده گرفته می‌شوند و پیش‌فرض‌های عادی اپراتور کامل بازیابی می‌شوند.
- احراز هویت HTTP مورداعتماد و حامل هویت (حالت `trusted-proxy`) در صورت وجود، `x-openclaw-scopes` را رعایت می‌کند.
- `gateway.auth.mode="none"` به این معناست که در صورت فعال‌بودن Plugin، این مسیر احراز هویت نمی‌شود. از آن فقط پشت یک ورودی خصوصی که کاملاً به آن اعتماد دارید استفاده کنید.
- پس از موفقیت احراز هویت مسیر Plugin، درخواست‌ها از همان کنترل‌کننده‌های متد Gateway و بررسی‌های دامنهٔ دسترسی RPC ‏WebSocket عبور می‌کنند.
- مسیر طی یک اجارهٔ تعلیق آماده‌شده همچنان قابل دسترسی می‌ماند. اعتبارسنجی محدود درخواست و پاسخ محلی کشف `commands.list` همچنان در دسترس‌اند. از میان متدهایی که به Gateway توزیع می‌شوند، فقط `gateway.suspend.prepare`، `gateway.suspend.status` و `gateway.suspend.resume` می‌توانند هنگام بسته‌بودن پذیرش اجرا شوند؛ سایر متدهای فهرست‌مجاز پاسخ عادی و قابل‌تلاش‌مجدد `UNAVAILABLE` ‏Gateway را برمی‌گردانند.
- این مسیر را روی loopback، ‏tailnet یا یک ورودی خصوصی مورداعتماد نگه دارید. آن را مستقیماً در معرض اینترنت عمومی قرار ندهید. هنگامی که فراخواننده‌ها از مرزهای اعتماد عبور می‌کنند، از Gatewayهای جداگانه استفاده کنید.

## درخواست

```http
POST /api/v1/admin/rpc
Authorization: Bearer <gateway-token>
Content-Type: application/json
```

```json
{
  "id": "optional-request-id",
  "method": "health",
  "params": {}
}
```

فیلدها:

- `id` (رشته، اختیاری): در پاسخ کپی می‌شود. در صورت حذف، یک UUID تولید می‌شود.
- `method` (رشته، الزامی): نام متد مجاز Gateway.
- `params` (هر نوع، اختیاری): پارامترهای ویژهٔ متد.

حداکثر اندازهٔ پیش‌فرض بدنهٔ درخواست 1 MB است.

## پاسخ

پاسخ‌های موفق از ساختار RPC ‏Gateway استفاده می‌کنند:

```json
{
  "id": "optional-request-id",
  "ok": true,
  "payload": {}
}
```

خطاهای متد Gateway از ساختار زیر استفاده می‌کنند:

```json
{
  "id": "optional-request-id",
  "ok": false,
  "error": {
    "code": "INVALID_REQUEST",
    "message": "bad params"
  }
}
```

وضعیت HTTP از کد خطا پیروی می‌کند:

| کد خطا                 | وضعیت HTTP |
| -------------------------- | ----------- |
| `INVALID_REQUEST`          | 400         |
| `APPROVAL_NOT_FOUND`       | 404         |
| `NOT_LINKED`، `NOT_PAIRED` | 409         |
| `UNAVAILABLE`              | 503         |
| `AGENT_TIMEOUT`            | 504         |
| هر کد دیگر             | 500         |

## متدهای مجاز

- کشف: `commands.list`
  نام متدهای RPC ‏HTTP مجاز این Plugin را برمی‌گرداند.
- gateway: `health`، `status`، `logs.tail`، `usage.status`، `usage.cost`، `gateway.restart.request`، `gateway.suspend.prepare`، `gateway.suspend.status`، `gateway.suspend.resume`
- پیکربندی: `config.get`، `config.schema`، `config.schema.lookup`، `config.set`، `config.patch`، `config.apply`
- کانال‌ها: `channels.status`، `channels.start`، `channels.stop`، `channels.logout`
- وب: `web.login.start`، `web.login.wait`
- مدل‌ها: `models.list`، `models.authStatus`
- عامل‌ها: `agents.list`، `agents.create`، `agents.update`، `agents.delete`
- تأییدها: `exec.approvals.get`، `exec.approvals.set`، `exec.approvals.node.get`، `exec.approvals.node.set`
- cron: `cron.status`، `cron.list`، `cron.get`، `cron.runs`، `cron.add`، `cron.update`، `cron.remove`، `cron.run`
- دستگاه‌ها: `device.pair.list`، `device.pair.approve`، `device.pair.reject`، `device.pair.remove`
- Nodeها: `node.list`، `node.describe`، `node.pair.list`، `node.pair.approve`، `node.pair.reject`، `node.pair.remove`، `node.rename`
- وظایف: `tasks.list`، `tasks.get`، `tasks.cancel`
- عیب‌یابی: `doctor.memory.status`، `update.status`

سایر متدهای Gateway تا زمانی که عمداً اضافه نشوند مسدود هستند.

## مقایسه با WebSocket

مسیر عادی RPC ‏WebSocket در Gateway همچنان API ترجیحی صفحهٔ کنترل برای کلاینت‌های OpenClaw است. از RPC مدیریتی HTTP فقط برای ابزارهای میزبان که به یک سطح درخواست/پاسخ HTTP نیاز دارند استفاده کنید.

کلاینت‌های WebSocket دارای توکن مشترک و بدون هویت دستگاه مورداعتماد نمی‌توانند هنگام اتصال، دامنه‌های دسترسی مدیریتی را خودشان اعلام کنند. RPC مدیریتی HTTP عمداً از مدل موجود اپراتور HTTP مورداعتماد پیروی می‌کند: وقتی Plugin فعال است، احراز هویت bearer با راز مشترک برای این سطح مدیریتی به‌عنوان دسترسی کامل اپراتور در نظر گرفته می‌شود.

## عیب‌یابی

`404 Not Found`

: Plugin غیرفعال است، Gateway پس از فعال‌شدن آن راه‌اندازی مجدد نشده است، یا درخواست به فرایند دیگری از Gateway ارسال می‌شود.

`401 Unauthorized`

: درخواست الزامات احراز هویت HTTP ‏Gateway را برآورده نکرد. توکن bearer یا سرآیندهای هویت پراکسی مورداعتماد را بررسی کنید.

`405 Method Not Allowed`

: درخواست از چیزی غیر از `POST` استفاده کرده است.

`413 Payload Too Large`

: بدنهٔ درخواست از محدودیت 1 MB فراتر رفته است.

`400 INVALID_REQUEST`

: بدنهٔ درخواست JSON معتبر نیست، فیلد `method` وجود ندارد، متد در فهرست‌مجاز Plugin نیست، یا شناسهٔ ازسرگیری تعلیق با اجارهٔ فعال مطابقت ندارد.

`503 UNAVAILABLE`

: متد Gateway در حال راه‌اندازی، دارای محدودیت نرخ، معلق یا منتظر عملیات رقیب تعلیق/ازسرگیری است. در صورت وجود، `error.details` را بررسی کنید و پیش از تلاش مجدد، `error.retryAfterMs` را رعایت کنید.

## مرتبط

- [دامنه‌های دسترسی اپراتور](/fa/gateway/operator-scopes)
- [امنیت Gateway](/fa/gateway/security)
- [دسترسی از راه دور](/fa/gateway/remote)
- [مانیفست Plugin](/fa/plugins/manifest#contracts-reference)
- [زیرمسیرهای SDK](/fa/plugins/sdk-subpaths)
