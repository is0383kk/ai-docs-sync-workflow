---
read_when:
    - ساخت یا اجرای کنترل کیفیت بصری زنده برای باگ‌های OpenClaw
    - افزودن تأیید قبل و بعد برای یک Pull request
    - افزودن سناریوهای انتقال زنده برای Discord، Slack، WhatsApp یا سرویس‌های دیگر
    - اجرای اثبات متمرکز مرورگر رابط کاربری کنترل برای یک ref کاندیدا
    - اشکال‌زدایی اجرای QA که به اسکرین‌شات، خودکارسازی مرورگر یا دسترسی VNC نیاز دارد
summary: Mantis شواهد بصری سرتاسری را برای مقایسه‌های زندهٔ انتقال و اثبات‌های متمرکز مرورگر که فقط مختص نامزد هستند ثبت می‌کند و سپس مصنوعات را به PRها پیوست می‌کند.
title: Mantis
x-i18n:
    generated_at: "2026-07-27T16:27:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 48a1b306e37aba7e8c67139df61f3680a9aec066361aa196d88c81270337bc1b
    source_path: concepts/mantis.md
    workflow: 16
---

Mantis شواهد بصری CI و یک دیدگاه PR برای رفتار OpenClaw منتشر می‌کند.
سناریوهای زندهٔ انتقال، یک خط مبنای مشخصاً معیوب را با یک ref نامزد مقایسه می‌کنند؛
در عوض، مسیرهای متمرکز مرورگر ممکن است یک نامزد را در برابر یک انتقال شبیه‌سازی‌شدهٔ
قطعی اثبات کنند. Discord نخست با احراز هویت واقعی ربات، کانال‌های guild،
واکنش‌ها، threadها و یک شاهد مرورگری عرضه شد. مسیرهای Slack، Telegram و چت متمرکز Control
UI نیز وجود دارند؛ WhatsApp و Matrix پیاده‌سازی نشده‌اند.

## مالکیت

- OpenClaw (`extensions/qa-lab/src/mantis/*`): زمان‌اجرای سناریو، `pnpm openclaw qa mantis <command>` CLI، طرح‌وارهٔ شواهد.
- آزمایشگاه QA (`extensions/qa-lab/src/live-transports/*`): چارچوب انتقال زنده، ربات‌های راه‌انداز/SUT، نویسنده‌های گزارش/شواهد.
- Crabbox (`openclaw/crabbox`): ماشین‌های Linux آماده، اجاره‌ها، VNC، `crabbox media preview`.
- GitHub Actions (`.github/workflows/mantis-*.yml`): نقاط ورود راه‌دور، نگه‌داری artifactها.
- ClawSweeper: فرمان‌های PR نگه‌دارنده را تجزیه می‌کند، گردش‌کارها را اجرا می‌کند و دیدگاه نهایی PR را ارسال می‌کند.

## فرمان‌های CLI

همهٔ فرمان‌ها `pnpm openclaw qa mantis <command>` هستند که در
`extensions/qa-lab/src/mantis/cli.ts` تعریف شده‌اند. در زمان ساخت/اجرا به `OPENCLAW_ENABLE_PRIVATE_QA_CLI=1`
نیاز دارد (گردش‌کارهای همراه، پیش از ساخت `OPENCLAW_BUILD_PRIVATE_QA=1` و
`OPENCLAW_ENABLE_PRIVATE_QA_CLI=1` را تنظیم می‌کنند).

| فرمان                         | هدف                                                                                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `discord-smoke`                 | تأیید می‌کند که ربات Discord متعلق به Mantis می‌تواند guild/channel را ببیند، پیام ارسال کند و واکنش نشان دهد.                                                                                 |
| `run`                           | یک سناریوی قبل/بعد را در برابر refهای خط مبنا و نامزد اجرا می‌کند (فقط Discord).                                                                           |
| `desktop-browser-smoke`         | یک دسکتاپ Crabbox را اجاره می‌کند/دوباره به‌کار می‌گیرد، مرورگری قابل‌مشاهده باز می‌کند و تصویر صفحه + ویدئو می‌گیرد.                                                                        |
| `slack-desktop-smoke`           | یک دسکتاپ Crabbox را اجاره می‌کند/دوباره به‌کار می‌گیرد، Slack QA را درون آن اجرا می‌کند، Slack Web را باز می‌کند و شواهد می‌گیرد.                                                                  |
| `telegram-desktop-builder`      | یک دسکتاپ Crabbox را اجاره می‌کند/دوباره به‌کار می‌گیرد، Telegram Desktop را نصب می‌کند و در صورت تمایل یک Gateway متعلق به OpenClaw را پیکربندی می‌کند.                                                        |
| `visual-task` / `visual-driver` | ثبت عمومی دسکتاپ Crabbox با ارزیابی‌های اختیاری درک تصویر؛ `visual-driver` نیمهٔ راه‌انداز است که زیر `crabbox record --while` اجرا می‌شود. |

همهٔ فرمان‌ها `--repo-root <path>` و `--output-dir <path>` را می‌پذیرند؛ فرمان‌های Crabbox
همچنین `--crabbox-bin`، `--provider`، `--machine-class`/`--class`،
`--lease-id`، `--idle-timeout`، `--ttl` و `--keep-lease` را می‌پذیرند. پیش‌فرض‌های CLI محلی
برای ارائه‌دهنده/رده، مگر آنکه خلافش ذکر شود، `hetzner`/`beast` هستند؛ گردش‌کارهای CI
معمولاً هر دو را بازنویسی می‌کنند.

### `discord-smoke`

```bash
pnpm openclaw qa mantis discord-smoke \
  --output-dir .artifacts/qa-e2e/mantis/discord-smoke
```

برای دریافت کاربر ربات، guild، کانال‌های guild و کانال هدف، API REST مربوط به Discord
(`https://discord.com/api/v10`) را فراخوانی می‌کند، تأیید می‌کند که کانال
به guild تعلق دارد، سپس (مگر با `--skip-post`) یک پیام ارسال می‌کند و
واکنش `👀` را اضافه می‌کند. `mantis-discord-smoke-summary.json` و
`mantis-discord-smoke-report.md` را می‌نویسد.

ترتیب تفکیک توکن: مقدار `--token-file`، سپس `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
(بازنویسی با `--token-env`) و پس از آن فایلی با نام تعیین‌شده توسط `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN_FILE`
(بازنویسی با `--token-file-env`). شناسه‌های guild/channel از
`OPENCLAW_QA_DISCORD_GUILD_ID` / `OPENCLAW_QA_DISCORD_CHANNEL_ID` می‌آیند (بازنویسی با
`--guild-id` / `--channel-id`) و باید snowflakeهای 17 تا 20 رقمی Discord باشند. برای
جایگزینی شناسه‌ها و نام‌های ربات/guild/channel/پیام
با `<redacted>` در خلاصه و گزارش منتشرشده، `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` را تنظیم کنید.

### `run`

```bash
pnpm openclaw qa mantis run \
  --transport discord \
  --scenario discord-status-reactions-tool-only \
  --baseline origin/main \
  --candidate HEAD \
  --output-dir .artifacts/qa-e2e/mantis/local-discord-status-reactions
```

`--transport` در حال حاضر فقط `discord` را می‌پذیرد. `--scenario` یکی از دو
شناسهٔ داخلی است که هرکدام ref خط مبنا و برچسب‌های مورد انتظار قبل/بعد
خود را دارند (`extensions/qa-lab/src/mantis/run.runtime.ts`):

| سناریو                                   | خط مبنای پیش‌فرض                           | انتظار از خط مبنا                         | انتظار از نامزد            |
| ------------------------------------------ | ------------------------------------------ | ---------------------------------------- | ---------------------------- |
| `discord-status-reactions-tool-only`       | `0bf06e953fdda290799fc9fb9244a8f67fdae593` | `queued-only`                            | `queued -> thinking -> done` |
| `discord-thread-reply-filepath-attachment` | `81349cdc2a9d5143fd0991ed858b739e7d96e05c` | پاسخ thread پیوست `filePath` را حذف می‌کند | پاسخ thread آن را شامل می‌شود     |

پیش‌فرض `--candidate` برابر `HEAD` است. پرچم‌های دیگر: `--credential-source`
(پیش‌فرض `convex`)، `--credential-role` (پیش‌فرض `ci`)، `--provider-mode`
(پیش‌فرض `live-frontier`)، `--fast` (به‌طور پیش‌فرض روشن)، `--skip-install`، `--skip-build`.

اجراکننده برای خط مبنا و نامزد، checkoutهای جداشدهٔ `git worktree` را
زیر `<output-dir>/worktrees/` ایجاد می‌کند، در هرکدام `pnpm install`/`pnpm build` را
اجرا می‌کند (مگر اینکه رد شوند)، سپس
`pnpm openclaw qa discord --scenario <id> --model openai/gpt-5.4 --alt-model openai/gpt-5.4 --allow-failures` را
در برابر هر worktree اجرا می‌کند. هر مسیر `discord-qa-reaction-timelines.json`
را به‌همراه یک جفت `<scenario-id>-timeline.html`/`.png` می‌نویسد؛ اجراکننده این
شواهد را زیر `baseline/`/`candidate/` کپی می‌کند، `comparison.json`،
`mantis-report.md` و `mantis-evidence.json` را در دایرکتوری خروجی می‌نویسد و
اگر مقایسه موفق نباشد، با کد غیرصفر خارج می‌شود (خط مبنا `fail` و نامزد
`pass`).

سناریوی دوم Discord (`discord-thread-reply-filepath-attachment`)
با ربات راه‌انداز یک پیام والد ارسال می‌کند، یک thread واقعی می‌سازد، کنش `message.thread-reply`
مربوط به SUT را با یک `filePath` محلی مخزن فراخوانی می‌کند، سپس برای
پاسخ و نام فایل پیوست، thread را به‌طور دوره‌ای بررسی می‌کند. انتظار دارد پیوستی
با نام `mantis-thread-report.md` وجود داشته باشد.

### `desktop-browser-smoke`

```bash
pnpm openclaw qa mantis desktop-browser-smoke \
  --output-dir .artifacts/qa-e2e/mantis/desktop-browser
```

یک دسکتاپ Crabbox را اجاره می‌کند یا دوباره به‌کار می‌گیرد، درون نشست VNC مرورگری
را با نشانی `--browser-url` (پیش‌فرض `https://openclaw.ai`) یا یک
`--html-file` رندرشده اجرا می‌کند، منتظر می‌ماند، با `scrot` تصویر صفحه می‌گیرد، در صورت تمایل با
`ffmpeg` یک MP4 ضبط می‌کند و `desktop-browser-smoke.png` / `.mp4` / `remote-metadata.json`
را با rsync به `--output-dir` بازمی‌گرداند.

پرچم‌ها:

- `--lease-id <cbx_...>` به‌جای ایجاد دسکتاپ جدید، یک دسکتاپ آماده را دوباره به‌کار می‌گیرد.
- `--browser-profile-dir <remote-path>` یک user-data-dir راه‌دور Chrome را دوباره به‌کار می‌گیرد تا دسکتاپ پایدار بین اجراها واردشده باقی بماند (برای پروفایل نمایشگر بلندمدت Discord Web استفاده می‌شود).
- `--browser-profile-archive-env <name>` پیش از اجرا، آرشیو پروفایل Chrome با قالب base64 از نوع `.tgz` را از آن متغیر محیطی بازیابی می‌کند (پیش‌فرض `OPENCLAW_MANTIS_BROWSER_PROFILE_TGZ_B64`)؛ برای شاهدهای واردشده مانند Discord Web استفاده می‌شود.
- `--video-duration <seconds>` مدت ثبت MP4 را کنترل می‌کند (پیش‌فرض 10s).
- `--keep-lease` (یا `OPENCLAW_MANTIS_KEEP_VM=1`) اجاره‌ای را که این اجرا ایجاد کرده برای بازرسی VNC باز نگه می‌دارد؛ اجراهای ناموفقی که اجاره ایجاد کرده‌اند نیز به‌طور پیش‌فرض آن را نگه می‌دارند.

برای شواهد Discord Web، Mantis از یک حساب نمایشگر اختصاصی استفاده می‌کند، نه توکن
ربات. اوراکل REST متعلق به Discord (از طریق `qa discord`) همچنان مرجع معتبر است؛ وقتی
`OPENCLAW_QA_DISCORD_CAPTURE_UI_METADATA=1` تنظیم شده باشد، سناریو همچنین یک
artifact نشانی Discord Web می‌نویسد و `OPENCLAW_QA_DISCORD_KEEP_THREADS=1`
thread را به‌اندازهٔ کافی باز نگه می‌دارد تا مرورگر آن را باز کند.

گردش‌کار GitHub یک پروفایل نمایشگر پایدار را از طریق
`MANTIS_DISCORD_VIEWER_CHROME_PROFILE_DIR` ترجیح می‌دهد (آرشیو کامل پروفایل ممکن است از
محدودیت اندازهٔ secret در GitHub بزرگ‌تر شود)؛ برای پروفایل‌های کوچک/راه‌انداز می‌تواند به‌جای آن یک
`.tgz` با قالب base64 را از `MANTIS_DISCORD_VIEWER_CHROME_PROFILE_TGZ_B64` بازیابی کند. اگر
هیچ‌یک از منابع پیکربندی نشده باشند، گردش‌کار همچنان تصاویر صفحهٔ قطعی
خط مبنا/نامزد را منتشر می‌کند و ثبت می‌کند که شاهد واردشده
رد شده است.

### `slack-desktop-smoke`

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --output-dir .artifacts/qa-e2e/mantis/slack-desktop \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

یک دسکتاپ Crabbox را اجاره می‌کند یا دوباره به‌کار می‌گیرد، checkout را در VM همگام می‌کند،
`pnpm openclaw qa slack` را درون آن اجرا می‌کند، Slack Web را در مرورگر VNC باز می‌کند،
از دسکتاپ تصویر می‌گیرد و هم artifactهای Slack QA (`slack-qa/`) و هم
تصویر صفحه/ویدئوی VNC را به محیط محلی کپی می‌کند. این تنها شکل Mantis است که در آن
Gateway مربوط به SUT و مرورگر، هر دو در یک VM اجرا می‌شوند.

با `--gateway-setup`، فرمان یک home پایدار و یک‌بارمصرف OpenClaw
در `$HOME/.openclaw-mantis/slack-openclaw` داخل VM ایجاد می‌کند، پیکربندی Slack
Socket Mode را برای کانال هدف وصله می‌کند،
`openclaw gateway run --dev --allow-unconfigured --port 38973` را اجرا می‌کند و
Chrome را در نشست VNC باز نگه می‌دارد؛ حذف `--gateway-setup` در عوض مسیر عادی
Slack QA ربات‌به‌ربات را اجرا می‌کند.

متغیرهای محیطی لازم برای `--credential-source env` (پیش‌فرض محلی `env` است؛
پیش‌فرض نقش `maintainer` است):

- `OPENCLAW_QA_SLACK_CHANNEL_ID`
- `OPENCLAW_QA_SLACK_DRIVER_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_BOT_TOKEN`
- `OPENCLAW_QA_SLACK_SUT_APP_TOKEN`
- `OPENCLAW_LIVE_OPENAI_KEY` برای مسیر مدل راه‌دور (اگر فقط `OPENAI_API_KEY`
  به‌صورت محلی تنظیم شده باشد، Mantis پیش از
  فراخوانی Crabbox آن را در `OPENCLAW_LIVE_OPENAI_KEY` کپی می‌کند)

با `--credential-source convex`، Mantis پیش از ایجاد VM، اطلاعات اعتبارنامهٔ SUT متعلق به Slack را از
مخزن مشترک اجاره می‌کند و شناسهٔ کانال، توکن برنامه و
توکن ربات را به‌عنوان متغیرهای محیطی `OPENCLAW_MANTIS_SLACK_*` به VM ارسال می‌کند، بنابراین گردش‌کارهای GitHub
فقط به secret کارگزار Convex نیاز دارند، نه توکن‌های خام Slack.

پرچم‌های دیگر: `--slack-url <url>` یک نشانی مشخص را باز می‌کند (در غیر این صورت Mantis
`https://app.slack.com/client/<team>/<channel>` را از `auth.test` استخراج می‌کند)؛
`--slack-channel-id <id>` کانال فهرست مجاز Gateway را تنظیم می‌کند؛
`OPENCLAW_MANTIS_SLACK_BROWSER_PROFILE_DIR` پروفایل پایدار Chrome را
داخل VM کنترل می‌کند (پیش‌فرض `$HOME/.config/openclaw-mantis/slack-chrome-profile`)؛
`--approval-checkpoints` سناریوهای بومی تأیید Slack
(`slack-approval-exec-native`، `slack-approval-plugin-native`) را اجرا می‌کند و
به‌جای راه‌اندازی Gateway، تصاویر صفحهٔ نقاط وارسی در انتظار/حل‌شده را رندر می‌کند (با
`--gateway-setup` ناسازگار است)؛ `--hydrate-mode source|prehydrated`،
`--provider-mode`، `--model`، `--alt-model` و `--fast` به مسیر زندهٔ
Slack منتقل می‌شوند.

تصاویر صفحهٔ نقاط وارسی تأیید از پیام API متعلق به Slack که
سناریو مشاهده کرده رندر می‌شوند، نه از رابط زندهٔ Slack؛ `slack-desktop-smoke.png` تنها زمانی
اثبات خود Slack Web است که پروفایل مرورگر اجاره از قبل وارد شده باشد.

### `telegram-desktop-builder`

```bash
pnpm openclaw qa mantis telegram-desktop-builder \
  --credential-source convex \
  --credential-role maintainer \
  --keep-lease
```

یک دسکتاپ Crabbox را اجاره می‌کند یا دوباره به‌کار می‌گیرد، Telegram Desktop بومی Linux را نصب می‌کند،
در صورت تمایل آرشیو نشست کاربر را بازیابی می‌کند، OpenClaw را با
توکن ربات SUT اجاره‌شدهٔ Telegram پیکربندی می‌کند،
`openclaw gateway run --dev --allow-unconfigured --port 38974` را اجرا می‌کند، یک
پیام آمادگی ربات راه‌انداز را به گروه خصوصی اجاره‌شده ارسال می‌کند و سپس یک
تصویر صفحه و MP4 ثبت می‌کند. توکن ربات فقط OpenClaw را پیکربندی می‌کند؛ هرگز
Telegram Desktop را وارد حساب نمی‌کند. نمایشگر دسکتاپ یک نشست کاربری جداگانهٔ Telegram است
که از `--telegram-profile-archive-env <name>` بازیابی می‌شود یا به‌صورت دستی
از طریق VNC وارد می‌شود و با `--keep-lease` زنده نگه داشته می‌شود.

پرچم‌ها: `--lease-id <cbx_...>` اجرا را در برابر VMای که از قبل وارد
Telegram Desktop شده تکرار می‌کند؛ `--telegram-profile-archive-env <name>` پیش از اجرا یک آرشیو پروفایل
`.tgz` با قالب base64 را بازیابی می‌کند؛ `--telegram-profile-dir <remote-path>`
دایرکتوری پروفایل راه‌دور را تنظیم می‌کند (پیش‌فرض `$HOME/.local/share/TelegramDesktop`)؛
`--no-gateway-setup` فقط Telegram Desktop را نصب و باز می‌کند؛
پیش‌فرض `--credential-source`/`--credential-role` برابر `convex`/`maintainer` است.

## مانیفست شواهد

هر سناریویی که در یک PR منتشر می‌شود، در کنار گزارش خود
`mantis-evidence.json` را می‌نویسد:

```json
{
  "schemaVersion": 1,
  "id": "discord-status-reactions",
  "title": "تضمین کیفیت واکنش‌های وضعیت Discord در Mantis",
  "summary": "خلاصهٔ اصلی خوانا برای انسان جهت درج در نظر PR.",
  "scenario": "discord-status-reactions-tool-only",
  "comparison": {
    "baseline": { "sha": "...", "status": "fail", "expected": "فقط در صف" },
    "candidate": { "sha": "...", "status": "pass", "expected": "در صف -> در حال فکر -> انجام‌شده" },
    "pass": true
  },
  "artifacts": [
    {
      "kind": "خط زمانی",
      "lane": "خط مبنا",
      "label": "خط مبنای فقط در صف",
      "path": "baseline/timeline.png",
      "targetPath": "baseline.png",
      "alt": "خط زمانی خط مبنای Discord",
      "width": 420
    }
  ]
}
```

`path` آرتیفکت نسبت به دایرکتوری مانیفست است؛ `targetPath`
نسبت به پیشوند پیکربندی‌شدهٔ آرتیفکت R2/S3 است. `scripts/mantis/publish-pr-evidence.mjs`
پیمایش مسیر را رد می‌کند و در صورت نبودن فایل، ورودی‌های دارای `"required": false`
را نادیده می‌گیرد.

انواع آرتیفکت: `timeline` (اسکرین‌شات قطعی قبل/بعد)،
`desktopScreenshot` (اسکرین‌شات VNC/مرورگر)، `motionPreview` (GIF متحرک درون‌خطی
از ضبط)، `motionClip` (MP4 برش‌خورده بر اساس حرکت)، `fullVideo` (ضبط
کامل)، `metadata` (فایل جانبی JSON/لاگ)، `report` (گزارش Markdown).

چیدمان آرتیفکت‌های روی دیسک یک اجرا:

```text
.artifacts/qa-e2e/mantis/<run-id>/
  mantis-report.md
  mantis-evidence.json
  baseline/
  candidate/
  comparison.json
```

اسکرین‌شات‌ها مدرک‌اند، نه اطلاعات محرمانه، اما همچنان به رعایت اصول حذف اطلاعات حساس نیاز دارند:
ممکن است نام کانال‌های خصوصی، نام‌های کاربری یا محتوای پیام‌ها دیده شود. برای بارگذاری عمومی
آرتیفکت‌ها `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` را تنظیم کنید؛ این گزینه
به‌طور پیش‌فرض در گردش‌کارهای GitHub مربوط به Discord/Slack/Telegram فعال است.

## خودکارسازی GitHub

`scripts/mantis/publish-pr-evidence.mjs` انتشاردهندهٔ قابل‌استفادهٔ مجدد است. گردش‌کارها
آن را با مانیفست، PR مقصد، ریشهٔ مقصد آرتیفکت، نشانگر نظر،
URL آرتیفکت، URL اجرا و منبع درخواست فراخوانی می‌کنند. این ابزار آرتیفکت‌های اعلام‌شده را در
باکت R2 مربوط به Mantis بارگذاری می‌کند، یک نظر PR با اولویت نمایش خلاصه و شامل
تصاویر/پیش‌نمایش‌های درون‌خطی و ویدئوهای پیوندشده می‌سازد، سپس نظر موجود دارای نشانگر را
به‌روزرسانی می‌کند یا نظر جدیدی می‌سازد. متغیرهای محیطی الزامی:

- `MANTIS_ARTIFACT_R2_ACCESS_KEY_ID`
- `MANTIS_ARTIFACT_R2_SECRET_ACCESS_KEY`
- `MANTIS_ARTIFACT_R2_BUCKET` (گردش‌کارها `openclaw-crabbox-artifacts` را تنظیم می‌کنند)
- `MANTIS_ARTIFACT_R2_ENDPOINT`
- `MANTIS_ARTIFACT_R2_REGION` (گردش‌کارها `auto` را تنظیم می‌کنند)
- `MANTIS_ARTIFACT_R2_PUBLIC_BASE_URL` (گردش‌کارها `https://artifacts.openclaw.ai` را تنظیم می‌کنند)

نظرها از طریق GitHub App مربوط به Mantis ‏(`MANTIS_GITHUB_APP_ID` /
`MANTIS_GITHUB_APP_PRIVATE_KEY`) ارسال می‌شوند، نه `github-actions[bot]`، و یک
نظر نشانگر مخفی به‌عنوان کلید درج یا به‌روزرسانی استفاده می‌شود.

| گردش‌کار                          | محرک                                                                                    | کاری که انجام می‌دهد                                                                                                                                                                                                                                                                                                     |
| --------------------------------- | ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Mantis Discord Smoke`            | اجرای دستی                                                                            | `discord-smoke` را روی یک ref انتخاب‌شده اجرا می‌کند.                                                                                                                                                                                                                                                                       |
| `Mantis Discord Status Reactions` | نظر PR یا اجرای دستی                                                              | worktreeهای جداگانهٔ خط مبنا/نامزد را می‌سازد، `discord-status-reactions-tool-only` را روی هرکدام اجرا می‌کند، خط زمانی هر مسیر را در مرورگر دسکتاپ Crabbox رندر می‌کند، با `crabbox media preview` پیش‌نمایش‌های GIF/MP4 برش‌خورده بر اساس حرکت می‌سازد، آرتیفکت‌ها را بارگذاری می‌کند و مدارک درون‌خطی را در PR می‌فرستد.                                 |
| `Mantis Scenario`                 | اجرای دستی                                                                            | توزیع‌کنندهٔ عمومی: `scenario_id` ‏(`discord-status-reactions-tool-only`، `discord-thread-reply-filepath-attachment`، `slack-desktop-smoke`، `telegram-live`، `telegram-desktop-proof`، `web-ui-chat-proof`)، `baseline_ref`، `candidate_ref` و `pr_number` را می‌گیرد و به گردش‌کار سناریوی متناظر ارسال می‌کند. |
| `Mantis Slack Desktop Smoke`      | اجرای دستی                                                                            | یک دسکتاپ لینوکس Crabbox اجاره می‌کند (پیش‌فرض `aws`، با امکان انتخاب `hetzner`)، `slack-desktop-smoke --gateway-setup` را روی نامزد اجرا می‌کند، دسکتاپ را ضبط می‌کند، یک پیش‌نمایش حرکتی می‌سازد، آرتیفکت‌ها را بارگذاری می‌کند و در صورت ارائهٔ شمارهٔ PR، مدارک را در PR می‌فرستد.                                                      |
| `Mantis Telegram Live`            | نظر PR یا اجرای دستی                                                              | مسیر تضمین کیفیت زندهٔ Telegram مبتنی بر API ربات (`openclaw qa telegram`) را اجرا می‌کند، `mantis-evidence.json` را از خلاصهٔ تضمین کیفیت می‌نویسد، HTML مدارک حذف‌شده از اطلاعات حساس را از طریق مرورگر دسکتاپ Crabbox رندر می‌کند، یک GIF حرکتی می‌سازد و مدارک را در PR می‌فرستد. برای این مسیر، ورود به Telegram Web لازم نیست.                               |
| `Mantis Telegram Desktop Proof`   | برچسب PR نگه‌دارنده (`mantis: telegram-visible-proof`) به‌همراه نظر PR، یا اجرای دستی | مدرک عامل‌محور و بومی قبل/بعد در Telegram Desktop. ‏PR، ‏refهای خط مبنا/نامزد و دستورالعمل‌های نگه‌دارنده را به Codex می‌سپارد؛ Codex مسیر اثبات Telegram Desktop با کاربر واقعی در Crabbox را برای هر دو ref اجرا می‌کند و یک جدول مدارک ۲ ستونی در PR می‌فرستد.                                                              |
| `Mantis Web UI Chat Proof`        | نظر PR یا اجرای دستی                                                              | اثبات متمرکز گفت‌وگوی Playwright در رابط کنترل OpenClaw را روی نامزد اجرا می‌کند، تأیید می‌کند که مرورگر از طریق Gateway شبیه‌سازی‌شده ارسال می‌کند، آرتیفکت‌های اسکرین‌شات/ویدئو را ثبت می‌کند و مدارک را در PR می‌فرستد. این مسیر فقط اثبات گفت‌وگوی وب است، نه اثبات WinUI/برنامهٔ بومی یا هر نوع اثبات بصری دلخواه.                           |

`Mantis Discord Status Reactions` و `Mantis Telegram Live` هر دو
`baseline_ref`/`candidate_ref` (یا `baseline=`/`candidate=` در یک نظر PR)
را می‌پذیرند و پیش از اجرا با اعتبارنامه‌های حاوی اطلاعات محرمانه، اعتبارسنجی می‌کنند که SHA حل‌شده
یا یکی از اجداد `origin/main`، یا یک تگ انتشار (`v*`) یا سرشاخهٔ یک PR باز باشد.

محرک‌های نظر، از یک PR با دسترسی نوشتن/نگه‌داری/مدیریت:

```text
@openclaw-mantis discord status reactions
@openclaw-mantis discord status reactions baseline=origin/main candidate=HEAD
@openclaw-mantis telegram
@openclaw-mantis telegram scenario=telegram-status-command
@openclaw-mantis telegram scenarios=telegram-status-command,channel-canary
@openclaw-mantis web ui chat
@openclaw-mantis web-ui-chat candidate=HEAD
```

محرک‌های نظر Telegram به‌طور پیش‌فرض SHA سرشاخهٔ PR را به‌عنوان نامزد و
`telegram-status-command` را به‌عنوان سناریو در نظر می‌گیرند؛ آن‌ها `provider=aws|hetzner` و
`lease=<cbx_...>` را برای هدف‌گیری یک ارائه‌دهندهٔ مشخص Crabbox یا یک
دسکتاپ ازپیش‌گرم‌شده می‌پذیرند. `Mantis Telegram Desktop Proof` فقط زمانی به نظر PR پاسخ می‌دهد که
PR از قبل برچسب `mantis: telegram-visible-proof` را داشته باشد.

محرک‌های نظر گفت‌وگوی رابط وب به‌طور پیش‌فرض SHA سرشاخهٔ PR را به‌عنوان نامزد در نظر می‌گیرند. آن‌ها
اثبات گفت‌وگوی رابط کنترل با Gateway شبیه‌سازی‌شده را اجرا و آرتیفکت‌های مرورگر را منتشر می‌کنند؛ برای
دیگر صفحات وب و سطوح برنامهٔ بومی از اثبات معمول Playwright/مرورگر، اسکرین‌شات‌های نگه‌دارنده، Crabbox یا
آرتیفکت‌های محلی استفاده کنید.

ClawSweeper همچنین می‌تواند یک سناریو را مستقیماً اجرا کند:

```text
@clawsweeper mantis discord discord-status-reactions-tool-only
```

## ماشین‌ها و اطلاعات محرمانه

پیش‌فرض‌های Crabbox در CLI محلی `--provider hetzner --class beast` هستند؛ با
`--provider`، `--class`/`--machine-class` یا
`OPENCLAW_MANTIS_CRABBOX_PROVIDER` / `OPENCLAW_MANTIS_CRABBOX_CLASS` آن‌ها را بازنویسی کنید. گردش‌کارهای
GitHub معمولاً هر دو را بازنویسی می‌کنند (برای مثال `--class standard` و ورودی انتخاب
ارائه‌دهندهٔ `aws`/`hetzner` در گردش‌کار Slack). اگر یک ارائه‌دهنده بیش‌ازحد
کند یا دردسترس‌نباشد، آن را پشت همان رابط Crabbox اضافه کنید، نه اینکه یک مسیر جایگزین را
به‌صورت ثابت در کد بنویسید.

خط مبنای ماشین مجازی: لینوکس با Chrome/Chromium دارای قابلیت دسکتاپ، دسترسی CDP، ‏VNC/
noVNC، ‏Node 22.22.3+، ‏24.15+ یا 25.9+ و pnpm، یک checkout از OpenClaw و
دسترسی خروجی به بستر انتقال مقصد، GitHub، ارائه‌دهندگان مدل و
کارگزار اعتبارنامه.

نام اعتبارنامه‌ها و محیط‌های استفاده‌شده در فرمان‌ها و گردش‌کارهای Mantis:

- `OPENCLAW_QA_DISCORD_MANTIS_BOT_TOKEN`
- `OPENCLAW_QA_DISCORD_GUILD_ID`
- `OPENCLAW_QA_DISCORD_CHANNEL_ID`
- `qa mantis run --credential-source env` محلی همچنین به
  `OPENCLAW_QA_DISCORD_DRIVER_BOT_TOKEN`، `OPENCLAW_QA_DISCORD_SUT_BOT_TOKEN`
  و `OPENCLAW_QA_DISCORD_SUT_APPLICATION_ID` نیاز دارد. گردش‌کارهای GitHub معمولاً به‌جای توکن‌های خام
  ربات Discord از `--credential-source convex` و اعتبارنامه‌های کارگزار زیر استفاده می‌کنند.
- `OPENCLAW_QA_REDACT_PUBLIC_METADATA=1` برای بارگذاری عمومی آرتیفکت‌ها
- `OPENCLAW_QA_CONVEX_SITE_URL`، `OPENCLAW_QA_CONVEX_SECRET_CI`
- `OPENAI_API_KEY` (یا `OPENCLAW_MANTIS_AGENT_OPENAI_API_KEY` مخصوص
  اثبات Telegram Desktop)
- `CRABBOX_COORDINATOR` / `CRABBOX_COORDINATOR_TOKEN` (گردش‌کارها همچنین
  `OPENCLAW_QA_MANTIS_CRABBOX_COORDINATOR` / `_TOKEN` را به‌عنوان مسیر جایگزین می‌پذیرند و
  پیش از فراخوانی Crabbox آن‌ها را به نام‌های ساده نگاشت می‌کنند)
- `CRABBOX_ACCESS_CLIENT_ID`، `CRABBOX_ACCESS_CLIENT_SECRET`
- `MANTIS_GITHUB_APP_ID`، `MANTIS_GITHUB_APP_PRIVATE_KEY`

اجراکنندهٔ Mantis هرگز نباید توکن‌های ربات Discord/Slack/Telegram،
کلیدهای API ارائه‌دهنده، کوکی‌های مرورگر، محتوای پروفایل احراز هویت، گذرواژه‌های VNC یا
بارهای خام اعتبارنامه را چاپ کند. اگر توکنی در یک مسئله، PR، گفت‌وگو یا لاگ افشا شد،
پس از ذخیره‌شدن اطلاعات محرمانهٔ جایگزین، آن را تعویض کنید.

## نتایج اجرا

سناریوهای انتقال قبل/بعد میان این نتایج تمایز قائل می‌شوند تا یک
محیط ناپایدار به‌عنوان پس‌رفت محصول برداشت نشود:

- **بازتولید اشکال**: خط مبنا به همان روشی که سناریو انتظار دارد ناموفق شد.
- **خرابی چارچوب آزمون**: راه‌اندازی محیط، اعتبارنامه‌ها، API بستر انتقال، مرورگر
  یا ارائه‌دهنده پیش از معنادارشدن معیار ارزیابی ناموفق شد.

اثبات مرورگر فقط برای نامزد گزارش می‌کند که آیا نامزد از Gateway شبیه‌سازی‌شده و
ادعاهای قابل‌مشاهدهٔ رابط کاربری عبور کرده است یا نه؛ این اثبات ادعای بازتولید خط مبنا ندارد.

## افزودن یک سناریو

سناریوهای انتقال زنده برای هر بستر انتقال با TypeScript تعریف می‌شوند (برای
شکل قبل/بعد Discord، به `MANTIS_SCENARIO_CONFIGS` در `extensions/qa-lab/src/mantis/run.runtime.ts`
مراجعه کنید)، نه با یک قالب فایل اعلانی مستقل.
هر سناریو به این موارد نیاز دارد: شناسه و عنوان، بستر انتقال، اعتبارنامه‌های الزامی، سیاست
ref خط مبنا، سیاست ref نامزد، وصلهٔ پیکربندی OpenClaw، مراحل راه‌اندازی/محرک،
معیار ارزیابی مورد انتظار خط مبنا و نامزد، اهداف ثبت بصری، بودجهٔ
مهلت زمانی و مراحل پاک‌سازی.

اثبات متمرکز مرورگر فقط برای نامزد می‌تواند از یک آزمون قطعی E2E و
گردش‌کار اختصاصی استفاده کند. دامنهٔ آن را صریح نگه دارید، پیش از
اجرا ref نامزد را اعتبارسنجی کنید، انتشار متکی بر اطلاعات محرمانه را ایزوله کنید و همان قرارداد
مانیفست مدارک را تولید کنید.

معیارهای ارزیابی کوچک و نوع‌دار را بر بررسی‌های بصری ترجیح دهید: وضعیت واکنش Discord یا
ارجاعات پیام، وضعیت API واکنش/`ts` رشتهٔ Slack، شناسه‌های پیام ایمیل
و سرایندها. زمانی از اسکرین‌شات‌های مرورگر استفاده کنید که رابط کاربری تنها مشاهده‌پذیر قابل‌اعتماد باشد،
و در صورت وجود معیار ارزیابی مبتنی بر API پلتفرم، بررسی‌های بصری را مکمل آن نگه دارید.

پس از Discord، Slack و Telegram، همین ساختار اجراکننده به WhatsApp
(ورود با QR، شناسایی مجدد، تحویل، رسانه، واکنش‌ها) و Matrix
(اتاق‌های رمزنگاری‌شده، روابط رشته/پاسخ، ادامه پس از راه‌اندازی مجدد) گسترش می‌یابد؛ هنوز
هیچ‌کدام پیاده‌سازی نشده‌اند.

## پرسش‌های باز

- هنگام استفادهٔ مجدد از بات موجود Mantis، کدام بات Discord باید راه‌انداز و کدام‌یک SUT باشد؟
- GitHub باید مصنوعات Mantis مربوط به PRها را چه مدت نگه دارد؟
- ClawSweeper چه زمانی باید به‌جای انتظار برای فرمان نگه‌دارنده، به‌طور خودکار یک سناریوی Mantis را پیشنهاد کند؟
- آیا پیش از بارگذاری اسکرین‌شات‌ها برای PRهای عمومی، باید اطلاعات حساس آن‌ها پوشانده شود یا برش داده شوند؟
