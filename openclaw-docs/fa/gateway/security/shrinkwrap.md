---
read_when:
    - می‌خواهید بدانید npm shrinkwrap در یک انتشار OpenClaw به چه معناست
    - در حال بازبینی فایل‌های قفل بسته، تغییرات وابستگی‌ها یا ریسک زنجیره تأمین هستید
    - پیش از انتشار، در حال اعتبارسنجی بسته‌های npm ریشه یا Plugin هستید
summary: توضیح ساده و فنی دربارهٔ shrinkwrap در npm برای انتشارهای OpenClaw
title: بسته‌بندی وابستگی‌های npm
x-i18n:
    generated_at: "2026-07-27T15:34:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1e6c0d4541da9220d50cde0b9db064e5a91b81d6562cb16ac697de7d4017098
    source_path: gateway/security/shrinkwrap.md
    workflow: 16
---

بررسی‌های منبع OpenClaw از `pnpm-lock.yaml` استفاده می‌کنند. بسته‌های منتشرشده OpenClaw در npm از `npm-shrinkwrap.json`، قفل‌فایل وابستگی قابل‌انتشار npm، استفاده می‌کنند تا نصب بسته‌ها از گراف وابستگی بازبینی‌شده هنگام انتشار استفاده کند.

## چرا اهمیت دارد

Shrinkwrap رسیدی برای درخت وابستگی‌ای است که همراه یک بسته npm عرضه می‌شود: به npm می‌گوید دقیقاً کدام نسخه‌های تعدی‌شونده را نصب کند.

| فایل                  | محل اهمیت         | معنای آن                     |
| --------------------- | ------------------------ | --------------------------------- |
| `pnpm-lock.yaml`      | بررسی منبع OpenClaw | گراف وابستگی نگه‌دارندگان       |
| `npm-shrinkwrap.json` | بسته منتشرشده npm    | گراف نصب npm برای کاربران       |
| `package-lock.json`   | برنامه‌های محلی npm           | قرارداد انتشار OpenClaw نیست |

برای انتشارهای OpenClaw، این یعنی:

- بسته منتشرشده از npm نمی‌خواهد هنگام نصب یک گراف وابستگی تازه ابداع کند؛
- تغییرات وابستگی قابل‌بازبینی هستند، زیرا در تفاوت قفل‌فایل ثبت می‌شوند؛
- اعتبارسنجی انتشار همان گرافی را آزمایش می‌کند که کاربران نصب خواهند کرد؛
- غافلگیری‌های مربوط به اندازه بسته یا وابستگی‌های بومی پیش از انتشار آشکار می‌شوند.

Shrinkwrap محیط ایزوله نیست. به‌خودی‌خود یک وابستگی را ایمن نمی‌کند و جایگزین جداسازی میزبان، `openclaw security audit`، منشأ بسته یا آزمون‌های دود نصب نمی‌شود.

OpenClaw یک Gateway، میزبان Plugin، مسیریاب مدل و محیط اجرای عامل است؛ بنابراین نصب پیش‌فرض بر زمان راه‌اندازی، فضای دیسک مصرفی، دانلود بسته‌های بومی و میزان مواجهه با زنجیره تأمین اثر می‌گذارد. Shrinkwrap مرزی پایدار برای بازبینی انتشار فراهم می‌کند: بازبین‌ها جابه‌جایی وابستگی‌های تعدی‌شونده را می‌بینند، اعتبارسنج‌ها تغییرات غیرمنتظره قفل‌فایل را رد می‌کنند و بسته‌های Plugin به‌جای اتکا به بسته ریشه، گراف وابستگی قفل‌شده خود را حمل می‌کنند.

## تولید و بررسی

بسته npm ریشه `openclaw`، بسته‌های Plugin در npm که متعلق به OpenClaw هستند (برای مثال `@openclaw/discord`) و بسته‌های قابل‌انتشار فضای کاری مانند [`@openclaw/ai`](/fa/reference/openclaw-ai) هنگام انتشار شامل `npm-shrinkwrap.json` می‌شوند. وابستگی‌های فضای کاری از shrinkwrap ریشه حذف می‌شوند، زیرا همراه بسته ریشه منتشر می‌شوند؛ در عوض، هر بسته قابل‌انتشار فضای کاری درخت تعدی‌شونده خود را تثبیت می‌کند. بسته‌های Plugin مناسب همچنین می‌توانند با `bundledDependencies` صریح منتشر شوند و فایل‌های وابستگی زمان اجرای خود را در tarball افزونه حمل کنند، نه اینکه فقط به تفکیک هنگام نصب متکی باشند.

```bash
# همه بسته‌های مدیریت‌شده با shrinkwrap (ریشه + افزونه‌های قابل‌انتشار)
pnpm deps:shrinkwrap:generate
pnpm deps:shrinkwrap:check

# فقط بسته ریشه
pnpm deps:shrinkwrap:root:generate
pnpm deps:shrinkwrap:root:check

# فقط بسته‌هایی که از مجموعه تغییرات فعلی تأثیر می‌پذیرند
pnpm deps:shrinkwrap:changed:generate
pnpm deps:shrinkwrap:changed:check
```

مولد، قالب قفل قابل‌انتشار npm را تفکیک می‌کند، اما نسخه‌های بسته تولیدشده‌ای را که از قبل در `pnpm-lock.yaml` وجود ندارند رد می‌کند. این کار مرز مربوط به قدمت وابستگی‌های pnpm، بازنویسی‌ها و بازبینی وصله‌ها را دست‌نخورده نگه می‌دارد.

موارد زیر را به‌عنوان موارد حساس امنیتی بازبینی کنید:

- `pnpm-lock.yaml`
- `npm-shrinkwrap.json`
- محتویات وابستگی Plugin همراه بسته
- هر تفاوت `package-lock.json`

اعتبارسنج‌های بسته OpenClaw وجود shrinkwrap را در tarballهای جدید بسته ریشه الزامی می‌دانند و `package-lock.json` را برای بسته‌های منتشرشده رد می‌کنند. مسیر انتشار Plugin در npm، shrinkwrap محلی Plugin را بررسی می‌کند، وابستگی‌های همراهِ محلی بسته را نصب می‌کند و سپس بسته‌بندی یا منتشر می‌کند.

## بررسی یک بسته منتشرشده

بسته ریشه:

```bash
npm pack openclaw@<version> --json --pack-destination /tmp/openclaw-pack
tar -tf /tmp/openclaw-pack/openclaw-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
```

بسته Plugin:

```bash
npm pack @openclaw/discord@<version> --json --pack-destination /tmp/openclaw-plugin-pack
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/npm-shrinkwrap.json$'
tar -tf /tmp/openclaw-plugin-pack/openclaw-discord-<version>.tgz | grep '^package/node_modules/'
```

اطلاعات زمینه‌ای: [npm-shrinkwrap.json](https://docs.npmjs.com/cli/v11/configuring-npm/npm-shrinkwrap-json).
