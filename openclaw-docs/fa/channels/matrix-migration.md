---
read_when:
    - ارتقای یک نصب موجود Matrix
    - مهاجرت تاریخچه رمزگذاری‌شده و وضعیت دستگاه Matrix
summary: OpenClaw چگونه Plugin قبلی Matrix را درجا ارتقا می‌دهد، از جمله محدودیت‌های بازیابی وضعیت رمزگذاری‌شده و مراحل بازیابی دستی.
title: مهاجرت Matrix
x-i18n:
    generated_at: "2026-07-27T13:52:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 475c96914900a5597f37001264bd3d8f69a69dbd0600f2704c2a1be46924fac4
    source_path: channels/matrix-migration.md
    workflow: 16
---

از Plugin عمومی پیشین `matrix` به پیاده‌سازی فعلی ارتقا دهید.

برای بیشتر کاربران، ارتقا بدون تغییرات اساسی انجام می‌شود:

- Plugin همچنان `@openclaw/matrix` باقی می‌ماند
- کانال همچنان `matrix` باقی می‌ماند
- پیکربندی شما همچنان زیر `channels.matrix` باقی می‌ماند
- اعتبارنامه‌های ذخیره‌شده در حافظه نهان به وضعیت مشترک Plugin در `state/openclaw.sqlite` منتقل می‌شوند
- وضعیت زمان اجرا همچنان زیر `~/.openclaw/matrix/` باقی می‌ماند

نیازی نیست کلیدهای پیکربندی را تغییر نام دهید یا Plugin را با نامی جدید دوباره نصب کنید.
بسته ریشه `openclaw` دیگر کد زمان اجرای Matrix یا وابستگی‌های SDK مربوط به Matrix را در خود ندارد. اگر `openclaw channels status` نشان می‌دهد Matrix پیکربندی شده است، اما
Plugin نصب نیست، `openclaw doctor --fix` یا
`openclaw plugins install @openclaw/matrix` را اجرا کنید؛ بسته‌های SDK مربوط به Matrix را
در بسته ریشه OpenClaw نصب نکنید.

## کارهایی که مهاجرت به‌طور خودکار انجام می‌دهد

مهاجرت Matrix هنگام اجرای [`openclaw doctor --fix`](/fa/gateway/doctor) انجام می‌شود. فایل‌های جانبی مبتنی بر فایل در کنار مخزن اختصاصی Matrix، سازوکار جایگزین خود را هنگام شروع کلاینت حفظ می‌کنند، اما واردکردن فایل اعتبارنامه فقط توسط Doctor انجام می‌شود؛ زمان اجرا فقط وضعیت متعارف اعتبارنامه در SQLite را می‌خواند.

مهاجرت Doctor موارد زیر را پوشش می‌دهد:

- واردکردن و تأیید فایل‌های منسوخ‌شده `~/.openclaw/credentials/matrix/credentials*.json` پیش از بایگانی‌کردن آن‌ها
- حفظ همان انتخاب حساب و پیکربندی `channels.matrix`
- واردکردن وضعیت فایل‌های جانبی مبتنی بر فایل (حافظه نهان همگام‌سازی `bot-storage.json`، ‏`recovery-key.json`، ‏`legacy-crypto-migration.json`، تصویرهای لحظه‌ای IndexedDB) به وضعیت SQLite مربوط به Matrix؛ فایل‌های مهاجرت‌یافته با پسوند `.migrated` بایگانی می‌شوند
- استفاده مجدد از کامل‌ترین ریشه موجود ذخیره‌سازی هش توکن برای همان حساب Matrix، ‏homeserver، کاربر و دستگاه، هنگامی که توکن دسترسی بعداً تغییر می‌کند

## ارتقا از نسخه‌های OpenClaw قدیمی‌تر از 2026.4

نسخه‌های موجود در شاخه انتشار 2026.6 نیز چیدمان مسطح و تک‌مخزنی اولیه
Matrix ‏(`~/.openclaw/matrix/bot-storage.json` به‌همراه
`~/.openclaw/matrix/crypto/`) را مهاجرت می‌دادند و بازیابی وضعیت رمزگذاری‌شده را از
مخزن رمزنگاری قدیمی rust آماده می‌کردند. نسخه‌های فعلی دیگر آن مهاجرت را در خود ندارند.

اگر در حال ارتقای نصبی هستید که هنوز از چیدمان مسطح استفاده می‌کند، ابتدا
به یکی از نسخه‌های 2026.6 ارتقا دهید، `openclaw doctor --fix` را اجرا کنید و Gateway را
یک‌بار راه‌اندازی کنید تا مخزن مسطح و هر کلید اتاق قابل‌بازیابی مهاجرت یابد. سپس
به جدیدترین نسخه به‌روزرسانی کنید.

Plugin عمومی پیشین Matrix به‌طور خودکار نسخه پشتیبان از کلیدهای اتاق Matrix ایجاد **نمی‌کرد**. اگر نصب قدیمی شما تاریخچه رمزگذاری‌شده‌ای داشت که فقط به‌صورت محلی موجود بود و هرگز پشتیبان‌گیری نشده بود، ممکن است برخی پیام‌های رمزگذاری‌شده قدیمی پس از ارتقا و فارغ از مسیر مهاجرت همچنان خوانده نشوند.

## روند پیشنهادی ارتقا

1. OpenClaw و Plugin مربوط به Matrix را به روش معمول به‌روزرسانی کنید.
2. اجرا کنید:

   ```bash
   openclaw doctor --fix
   ```

3. Gateway را راه‌اندازی یا بازراه‌اندازی کنید.
4. وضعیت فعلی تأیید و پشتیبان را بررسی کنید:

   ```bash
   openclaw matrix verify status
   openclaw matrix verify backup status
   ```

5. کلید بازیابی حساب Matrix مورد ترمیم را در یک متغیر محیطی مختص همان حساب قرار دهید. برای یک حساب پیش‌فرض، `MATRIX_RECOVERY_KEY` مناسب است. برای چند حساب، برای هر حساب یک متغیر استفاده کنید؛ برای مثال `MATRIX_RECOVERY_KEY_ASSISTANT`، و `--account assistant` را به فرمان اضافه کنید.

6. اگر OpenClaw اعلام کرد که کلید بازیابی لازم است، فرمان مربوط به حساب منطبق را اجرا کنید:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify backup restore --recovery-key-stdin --account assistant
   ```

7. اگر این دستگاه هنوز تأیید نشده است، فرمان مربوط به حساب منطبق را اجرا کنید:

   ```bash
   printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin
   printf '%s\n' "$MATRIX_RECOVERY_KEY_ASSISTANT" | openclaw matrix verify device --recovery-key-stdin --account assistant
   ```

   اگر کلید بازیابی پذیرفته شد و نسخه پشتیبان قابل‌استفاده است، اما `Cross-signing verified`
   همچنان `no` است، خودتأییدی را از یک کلاینت دیگر Matrix تکمیل کنید:

   ```bash
   openclaw matrix verify self
   ```

   درخواست را در یک کلاینت دیگر Matrix بپذیرید، ایموجی‌ها یا اعداد اعشاری را مقایسه کنید
   و فقط در صورت مطابقت، `yes` را وارد کنید. فرمان پیش از اعلام موفقیت، منتظر
   شکل‌گیری اعتماد کامل هویتی Matrix می‌ماند.

8. اگر عمداً تاریخچه قدیمی و غیرقابل‌بازیابی را کنار می‌گذارید و برای پیام‌های آینده یک خط مبنای پشتیبان تازه می‌خواهید، اجرا کنید:

   ```bash
   openclaw matrix verify backup reset --yes
   ```

   فقط زمانی `--rotate-recovery-key` را اضافه کنید که کلید بازیابی قدیمی دیگر نباید نسخه پشتیبان تازه را باز کند.

9. اگر هنوز نسخه پشتیبان کلید در سمت سرور وجود ندارد، برای بازیابی‌های آینده یکی ایجاد کنید:

   ```bash
   openclaw matrix verify bootstrap
   ```

## پیام‌های رایج و معنای آن‌ها

`Failed migrating legacy Matrix client storage: ...`

- معنا: سازوکار جایگزین سمت کلاینت Matrix وضعیت فایل جانبی مبتنی بر فایل را پیدا کرد، اما واردکردن آن به SQLite ناموفق بود. OpenClaw انتقال‌های تکمیل‌شده را برمی‌گرداند و به‌جای شروع بی‌سروصدا با یک مخزن تازه، آن سازوکار جایگزین را متوقف می‌کند.
- اقدام لازم: مجوزهای سامانه فایل یا تداخل‌ها را بررسی کنید، وضعیت قدیمی را دست‌نخورده نگه دارید و پس از رفع خطا دوباره تلاش کنید.

`Matrix is installed from a custom path: ...`

- معنا: Matrix به نصب از یک مسیر ثابت شده است؛ بنابراین به‌روزرسانی‌های شاخه اصلی آن را به‌طور خودکار با بسته پیش‌فرض Matrix جایگزین نمی‌کنند.
- اقدام لازم: هنگامی که می‌خواهید به Plugin پیش‌فرض Matrix بازگردید، با `openclaw plugins install @openclaw/matrix` دوباره نصب کنید.

`Matrix is installed from a custom path that no longer exists: ...`

- معنا: رکورد نصب Plugin شما به یک مسیر محلی اشاره می‌کند که دیگر وجود ندارد.
- اقدام لازم: با `openclaw plugins install @openclaw/matrix` دوباره نصب کنید، یا اگر از یک نسخه دریافت‌شده از مخزن کد اجرا می‌کنید، از `openclaw plugins install ./path/to/local/matrix-plugin` استفاده کنید. `openclaw doctor --fix` نیز می‌تواند ارجاع‌های منسوخ Plugin مربوط به Matrix را برای شما حذف کند.

### پیام‌های بازیابی دستی

`openclaw matrix verify status` و `openclaw matrix verify backup status` هنگامی که نسخه پشتیبان کلید اتاق روی این دستگاه سالم نیست، یک خط `Backup issue:` به‌همراه راهنمای `Next steps:` چاپ می‌کنند:

| مشکل نسخه پشتیبان                                                   | معنا                                               | راه‌حل                                                                                                                                   |
| --------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `no room-key backup exists on the homeserver`                         | چیزی برای بازیابی وجود ندارد                      | `openclaw matrix verify bootstrap` برای ایجاد نسخه پشتیبان کلید اتاق                                                                      |
| `backup decryption key is not loaded on this device`                  | کلید وجود دارد، اما اینجا فعال نیست               | `openclaw matrix verify backup restore`؛ اگر همچنان نمی‌تواند کلید را بارگیری کند، کلید بازیابی را از طریق `--recovery-key-stdin` به ورودی لوله کنید                |
| `backup decryption key could not be loaded from secret storage (...)` | بارگیری مخزن محرمانه ناموفق است یا پشتیبانی نمی‌شود | کلید بازیابی را به ورودی لوله کنید: `printf '%s\n' "$MATRIX_RECOVERY_KEY" \| openclaw matrix verify backup restore --recovery-key-stdin`               |
| `backup key mismatch (...)`                                           | کلید ذخیره‌شده با نسخه پشتیبان فعال سرور مطابقت ندارد | `verify backup restore --recovery-key-stdin` را با کلید نسخه پشتیبان فعال سرور دوباره اجرا کنید، یا برای یک خط مبنای تازه از `verify backup reset --yes` استفاده کنید |
| `backup signature chain is not trusted by this device`                | دستگاه هنوز به زنجیره امضای متقابل اعتماد ندارد  | `verify device --recovery-key-stdin`، سپس اگر اعتماد همچنان ناقص بود، `verify self` را از یک کلاینت تأییدشده دیگر اجرا کنید                        |
| `backup exists but is not active on this device`                      | نسخه پشتیبان سرور موجود است، نشست محلی غیرفعال است | ابتدا دستگاه را تأیید کنید، سپس با `openclaw matrix verify backup status` دوباره بررسی کنید                                                         |
| `backup trust state could not be fully determined`                    | نتیجه عیب‌یابی قطعی نبود                          | `openclaw matrix verify status --verbose`                                                                                                 |

سایر خطاهای بازیابی:

`Matrix recovery key is required`

- معنا: یک مرحله بازیابی را بدون ارائه کلید بازیابی اجرا کرده‌اید، درحالی‌که کلید لازم بوده است.
- اقدام لازم: فرمان را با `--recovery-key-stdin` دوباره اجرا کنید؛ برای مثال `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify device --recovery-key-stdin`.

`Invalid Matrix recovery key: ...`

- معنا: کلید ارائه‌شده قابل تجزیه نبود یا با قالب مورد انتظار مطابقت نداشت.
- اقدام لازم: با کلید بازیابی دقیق موجود در کلاینت Matrix یا خروجی صادرشده کلید بازیابی دوباره تلاش کنید.

`Matrix recovery key was applied, but this device still lacks full Matrix identity trust.`

- معنا: کلید بازیابی، داده‌های قابل‌استفاده نسخه پشتیبان را باز کرد، اما Matrix هنوز اعتماد کامل هویت امضای متقابل را برای این دستگاه برقرار نکرده است. خروجی فرمان را برای `Recovery key accepted`، ‏`Backup usable`، ‏`Cross-signing verified` و `Device verified by owner` بررسی کنید.
- اقدام لازم: `openclaw matrix verify self` را اجرا کنید، درخواست را در یک کلاینت دیگر Matrix بپذیرید، SAS را مقایسه کنید و فقط در صورت مطابقت، `yes` را وارد کنید. فقط زمانی از `printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify bootstrap --recovery-key-stdin --force-reset-cross-signing` استفاده کنید که عمداً می‌خواهید هویت فعلی امضای متقابل را جایگزین کنید.

اگر از دست‌دادن تاریخچه رمزگذاری‌شده قدیمی و غیرقابل‌بازیابی را می‌پذیرید، می‌توانید در عوض
خط مبنای فعلی پشتیبان را با `openclaw matrix verify backup reset --yes` بازنشانی کنید. هنگامی که
مقدار محرمانه ذخیره‌شده پشتیبان خراب است، این بازنشانی مخزن محرمانه را نیز ترمیم می‌کند تا
کلید پشتیبان جدید پس از بازراه‌اندازی به‌درستی بارگیری شود.

## اگر تاریخچه رمزگذاری‌شده همچنان بازنمی‌گردد

این بررسی‌ها را به‌ترتیب اجرا کنید:

```bash
openclaw matrix verify status --verbose
openclaw matrix verify backup status --verbose
printf '%s\n' "$MATRIX_RECOVERY_KEY" | openclaw matrix verify backup restore --recovery-key-stdin --verbose
```

اگر نسخه پشتیبان با موفقیت بازیابی شد، اما تاریخچه برخی اتاق‌های قدیمی همچنان موجود نیست، احتمالاً Plugin پیشین هرگز از آن کلیدهای ازدست‌رفته پشتیبان نگرفته است.

## اگر می‌خواهید برای پیام‌های آینده از نو شروع کنید

اگر از دست‌دادن تاریخچه رمزگذاری‌شده قدیمی و غیرقابل‌بازیابی را می‌پذیرید و فقط یک خط مبنای پشتیبان پاک برای ادامه کار می‌خواهید، این فرمان‌ها را به‌ترتیب اجرا کنید:

```bash
openclaw matrix verify backup reset --yes
openclaw matrix verify backup status --verbose
openclaw matrix verify status
```

اگر پس از آن دستگاه همچنان تأیید نشده است، با مقایسه ایموجی‌های SAS یا کدهای اعشاری در کلاینت Matrix و تأیید مطابقت آن‌ها، فرایند تأیید را تکمیل کنید.

## مرتبط

- [Matrix](/fa/channels/matrix): راه‌اندازی و پیکربندی کانال.
- [قواعد ارسال Matrix](/fa/channels/matrix-push-rules): مسیریابی اعلان‌ها.
- [Doctor](/fa/gateway/doctor): بررسی سلامت و محرک خودکار مهاجرت.
- [راهنمای مهاجرت](/fa/install/migrating): همه مسیرهای مهاجرت (انتقال میان ماشین‌ها، واردکردن میان سامانه‌ها).
- [Pluginها](/fa/tools/plugin): نصب و ثبت Plugin.
