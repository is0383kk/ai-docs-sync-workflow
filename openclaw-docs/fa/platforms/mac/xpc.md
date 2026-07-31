---
read_when:
    - ویرایش قراردادهای IPC یا IPC برنامه نوار منو
summary: معماری IPC در macOS برای برنامه OpenClaw، انتقال Node در Gateway و PeekabooBridge
title: IPC در macOS
x-i18n:
    generated_at: "2026-07-27T16:44:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# معماری IPC در macOS برای OpenClaw

یک سوکت محلی Unix، سرویس میزبان Node را برای تأییدیه‌های اجرا و `system.run` به برنامه macOS متصل می‌کند. یک CLI اشکال‌زدایی `openclaw-mac` با نام (`apps/macos/Sources/OpenClawMacCLI`) برای بررسی‌های کشف/اتصال وجود دارد؛ کنش‌های عامل همچنان از طریق WebSocket مربوط به Gateway و `node.invoke` جریان می‌یابند. مسیر مبتنی بر Node با نام `computer.act`، خودکارسازی تعبیه‌شده Peekaboo را درون همان فرایند اجرا می‌کند؛ کلاینت‌های مستقل Peekaboo از PeekabooBridge استفاده می‌کنند.

## اهداف

- یک نمونه واحد از برنامه GUI که مالک تمام کارهای مرتبط با TCC است (اعلان‌ها، ضبط صفحه، میکروفون، گفتار، AppleScript).
- یک سطح کوچک برای خودکارسازی: Gateway + فرمان‌های Node، `computer.act` درون‌فرایندی، به‌علاوه PeekabooBridge برای کلاینت‌های مستقل خودکارسازی رابط کاربری.
- مجوزهای قابل‌پیش‌بینی: همیشه همان شناسه بسته امضاشده که توسط launchd راه‌اندازی می‌شود تا مجوزهای TCC پایدار بمانند.

## نحوه کار

### انتقال Gateway + Node

- برنامه Gateway را (در حالت محلی) اجرا می‌کند و به‌عنوان یک Node به آن متصل می‌شود.
- کنش‌های عامل از طریق `node.invoke` انجام می‌شوند (برای مثال `system.run`، `system.notify`، `canvas.*`).
- فرمان‌های Node شامل `canvas.*`، `camera.snap`، `camera.clip`، `screen.snapshot`، `screen.record`، `computer.act`، `system.run` و `system.notify` هستند.
- Node یک نگاشت `permissions` گزارش می‌کند تا عامل‌ها بتوانند ببینند آیا دسترسی به صفحه‌نمایش، دوربین، میکروفون، گفتار، خودکارسازی یا دسترس‌پذیری موجود است یا خیر.

### سرویس Node + IPC برنامه

- یک سرویس میزبان Node بدون رابط گرافیکی به WebSocket مربوط به Gateway متصل می‌شود.
- درخواست‌های `system.run` از طریق یک سوکت محلی Unix (`ExecApprovalsSocket.swift`) به برنامه macOS هدایت می‌شوند.
- برنامه عملیات اجرا را در زمینه رابط کاربری انجام می‌دهد، در صورت نیاز درخواست تأیید نمایش می‌دهد و خروجی را بازمی‌گرداند.

نمودار (SCI):

```text
عامل -> Gateway -> سرویس Node (WS)
                      |  IPC (UDS + توکن + HMAC + TTL)
                      v
                  برنامه Mac (رابط کاربری + TCC + system.run)
```

### PeekabooBridge (خودکارسازی رابط کاربری)

- ابزار داخلی `computer` عامل از این سوکت استفاده **نمی‌کند**. یک Node جفت‌شده macOS، درخواست `computer.act` را در فرایند برنامه با سرویس‌های تعبیه‌شده Peekaboo برآورده می‌کند.
- خودکارسازی رابط کاربری از یک سوکت UNIX جداگانه (`~/Library/Application Support/OpenClaw/<socket>`) و پروتکل JSON مربوط به PeekabooBridge استفاده می‌کند.
- ترتیب ترجیح میزبان (سمت کلاینت): Peekaboo.app -> Claude.app -> OpenClaw.app -> اجرای محلی.
- امنیت: میزبان‌های پل به TeamID موجود در فهرست مجاز نیاز دارند (`PeekabooBridgeHostCoordinator` همراه برنامه، یک تیم ثابت به‌علاوه تیم امضاکننده خود برنامه را در فهرست مجاز قرار می‌دهد)؛ یک راه گریز با UID یکسان و مختص DEBUG، توسط `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` محافظت می‌شود (قرارداد Peekaboo).
- برای جزئیات، [کاربرد PeekabooBridge](/fa/platforms/mac/peekaboo) را ببینید.

## جریان‌های عملیاتی

- راه‌اندازی مجدد/ساخت مجدد: `scripts/restart-mac.sh` نمونه‌های موجود را متوقف می‌کند، با Swift دوباره می‌سازد، مجدداً بسته‌بندی می‌کند و از نو راه‌اندازی می‌کند. این فرمان یک هویت امضای موجود را به‌طور خودکار شناسایی می‌کند و اگر موردی یافت نشود، به `--no-sign` برمی‌گردد؛ برای الزامی‌کردن امضا، `--sign` را ارسال کنید (اگر کلیدی موجود نباشد، شکست می‌خورد) یا برای اجبار مسیر بدون امضا، `--no-sign` را ارسال کنید. متغیر `SIGN_IDENTITY` تنظیم‌شده در محیط، در مسیر امضاشده حذف می‌شود تا تشخیص خودکار هویت در خود `scripts/codesign-mac-app.sh` گواهی را انتخاب کند.
- نمونه واحد: برنامه برای یافتن شناسه بسته تکراری، `NSWorkspace.runningApplications` را بررسی می‌کند و اگر بیش از یک نمونه پیدا شود، خارج می‌شود (`isDuplicateInstance()` در `MenuBar.swift`).

## نکات مقاوم‌سازی

- ترجیحاً برای همه سطوح ممتاز، تطابق TeamID الزامی باشد.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (فقط DEBUG) ممکن است برای توسعه محلی به فراخوان‌های دارای UID یکسان اجازه دهد.
- تمام ارتباطات صرفاً محلی باقی می‌مانند؛ هیچ سوکت شبکه‌ای در معرض دسترسی قرار نمی‌گیرد.
- درخواست‌های TCC فقط از بسته برنامه GUI منشأ می‌گیرند؛ شناسه بسته امضاشده را در ساخت‌های مجدد ثابت نگه دارید.
- مقاوم‌سازی سوکت تأییدیه‌های اجرا: حالت فایل `0600`، توکن مشترک، بررسی UID همتا (`getpeereid`)، چالش/پاسخ HMAC-SHA256 و یک TTL کوتاه برای درخواست‌ها.

## مرتبط

- [برنامه macOS](/fa/platforms/macos)
- [جریان IPC در macOS (تأییدیه‌های اجرا)](/fa/tools/exec-approvals-advanced#macos-ipc-flow)
