---
read_when:
    - میزبانی PeekabooBridge در OpenClaw.app
    - یکپارچه‌سازی Peekaboo از طریق Swift Package Manager
    - تغییر پروتکل/مسیرهای PeekabooBridge
    - تصمیم‌گیری میان PeekabooBridge، Codex Computer Use و cua-driver MCP
summary: یکپارچه‌سازی PeekabooBridge برای خودکارسازی رابط کاربری macOS
title: پل Peekaboo
x-i18n:
    generated_at: "2026-07-27T15:48:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 24d4187b2f5c5f11f44a24e25b350adaa3b068f24dce640ec695d52eb61f8e9a
    source_path: platforms/mac/peekaboo.md
    workflow: 16
---

OpenClaw می‌تواند **PeekabooBridge** را به‌عنوان یک واسط محلی و آگاه از مجوزها برای خودکارسازی رابط کاربری میزبانی کند (`PeekabooBridgeHostCoordinator`، با پشتیبانی بستهٔ Swift به نام `steipete/Peekaboo`). این کار به CLI ‏`peekaboo` اجازه می‌دهد با استفادهٔ مجدد از مجوزهای TCC برنامهٔ macOS، خودکارسازی رابط کاربری را انجام دهد.

## این چیست (و چیست نیست)

- **میزبان**: OpenClaw.app می‌تواند به‌عنوان میزبان PeekabooBridge عمل کند.
- **کلاینت**: CLI ‏`peekaboo` (هیچ سطح مجزای `openclaw ui ...` وجود ندارد).
- **رابط کاربری**: هم‌پوشانی‌های بصری در Peekaboo.app باقی می‌مانند؛ OpenClaw یک میزبان واسط سبک است.

## ارتباط با دیگر مسیرهای کنترل دسکتاپ

OpenClaw چهار مسیر کنترل دسکتاپ دارد که عمداً از یکدیگر جدا نگه داشته می‌شوند:

- **میزبان PeekabooBridge**: OpenClaw.app سوکت محلی PeekabooBridge را میزبانی می‌کند. CLI ‏`peekaboo` کلاینت است و برای گرفتن اسکرین‌شات، کلیک‌کردن، کار با منوها و گفت‌وگوها، انجام عملیات Dock و مدیریت پنجره‌ها از مجوزهای macOS متعلق به OpenClaw.app استفاده می‌کند.
- **استفادهٔ عامل‌محور از رایانه (`computer.act`)**: ابزار داخلی `computer` عامل Gateway با استفاده از `screen.snapshot` اسکرین‌شات می‌گیرد و از طریق فرمان خطرناک Node به نام `computer.act` اشاره‌گر و صفحه‌کلید را کنترل می‌کند. یک Node در macOS، ‏`computer.act` را درون همان فرایند و با استفاده از سرویس‌های خودکارسازی تعبیه‌شدهٔ Peekaboo که این پل ارائه می‌کند، به‌همراه عملیات ابتدایی و محدود CoreGraphics اجرا می‌کند؛ بدون اینکه از سوکت PeekabooBridge یا CLI ‏`peekaboo` عبور کند. [استفاده از رایانه](/fa/nodes/computer-use) را ببینید.
- **استفاده از رایانه با Codex**: Plugin همراه `codex`، ‏Plugin ‏MCP متعلق به Codex با نام `computer-use` ‏(`extensions/codex/src/app-server/computer-use.ts`) را بررسی می‌کند و می‌تواند آن را نصب کند؛ سپس در نوبت‌های حالت Codex، کنترل فراخوانی‌های بومی ابزار کنترل دسکتاپ را به Codex می‌سپارد. OpenClaw این عملیات را از طریق PeekabooBridge واسطه‌گری نمی‌کند.
- **MCP مستقیم `cua-driver`**: OpenClaw می‌تواند سرور بالادستی `cua-driver mcp` متعلق به TryCua را به‌عنوان یک سرور عادی MCP ثبت کند و طرح‌واره‌ها و گردش‌کار pid/پنجره/نمایهٔ عنصر متعلق به خود درایور CUA را در اختیار عامل‌ها قرار دهد، بدون اینکه از بازار Codex یا سوکت PeekabooBridge مسیریابی شود.

برای سطح گستردهٔ خودکارسازی macOS از طریق میزبان پل آگاه از مجوز OpenClaw.app، از Peekaboo استفاده کنید. وقتی عامل Gateway باید دسکتاپ را از طریق یک فرمان یکنواخت Node به نام `computer.act` ببیند و کنترل کند که هر مدل بینایی بتواند آن را هدایت کند، از استفادهٔ عامل‌محور از رایانه بهره بگیرید. وقتی یک عامل در حالت Codex باید به Plugin بومی Codex متکی باشد، از استفاده از رایانه با Codex استفاده کنید. برای در دسترس قراردادن درایور CUA برای هر زمان‌اجرای مدیریت‌شده توسط OpenClaw به‌عنوان یک سرور عادی MCP، از `cua-driver mcp` مستقیم استفاده کنید.

## فعال‌کردن پل

در برنامهٔ macOS: **Settings -> Enable Peekaboo Bridge**. این کلید مستلزم روشن‌بودن **Allow Computer Control** است، زیرا هر دو امکان خودکارسازی محلی رابط کاربری را فراهم می‌کنند؛ وقتی Computer Control خاموش باشد، کلید غیرفعال است و میزبان اجرا نمی‌شود. برای هدایت Peekaboo بدون Computer Control، به‌جای آن برنامهٔ Mac خود Peekaboo را به‌عنوان میزبان اجرا کنید.

وقتی فعال باشد (و Computer Control روشن باشد)، OpenClaw یک سرور سوکت محلی UNIX را در `~/Library/Application Support/OpenClaw/<socket-name>` راه‌اندازی می‌کند. اگر غیرفعال باشد، میزبان متوقف می‌شود و `peekaboo` به دیگر میزبان‌های موجود بازمی‌گردد. هماهنگ‌کننده همچنین پیوندهای نمادین سوکت قدیمی (`clawdbot`، ‏`clawdis`، ‏`moltbot` در Application Support) را حفظ می‌کند که برای نصب‌های قدیمی‌تر `peekaboo` به سوکت فعلی اشاره دارند.

## ترتیب کشف کلاینت

کلاینت‌های Peekaboo معمولاً میزبان‌ها را به این ترتیب امتحان می‌کنند:

1. Peekaboo.app (تجربهٔ کاربری کامل)
2. Claude.app (اگر نصب شده باشد)
3. OpenClaw.app (واسط سبک)

برای مشاهدهٔ میزبان فعال و مسیر سوکت در حال استفاده، از `peekaboo bridge status --verbose` استفاده کنید. برای بازنویسی، از این دستور استفاده کنید:

```bash
export PEEKABOO_BRIDGE_SOCKET=/path/to/bridge.sock
```

## امنیت و مجوزها

- پل **امضاهای کد فراخواننده** را اعتبارسنجی می‌کند؛ فهرست مجازی از TeamIDها اعمال می‌شود (TeamID میزبان Peekaboo به‌اضافهٔ TeamID خود برنامهٔ در حال اجرا).
- برای Accessibility، هویت امضاشدهٔ پل/برنامه را به یک زمان‌اجرای عمومی `node` ترجیح دهید. اعطای Accessibility به `node` باعث می‌شود هر بسته‌ای که توسط آن فایل اجرایی Node راه‌اندازی شود، دسترسی خودکارسازی رابط گرافیکی را به ارث ببرد؛ [مجوزهای macOS](/fa/platforms/mac/permissions#accessibility-grants-for-node-and-cli-runtimes) را ببینید.
- مهلت درخواست‌ها پس از 10 ثانیه پایان می‌یابد (`requestTimeoutSec: 10`).
- اگر مجوزهای لازم موجود نباشند، پل به‌جای بازکردن System Settings، پیام خطای روشنی برمی‌گرداند.

## رفتار تصویر لحظه‌ای (خودکارسازی)

تصاویر لحظه‌ای با یک بازهٔ اعتبار 10 دقیقه‌ای و سقف 50 تصویر لحظه‌ای (`InMemorySnapshotManager`) در حافظه ذخیره می‌شوند؛ مصنوعات هنگام پاک‌سازی حذف نمی‌شوند. اگر به نگه‌داری طولانی‌تری نیاز دارید، دوباره از کلاینت ثبت کنید.

## عیب‌یابی

- اگر `peekaboo` پیام "کلاینت پل مجاز نیست" را گزارش می‌کند، مطمئن شوید کلاینت به‌درستی امضا شده است یا میزبان را فقط در حالت **debug** با `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` اجرا کنید.
- اگر هیچ میزبانی پیدا نشد، یکی از برنامه‌های میزبان (Peekaboo.app یا OpenClaw.app) را باز کنید و تأیید کنید که مجوزها اعطا شده‌اند.

## مرتبط

- [برنامهٔ macOS](/fa/platforms/macos)
- [مجوزهای macOS](/fa/platforms/mac/permissions)
