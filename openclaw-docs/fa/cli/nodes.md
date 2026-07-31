---
read_when:
    - شما در حال مدیریت Nodeهای جفت‌شده (دوربین‌ها، صفحه‌نمایش، بوم) هستید
    - باید درخواست‌ها را تأیید کنید یا فرمان‌های Node را فراخوانی کنید
summary: مرجع CLI برای `openclaw nodes` (وضعیت، جفت‌سازی، فراخوانی، دوربین/بوم/صفحه‌نمایش/موقعیت مکانی/اعلان)
title: نودها
x-i18n:
    generated_at: "2026-07-27T13:59:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 53003bcd3d30b0e754aa0717452700595c0cf69d9ecd6301b8a1bf320ea1838a
    source_path: cli/nodes.md
    workflow: 16
---

# `openclaw nodes`

Nodeهای جفت‌شده (دستگاه‌ها) را مدیریت و قابلیت‌های Node را فراخوانی کنید.

مرتبط: [نمای کلی Nodeها](/fa/nodes) - [حضور فعال رایانه](/fa/nodes/presence) - [Nodeهای دوربین](/fa/nodes/camera) - [Nodeهای تصویر](/fa/nodes/images)

گزینه‌های مشترک در همه زیرفرمان‌ها: `--url <url>`، `--token <token>`، `--timeout <ms>` (پیش‌فرض `10000`)، `--json`.

## وضعیت

```bash
openclaw nodes status
openclaw nodes status --connected
openclaw nodes status --last-connected 24h
openclaw nodes list
openclaw nodes describe --node <idOrNameOrIp>
```

`status` و `list` هر دو `--connected` (فقط Nodeهای متصل) و `--last-connected <duration>` (برای مثال `24h`، `7d`؛ فقط Nodeهایی که در بازه زمانی مشخص‌شده متصل شده‌اند) را می‌پذیرند. `list`، Nodeهای در انتظار و جفت‌شده را در جدول‌های جداگانه نمایش می‌دهد و ردیف‌های جفت‌شده شامل مدت‌زمان سپری‌شده از آخرین اتصال (Last Connect) هستند؛ `status` یک جدول ادغام‌شده شامل جزئیات قابلیت، نسخه و آخرین ورودی هر Node نمایش می‌دهد. یک Node متصل macOS تنها پس از آن آخرین ورودی را گزارش می‌کند که کاربر **تشخیص رایانه فعال** را فعال و مجوز Accessibility را اعطا کند؛ تازه‌ترین ردیف با `active` علامت‌گذاری می‌شود. [حضور فعال رایانه](/fa/nodes/presence) را ببینید. `describe` قابلیت‌ها، مجوزها، فعالیت و فرمان‌های فراخوانی مؤثر/در انتظار یک Node را نمایش می‌دهد.

## جفت‌سازی

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
openclaw nodes reject <requestId>
openclaw nodes remove --node <id|name|ip>
openclaw nodes rename --node <id|name|ip> --name <displayName>
```

این فرمان‌ها مخزن `node.pair.*` تحت مالکیت Gateway را کنترل می‌کنند که از جفت‌سازی دستگاه (`openclaw devices approve`) ــ که دست‌دهی `connect` مربوط به WS در Node را کنترل می‌کند ــ جدا است. برای آشنایی با ارتباط این دو، [Nodeها](/fa/nodes) را ببینید.

- `remove` ورودی نقش جفت‌شده Node را لغو می‌کند. برای یک Node مبتنی بر دستگاه، این کار نقش `node` را در مخزن جفت‌سازی دستگاه لغو و نشست‌های نقش Node آن را قطع می‌کند: دستگاهی با نقش‌های ترکیبی ردیف خود را حفظ می‌کند و فقط نقش `node` را از دست می‌دهد؛ ردیف دستگاهی که فقط Node است حذف می‌شود. همچنین هر رکورد منطبق قدیمیِ جفت‌سازی Node تحت مالکیت Gateway را پاک می‌کند.
- `pending` فقط به دامنه `operator.pairing` نیاز دارد.
- `gateway.nodes.pairing.autoApproveCidrs` می‌تواند برای جفت‌سازی نخستین‌بارِ دستگاه `role: node` که صراحتاً مورد اعتماد است، مرحله انتظار را نادیده بگیرد. به‌طور پیش‌فرض غیرفعال است؛ ارتقای نقش‌ها را تأیید نمی‌کند.
- `gateway.nodes.pairing.sshVerify` (به‌طور پیش‌فرض فعال) جفت‌سازی نخستین‌بارِ دستگاه `role: node` را هنگامی به‌طور خودکار تأیید می‌کند که Gateway بتواند کلید دستگاه را از طریق SSH به میزبان Node اعتبارسنجی کند؛ نخستین سطح قابلیت نیز در همان مرحله تأیید می‌شود. [جفت‌سازی Node](/fa/gateway/pairing#ssh-verified-device-auto-approval-default) را ببینید.
- الزامات دامنه `approve` از فرمان‌های اعلام‌شده در درخواست در انتظار پیروی می‌کنند:
  - درخواست بدون فرمان: `operator.pairing`
  - فرمان‌های عادی Node: `operator.pairing` + `operator.write`
  - فرمان‌های حساس مدیریتی (`system.run`، `system.run.prepare`، `system.which`، `browser.proxy`، `fs.listDir` و `system.execApprovals.get/set`): `operator.pairing` + `operator.admin`
- دامنه `remove`: `operator.pairing` می‌تواند ردیف‌های Node غیرعملگر را حذف کند؛ فراخواننده‌ای با توکن دستگاه که نقش Node خود را در دستگاهی با نقش‌های ترکیبی لغو می‌کند، علاوه بر آن به `operator.admin` نیاز دارد.

## فراخوانی

```bash
openclaw nodes invoke --node <id> --command system.which --params '{"bins":["uname"]}'
```

پرچم‌ها:

- `--command <command>` (الزامی): برای مثال `canvas.eval`.
- `--params <json>`: رشته شیء JSON (پیش‌فرض `{}`).
- `--invoke-timeout <ms>`: مهلت زمانی فراخوانی Node (پیش‌فرض `15000`).
- `--idempotency-key <key>`: کلید اختیاری هم‌توانی.

`system.run` و `system.run.prepare` در اینجا مسدود هستند؛ برای اجرای پوسته، به‌جای آن از ابزار `exec` همراه با `host=node` استفاده کنید. `system.which` از طریق `invoke` مجاز است.

## اعلان، پوش، موقعیت مکانی، صفحه‌نمایش

```bash
openclaw nodes notify --node <id> --title "Build" --body "Done" --priority timeSensitive
openclaw nodes push --node <id> --title "OpenClaw" --environment sandbox
openclaw nodes location get --node <id> --accuracy precise
openclaw nodes screen record --node <id> --duration 10s --fps 10 --out ./clip.mp4
```

- `notify` یک اعلان محلی به Nodeای ارسال می‌کند که `system.notify` را اعلام کرده باشد؛ از جمله Nodeهای macOS، iOS، Android و watchOS مستقیم. تحویل مستقیم به watchOS مستلزم فعال‌بودن OpenClaw است. به `--title` یا `--body` نیاز دارد. گزینه‌ها: `--sound <name>`، `--priority <passive|active|timeSensitive>`، `--delivery <system|overlay|auto>` (پیش‌فرض `system`)، `--invoke-timeout <ms>` (پیش‌فرض `15000`).
- `push` یک پوش آزمایشی APNs به Node دارای iOS ارسال می‌کند. گزینه‌ها: `--title <text>` (پیش‌فرض `OpenClaw`)، `--body <text>`، `--environment <sandbox|production>` برای بازنویسی محیط APNs شناسایی‌شده.
- `location get` موقعیت مکانی فعلی Node را دریافت می‌کند. گزینه‌ها: `--max-age <ms>` (استفاده مجدد از موقعیت ذخیره‌شده در حافظه نهان)، `--accuracy <coarse|balanced|precise>`، `--location-timeout <ms>` (پیش‌فرض `10000`)، `--invoke-timeout <ms>` (پیش‌فرض `20000`).
- `screen record` یک کلیپ کوتاه ضبط و مسیر ذخیره‌شده را نمایش می‌دهد (یا با `--json` خروجی JSON می‌نویسد). گزینه‌ها: `--screen <index>` (پیش‌فرض `0`)، `--duration <ms|10s>` (پیش‌فرض `10000`)، `--fps <fps>` (پیش‌فرض `10`)، `--no-audio`، `--out <path>`، `--invoke-timeout <ms>` (پیش‌فرض `120000`).

فرمان‌های دوربین و Canvas مستندات جداگانه‌ای دارند: [Nodeهای دوربین](/fa/nodes/camera)، [Canvas](/fa/platforms/mac/canvas). Canvas توسط Plugin آزمایشی همراه Canvas پیاده‌سازی شده است؛ هسته `openclaw nodes canvas` را به‌عنوان نقطه اتصال سازگاری حفظ می‌کند.

## مرتبط

- [مرجع CLI](/fa/cli)
- [Nodeها](/fa/nodes)
