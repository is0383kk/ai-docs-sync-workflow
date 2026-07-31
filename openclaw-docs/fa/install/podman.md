---
read_when:
    - شما یک Gateway کانتینری با Podman به‌جای Docker می‌خواهید
summary: OpenClaw را در یک کانتینر بدون دسترسی root در Podman اجرا کنید
title: Podman
x-i18n:
    generated_at: "2026-07-27T15:22:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2db1f2b0413d7b9e1b2007aaae2da9d07fa44a1b52901d4a6cbc6274e54567f1
    source_path: install/podman.md
    workflow: 16
---

Gateway در OpenClaw را در یک کانتینر Podman بدون دسترسی root اجرا کنید که توسط کاربر فعلی و غیر root شما مدیریت می‌شود.

مدل کار:

- Podman کانتینر Gateway را اجرا می‌کند.
- CLI میزبان شما، یعنی `openclaw`، صفحه کنترل است.
- حالت پایدار به‌طور پیش‌فرض در میزبان و زیر `~/.openclaw` نگهداری می‌شود.
- برای مدیریت روزمره، به‌جای `sudo -u openclaw`، `podman exec` یا یک کاربر سرویس جداگانه، از `openclaw --container <name> ...` استفاده می‌شود.

## پیش‌نیازها

- **Podman** در حالت بدون root
- نصب بودن **CLI در OpenClaw** روی میزبان
- **اختیاری:** اگر می‌خواهید راه‌اندازی خودکار توسط Quadlet مدیریت شود، `systemd --user`
- **اختیاری:** فقط اگر می‌خواهید روی یک میزبان بدون نمایشگر، ماندگاری پس از راه‌اندازی سیستم را با `loginctl enable-linger "$(whoami)"` فعال کنید، `sudo`

## شروع سریع

<Steps>
  <Step title="راه‌اندازی یک‌باره">
    از ریشه مخزن، `./scripts/podman/setup.sh` را اجرا کنید.

    این کار `openclaw:local` را در فضای ذخیره‌سازی Podman بدون root شما می‌سازد (یا در صورت تنظیم بودن، `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` را دریافت می‌کند)، در صورت نبودن `~/.openclaw/openclaw.json` آن را همراه با `gateway.mode: "local"` ایجاد می‌کند و در صورت نبودن `~/.openclaw/.env` آن را همراه با یک `OPENCLAW_GATEWAY_TOKEN` تولیدشده ایجاد می‌کند.

    متغیرهای محیطی اختیاری زمان ساخت:

    | متغیر | اثر |
    | --- | --- |
    | `OPENCLAW_IMAGE` / `OPENCLAW_PODMAN_IMAGE` | به‌جای ساخت `openclaw:local`، از یک ایمیج موجود یا دریافت‌شده استفاده می‌کند |
    | `OPENCLAW_IMAGE_APT_PACKAGES` | هنگام ساخت ایمیج، بسته‌های apt اضافی را نصب می‌کند (همچنین `OPENCLAW_DOCKER_APT_PACKAGES` قدیمی را می‌پذیرد) |
    | `OPENCLAW_IMAGE_PIP_PACKAGES` | هنگام ساخت ایمیج، بسته‌های Python اضافی را نصب می‌کند؛ نسخه‌ها را ثابت کنید و فقط از ایندکس‌های بسته‌ای استفاده کنید که به آن‌ها اعتماد دارید |
    | `OPENCLAW_EXTENSIONS` | Pluginهای انتخاب‌شده و پشتیبانی‌شده را کامپایل/بسته‌بندی می‌کند و وابستگی‌های زمان اجرای آن‌ها را نصب می‌کند |
    | `OPENCLAW_INSTALL_BROWSER` | Chromium و Xvfb را برای خودکارسازی مرورگر از پیش نصب می‌کند (روی `1` تنظیم کنید) |

    برای راه‌اندازی تحت مدیریت Quadlet به‌جای آن (فقط Linux و سرویس‌های کاربری systemd):

    ```bash
    ./scripts/podman/setup.sh --quadlet
    ```

    یا `OPENCLAW_PODMAN_QUADLET=1` را تنظیم کنید.

  </Step>

  <Step title="راه‌اندازی کانتینر Gateway">
    ```bash
    ./scripts/run-openclaw-podman.sh launch
    ```

    کانتینر را با uid/gid کاربر فعلی شما و `--userns=keep-id` راه‌اندازی می‌کند و حالت OpenClaw شما را با bind mount داخل کانتینر متصل می‌کند.

  </Step>

  <Step title="اجرای فرایند آغاز به کار داخل کانتینر">
    ```bash
    ./scripts/run-openclaw-podman.sh launch setup
    ```

    سپس `http://127.0.0.1:18789/` را باز کنید و از توکن موجود در `~/.openclaw/.env` استفاده کنید.

    احراز هویت مدل: هنگام راه‌اندازی از احراز هویت مدیریت‌شده توسط OpenClaw استفاده کنید (کلیدهای API مربوط به Anthropic، یا احراز هویت OAuth مرورگر/کد دستگاه OpenAI Codex برای OpenAI مبتنی بر Codex). راه‌انداز Podman، محل نگهداری اعتبارنامه‌های CLI میزبان مانند `~/.claude` یا `~/.codex` را داخل کانتینر راه‌اندازی یا Gateway متصل نمی‌کند. ورودهای موجود CLI میزبان فقط مسیرهای تسهیل‌کننده روی همان میزبان هستند -- برای نصب‌های کانتینری، احراز هویت ارائه‌دهنده را در حالت متصل‌شده `~/.openclaw` نگه دارید که فرایند راه‌اندازی آن را مدیریت می‌کند.

  </Step>

  <Step title="مدیریت کانتینر در حال اجرا از CLI میزبان">
    ```bash
    export OPENCLAW_CONTAINER=openclaw
    ```

    سپس فرمان‌های عادی `openclaw` به‌طور خودکار داخل همان کانتینر اجرا می‌شوند:

    ```bash
    openclaw dashboard --no-open
    openclaw gateway status --deep   # شامل اسکن سرویس اضافی است
    openclaw doctor
    openclaw channels login
    ```

    در macOS، ماشین Podman ممکن است باعث شود مرورگر از دید Gateway غیرمحلی به نظر برسد. اگر Control UI پس از راه‌اندازی خطاهای احراز هویت دستگاه را گزارش کرد، از راهنمای Tailscale در [Podman و Tailscale](#podman-and-tailscale) استفاده کنید.

  </Step>
</Steps>

راه‌انداز دستی فقط فهرست مجاز کوچکی از کلیدهای مرتبط با Podman را از `~/.openclaw/.env` می‌خواند و متغیرهای محیطی زمان اجرا را به‌صورت صریح به کانتینر می‌فرستد؛ این راه‌انداز کل فایل محیطی را در اختیار Podman قرار نمی‌دهد.

<a id="podman-and-tailscale"></a>

## Podman و Tailscale

برای دسترسی HTTPS یا دسترسی از راه دور مرورگر، مستندات اصلی Tailscale را دنبال کنید.

نکات ویژه Podman:

- میزبان انتشار Podman را روی `127.0.0.1` نگه دارید.
- استفاده از `tailscale serve` مدیریت‌شده توسط میزبان را به `openclaw gateway --tailscale serve` ترجیح دهید.
- در macOS، اگر بافت احراز هویت دستگاه در مرورگر محلی قابل‌اعتماد نیست، به‌جای راهکارهای موقتی تونل محلی از دسترسی Tailscale استفاده کنید.

[Tailscale](/fa/gateway/tailscale) و [Control UI](/fa/web/control-ui) را ببینید.

## Systemd (Quadlet، اختیاری)

اگر `./scripts/podman/setup.sh --quadlet` را اجرا کرده باشید، فرایند راه‌اندازی یک فایل Quadlet در `~/.config/containers/systemd/openclaw.container` نصب می‌کند.

| عملیات | فرمان                                    |
| ------ | ------------------------------------------ |
| شروع  | `systemctl --user start openclaw.service`  |
| توقف   | `systemctl --user stop openclaw.service`   |
| وضعیت | `systemctl --user status openclaw.service` |
| گزارش‌ها   | `journalctl --user -u openclaw.service -f` |

پس از ویرایش فایل Quadlet:

```bash
systemctl --user daemon-reload
systemctl --user restart openclaw.service
```

برای ماندگاری پس از راه‌اندازی سیستم روی میزبان‌های SSH/بدون نمایشگر، قابلیت lingering را برای کاربر فعلی خود فعال کنید:

```bash
sudo loginctl enable-linger "$(whoami)"
```

سرویس Quadlet تولیدشده یک ساختار پیش‌فرض ثابت و سخت‌سازی‌شده را حفظ می‌کند: پورت‌های منتشرشده `127.0.0.1` (Gateway در `18789`، پل در `18790`)، `--bind lan` داخل کانتینر، فضای نام کاربر `keep-id`، `OPENCLAW_NO_RESPAWN=1`، `Restart=on-failure` و `TimeoutStartSec=300`. این سرویس `~/.openclaw/.env` را به‌عنوان `EnvironmentFile` زمان اجرا برای مقادیری مانند `OPENCLAW_GATEWAY_TOKEN` می‌خواند، اما فهرست مجاز بازنویسی‌های ویژه Podman در راه‌انداز دستی را مصرف نمی‌کند. برای پورت‌های انتشار سفارشی، میزبان انتشار یا سایر پرچم‌های اجرای کانتینر، به‌جای آن از راه‌انداز دستی استفاده کنید، یا `~/.config/containers/systemd/openclaw.container` را مستقیماً ویرایش کنید و سپس سرویس را دوباره بارگذاری و راه‌اندازی کنید.

## پیکربندی، محیط و ذخیره‌سازی

- **دایرکتوری پیکربندی:** `~/.openclaw`
- **دایرکتوری فضای کاری:** `~/.openclaw/workspace`
- **فایل توکن:** `~/.openclaw/.env`
- **دستیار راه‌اندازی:** `./scripts/run-openclaw-podman.sh`

اسکریپت راه‌اندازی و Quadlet، حالت میزبان را با bind mount داخل کانتینر متصل می‌کنند: `OPENCLAW_CONFIG_DIR` -> `/home/node/.openclaw`، `OPENCLAW_WORKSPACE_DIR` -> `/home/node/.openclaw/workspace`. این موارد به‌طور پیش‌فرض دایرکتوری‌های میزبان هستند، نه حالت ناشناس کانتینر؛ بنابراین `openclaw.json`، `auth-profiles.json` هر عامل، حالت کانال/ارائه‌دهنده، نشست‌ها و فضای کاری پس از جایگزینی کانتینر باقی می‌مانند. فرایند راه‌اندازی همچنین `gateway.controlUi.allowedOrigins` را برای `127.0.0.1` و `localhost` روی پورت منتشرشده Gateway مقداردهی اولیه می‌کند تا داشبورد محلی با اتصال غیر loopback کانتینر کار کند.

متغیرهای محیطی مفید برای راه‌انداز دستی (این موارد را در `~/.openclaw/.env` ماندگار کنید؛ راه‌انداز پیش از نهایی‌سازی پیش‌فرض‌های کانتینر/ایمیج، آن فایل را می‌خواند):

| متغیر                                        | پیش‌فرض          | اثر                                 |
| ------------------------------------------ | ---------------- | -------------------------------------- |
| `OPENCLAW_PODMAN_CONTAINER`                | `openclaw`       | نام کانتینر                         |
| `OPENCLAW_PODMAN_IMAGE` / `OPENCLAW_IMAGE` | `openclaw:local` | ایمیج مورد استفاده برای اجرا                           |
| `OPENCLAW_PODMAN_GATEWAY_HOST_PORT`        | `18789`          | پورت میزبان نگاشت‌شده به `18789` کانتینر  |
| `OPENCLAW_PODMAN_BRIDGE_HOST_PORT`         | `18790`          | پورت میزبان نگاشت‌شده به `18790` کانتینر  |
| `OPENCLAW_PODMAN_PUBLISH_HOST`             | `127.0.0.1`      | رابط میزبان برای پورت‌های منتشرشده     |
| `OPENCLAW_GATEWAY_BIND`                    | `lan`            | حالت اتصال Gateway داخل کانتینر |
| `OPENCLAW_PODMAN_USERNS`                   | `keep-id`        | `keep-id`، `auto` یا `host`           |

اگر از `OPENCLAW_CONFIG_DIR` یا `OPENCLAW_WORKSPACE_DIR` غیرپیش‌فرض استفاده می‌کنید، همان متغیرها را هم برای `./scripts/podman/setup.sh` و هم برای فرمان‌های بعدی `./scripts/run-openclaw-podman.sh launch` تنظیم کنید -- راه‌انداز محلی مخزن، بازنویسی مسیرهای سفارشی را بین پوسته‌ها ماندگار نمی‌کند.

## ارتقای ایمیج‌ها

پس از ساخت دوباره یا دریافت یک ایمیج جدید، کانتینر یا سرویس Quadlet را دوباره راه‌اندازی کنید.
در نخستین راه‌اندازی یک نسخه جدید OpenClaw، Gateway پیش از اعلام آمادگی، تعمیرات ایمن حالت و
Plugin را اجرا می‌کند.

اگر Gateway به‌جای آماده‌شدن خارج شد، همان ایمیج را یک‌بار با
`openclaw doctor --fix` و با همان حالت/پیکربندی متصل‌شده اجرا کنید، سپس Gateway را
به‌صورت عادی دوباره راه‌اندازی کنید:

```bash
OPENCLAW_CONFIG_DIR="${OPENCLAW_CONFIG_DIR:-$HOME/.openclaw}"
OPENCLAW_WORKSPACE_DIR="${OPENCLAW_WORKSPACE_DIR:-$OPENCLAW_CONFIG_DIR/workspace}"
OPENCLAW_PODMAN_IMAGE="${OPENCLAW_PODMAN_IMAGE:-${OPENCLAW_IMAGE:-openclaw:local}}"

podman run --rm -it \
  --userns=keep-id \
  --user "$(id -u):$(id -g)" \
  -e HOME=/home/node \
  -e NPM_CONFIG_CACHE=/home/node/.openclaw/.npm \
  -v "$OPENCLAW_CONFIG_DIR:/home/node/.openclaw:rw" \
  -v "$OPENCLAW_WORKSPACE_DIR:/home/node/.openclaw/workspace:rw" \
  "$OPENCLAW_PODMAN_IMAGE" \
  openclaw doctor --fix
```

در میزبان‌های SELinux، اگر Podman دسترسی به حالت متصل‌شده را مسدود می‌کند،
`,Z` را به هر دو bind mount اضافه کنید.

## فرمان‌های مفید

- **گزارش‌های کانتینر:** `podman logs -f openclaw`
- **توقف کانتینر:** `podman stop openclaw`
- **حذف کانتینر:** `podman rm -f openclaw`
- **باز کردن نشانی داشبورد از CLI میزبان:** `openclaw dashboard --no-open`
- **سلامت/وضعیت از طریق CLI میزبان:** `openclaw gateway status --deep` (کاوش RPC + اسکن سرویس اضافی)

## عیب‌یابی

- **خطای دسترسی رد شد (EACCES) برای پیکربندی یا فضای کاری:** کانتینر به‌طور پیش‌فرض با `--userns=keep-id` و `--user <your uid>:<your gid>` اجرا می‌شود. مطمئن شوید مسیرهای پیکربندی/فضای کاری میزبان متعلق به کاربر فعلی شما هستند.
- **راه‌اندازی Gateway مسدود شده است (`gateway.mode=local` وجود ندارد):** مطمئن شوید `~/.openclaw/openclaw.json` وجود دارد و `gateway.mode="local"` را تنظیم می‌کند. `scripts/podman/setup.sh` در صورت نبودن آن را ایجاد می‌کند.
- **کانتینر پس از به‌روزرسانی ایمیج دوباره راه‌اندازی می‌شود:** فرمان یک‌باره `openclaw doctor --fix` را در [ارتقای ایمیج‌ها](#upgrading-images) اجرا کنید، سپس Gateway را دوباره راه‌اندازی کنید.
- **فرمان‌های CLI کانتینر به مقصد اشتباه می‌رسند:** از `openclaw --container <name> ...` به‌صورت صریح استفاده کنید، یا `OPENCLAW_CONTAINER=<name>` را در پوسته خود export کنید.
- **`openclaw update` با `--container` ناموفق می‌شود:** مورد انتظار است. ایمیج را دوباره بسازید/دریافت کنید، سپس کانتینر یا سرویس Quadlet را دوباره راه‌اندازی کنید.
- **سرویس Quadlet راه‌اندازی نمی‌شود:** `systemctl --user daemon-reload` و سپس `systemctl --user start openclaw.service` را اجرا کنید. در سیستم‌های بدون نمایشگر ممکن است به `sudo loginctl enable-linger "$(whoami)"` نیز نیاز داشته باشید.
- **SELinux اتصال‌های bind mount را مسدود می‌کند:** رفتار پیش‌فرض اتصال را تغییر ندهید؛ وقتی SELinux در حالت enforcing یا permissive باشد، راه‌انداز در Linux به‌طور خودکار `:Z` را اضافه می‌کند.

## مرتبط

- [Docker](/fa/install/docker)
- [فرایند پس‌زمینه Gateway](/fa/gateway/background-process)
- [عیب‌یابی Gateway](/fa/gateway/troubleshooting)
