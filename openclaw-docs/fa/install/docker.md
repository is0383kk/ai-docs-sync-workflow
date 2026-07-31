---
read_when:
    - به‌جای نصب‌های محلی، یک Gateway کانتینری می‌خواهید
    - در حال اعتبارسنجی جریان Docker هستید
summary: راه‌اندازی و آغاز به کار اختیاری مبتنی بر Docker برای OpenClaw
title: داکر
x-i18n:
    generated_at: "2026-07-27T14:10:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1784bd49f6847db75633840a4d5a8e49205200728bd2e9d59b646a446e508d6
    source_path: install/docker.md
    workflow: 16
---

Docker **اختیاری** است. از آن برای یک محیط Gateway ایزوله و یک‌بارمصرف یا میزبانی بدون نصب‌های محلی استفاده کنید. اگر از قبل روی دستگاه خودتان توسعه می‌دهید، به‌جای آن از جریان نصب معمول استفاده کنید.

وقتی `agents.defaults.sandbox` فعال باشد، بک‌اند پیش‌فرض سندباکس از Docker استفاده می‌کند؛ اما سندباکس به‌طور پیش‌فرض غیرفعال است و نیازی ندارد خود Gateway در Docker اجرا شود. بک‌اندهای سندباکس SSH و OpenShell نیز در دسترس‌اند؛ به [سندباکس‌سازی](/fa/gateway/sandboxing) مراجعه کنید.

از چند کاربر میزبانی می‌کنید؟ برای مدل یک سلول به‌ازای هر مستأجر، به [میزبانی چندمستأجری](/fa/gateway/multi-tenant-hosting) مراجعه کنید.

## پیش‌نیازها

- Docker Desktop (یا Docker Engine) + Docker Compose v2
- حداقل 2 GB حافظهٔ RAM برای ساخت ایمیج (`pnpm install` ممکن است در میزبان‌های 1 GB با خروج 137 به‌دلیل OOM متوقف شود)
- فضای دیسک کافی برای ایمیج‌ها و گزارش‌ها
- در یک VPS/میزبان عمومی، [سخت‌سازی امنیتی برای دسترسی شبکه](/fa/gateway/security)، به‌ویژه زنجیرهٔ فایروال `DOCKER-USER` در Docker را بررسی کنید

## Gateway کانتینری

<Steps>
  <Step title="ساخت ایمیج">
    از ریشهٔ مخزن:

    ```bash
    ./scripts/docker/setup.sh
    ```

    این دستور ایمیج Gateway را به‌صورت محلی با نام `openclaw:local` می‌سازد. برای استفاده از یک ایمیج ازپیش‌ساخته‌شده:

    ```bash
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    ایمیج‌های ازپیش‌ساخته‌شده ابتدا در [GitHub Container Registry](https://github.com/openclaw/openclaw/pkgs/container/openclaw) منتشر می‌شوند. GHCR رجیستری اصلی برای خودکارسازی انتشار، استقرارهای سنجاق‌شده و بررسی‌های منشأ است. همان نسخهٔ انتشار، یک آینهٔ Docker Hub نیز در `openclaw/openclaw` منتشر می‌کند:

    ```bash
    export OPENCLAW_IMAGE="openclaw/openclaw:latest"
    ./scripts/docker/setup.sh
    ```

    از `ghcr.io/openclaw/openclaw` یا `openclaw/openclaw` استفاده کنید و از آینه‌های غیررسمی بپرهیزید، زیرا زمان‌بندی انتشار یا خط‌مشی نگهداری OpenClaw را ندارند. برچسب‌های مختص نسخه شامل انتشارهایی مانند `2026.2.26` و پیش‌انتشارهایی مانند `2026.2.26-beta.1` هستند. انتشارهای پایدار `latest` و `main` را جابه‌جا می‌کنند؛ انتشارهای Gateway مربوط به ماه انتهایی فقط `extended-stable` را جابه‌جا می‌کنند. گونه‌ها شامل `slim`، `main-slim`، `extended-stable-slim`، `latest-browser`، `main-browser` و `extended-stable-browser` هستند. ایمیج‌های پیش‌فرض Pluginهای `codex` و `diagnostics-otel` را در خود دارند. یک گونهٔ `-browser` نیز همراه با Chromium ازپیش‌تعبیه‌شده ارائه می‌شود که برای ابزار [مرورگر سندباکس‌شده](/fa/gateway/sandboxing#sandboxed-browser) بدون نیاز به نصب Playwright در نخستین اجرا مفید است.

  </Step>

  <Step title="اجرای مجدد در محیط بدون اتصال">
    در میزبان‌های آفلاین، ابتدا ایمیج را منتقل و بارگذاری کنید:

    ```bash
    docker load -i openclaw-image.tar
    export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
    ./scripts/docker/setup.sh --offline
    ```

    `--offline` بررسی می‌کند که `OPENCLAW_IMAGE` از قبل به‌صورت محلی وجود داشته باشد، واکشی‌ها/ساخت‌های ضمنی Compose را غیرفعال می‌کند و سپس جریان معمول را اجرا می‌کند: همگام‌سازی `.env`، اصلاح مجوزها، راه‌اندازی اولیه، همگام‌سازی پیکربندی Gateway و راه‌اندازی Compose.

    اگر `OPENCLAW_SANDBOX=1`، راه‌اندازی آفلاین همچنین ایمیج‌های سندباکس پیش‌فرض پیکربندی‌شده و مختص هر عامل را روی دیمون پشت `OPENCLAW_DOCKER_SOCKET` بررسی می‌کند؛ از جمله برچسب قرارداد مرورگر روی ایمیج‌های مرورگر مبتنی بر Docker. اگر ایمیج موردنیازی وجود نداشته باشد یا قدیمی باشد، راه‌اندازی بدون تغییر پیکربندی سندباکس خارج می‌شود، نه اینکه موفقیتی معیوب گزارش کند.

  </Step>

  <Step title="تکمیل راه‌اندازی اولیه">
    اسکریپت راه‌اندازی، راه‌اندازی اولیه را به‌طور خودکار اجرا می‌کند:

    - کلیدهای API ارائه‌دهنده را درخواست می‌کند
    - یک توکن Gateway تولید می‌کند و آن را در `.env` می‌نویسد
    - دایرکتوری کلید محرمانهٔ نمایهٔ احراز هویت را ایجاد می‌کند
    - Gateway را از طریق Docker Compose راه‌اندازی می‌کند

    راه‌اندازی اولیهٔ پیش از شروع و نوشتن پیکربندی مستقیماً از طریق `openclaw-gateway` (با `--no-deps --entrypoint node`) اجرا می‌شوند، زیرا `openclaw-cli` فضای نام شبکهٔ Gateway را به اشتراک می‌گذارد و تنها پس از ایجاد کانتینر Gateway کار می‌کند.

  </Step>

  <Step title="باز کردن رابط کنترل">
    `http://127.0.0.1:18789/` را باز کنید و توکن نوشته‌شده در `.env` را در تنظیمات جای‌گذاری کنید. اگر کانتینر را به احراز هویت با گذرواژه تغییر داده‌اید، به‌جای آن از همان گذرواژه استفاده کنید.

    دوباره به نشانی نیاز دارید؟

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

  </Step>

  <Step title="پیکربندی کانال‌ها (اختیاری)">
    ```bash
    # WhatsApp (کد QR)
    docker compose run --rm openclaw-cli channels login

    # Telegram
    docker compose run --rm openclaw-cli channels add --channel telegram --token "<token>"

    # Discord
    docker compose run --rm openclaw-cli channels add --channel discord --token "<token>"
    ```

    مستندات: [WhatsApp](/fa/channels/whatsapp)، [Telegram](/fa/channels/telegram)، [Discord](/fa/channels/discord)

  </Step>
</Steps>

### جریان دستی

```bash
BUILD_GIT_COMMIT="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
docker build \
  --build-arg "GIT_COMMIT=${BUILD_GIT_COMMIT}" \
  --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
  -t openclaw:local -f Dockerfile .
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js onboard --mode local --no-install-daemon
docker compose run --rm --no-deps --entrypoint node openclaw-gateway \
  dist/index.js config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"},{"path":"gateway.controlUi.allowedOrigins","value":["http://localhost:18789","http://127.0.0.1:18789"]}]'
docker compose up -d openclaw-gateway
```

زمینهٔ Docker، `.git` را مستثنا می‌کند. همان‌طور که در بالا نشان داده شد، هویت منبع را به‌صورت آرگومان‌های ساخت ارسال کنید تا صفحهٔ «درباره» ایمیج، کامیت بررسی‌شده و یک برچسب زمانی ساخت را گزارش کند. `scripts/docker/setup.sh` هر دو مقدار را به‌طور خودکار تعیین و ارسال می‌کند.

<Note>
`docker compose` را از ریشهٔ مخزن اجرا کنید. اگر `OPENCLAW_EXTRA_MOUNTS` یا `OPENCLAW_HOME_VOLUME` را فعال کرده‌اید، اسکریپت راه‌اندازی `docker-compose.extra.yml` را می‌نویسد؛ آن را پس از هر `docker-compose.override.yml` که خودتان نگهداری می‌کنید اضافه کنید؛ برای مثال `-f docker-compose.yml -f docker-compose.override.yml -f docker-compose.extra.yml`.
</Note>

### ارتقای ایمیج‌های کانتینر

هنگامی که ایمیج OpenClaw را جایگزین می‌کنید اما همان وضعیت/پیکربندی متصل‌شده را نگه می‌دارید، Gateway جدید پیش از آماده‌شدن، مهاجرت‌های ارتقای ایمن برای راه‌اندازی و همگرایی Pluginها را اجرا می‌کند. ارتقاهای معمول ایمیج نباید به اجرای جداگانهٔ `openclaw doctor --fix` نیاز داشته باشند.

اگر راه‌اندازی نتواند این اصلاحات را با ایمنی کامل کند، Gateway به‌جای گزارش وضعیت سالم خارج می‌شود. با یک خط‌مشی راه‌اندازی مجدد، Docker، Podman یا Kubernetes ممکن است کانتینر Gateway را در حال راه‌اندازی مجدد نشان دهند. حجم وضعیت متصل‌شده را حفظ کنید، سپس همان ایمیج را یک‌بار با `openclaw doctor --fix` به‌عنوان فرمان کانتینر و با استفاده از همان اتصال‌های وضعیت/پیکربندی مورد استفادهٔ Gateway اجرا کنید:

```bash
docker run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
podman run --rm -v <openclaw-state>:/home/node/.openclaw <image> openclaw doctor --fix
```

پس از پایان کار doctor، کانتینر Gateway را با فرمان پیش‌فرضش دوباره راه‌اندازی کنید. در Kubernetes، همان فرمان را در یک Job یک‌بارمصرف یا پاد اشکال‌زدایی متصل به همان PVC اجرا کنید، سپس Deployment یا StatefulSet را دوباره راه‌اندازی کنید.

### متغیرهای محیطی

متغیرهای اختیاری پذیرفته‌شده توسط `scripts/docker/setup.sh` (و برای کانتینر Gateway، مستقیماً توسط `docker-compose.yml`):

| متغیر                                           | هدف                                                                                                              |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_IMAGE`                                | استفاده از یک ایمیج راه‌دور به‌جای ساخت محلی                                                                    |
| `OPENCLAW_IMAGE_APT_PACKAGES`                   | نصب بسته‌های اضافی apt هنگام ساخت (جداشده با فاصله). نام مستعار قدیمی: `OPENCLAW_DOCKER_APT_PACKAGES`           |
| `OPENCLAW_IMAGE_PIP_PACKAGES`                   | نصب بسته‌های اضافی Python هنگام ساخت (جداشده با فاصله)                                                          |
| `OPENCLAW_EXTENSIONS`                           | کامپایل/بسته‌بندی Pluginهای منتخب پشتیبانی‌شده و نصب وابستگی‌های زمان اجرای آن‌ها (شناسه‌های جداشده با ویرگول یا فاصله) |
| `OPENCLAW_DOCKER_BUILD_NODE_OPTIONS`            | بازنویسی گزینه‌های Node برای ساخت محلی از منبع (پیش‌فرض `--max-old-space-size=8192`)                                |
| `OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB` | بازنویسی حافظهٔ heap مربوط به tsdown برای ساخت محلی از منبع، برحسب MB                                           |
| `OPENCLAW_DOCKER_BUILD_SKIP_DTS`                | صرف‌نظر از خروجی اعلان‌ها هنگام ساخت ایمیج‌های محلی صرفاً برای زمان اجرا (پیش‌فرض `1`)                                      |
| `OPENCLAW_INSTALL_BROWSER`                      | تعبیهٔ Chromium + Xvfb در ایمیج هنگام ساخت                                                                       |
| `OPENCLAW_EXTRA_MOUNTS`                         | اتصال‌های bind اضافی میزبان (`source:target[:opts]` جداشده با ویرگول)                                                   |
| `OPENCLAW_HOME_VOLUME`                          | ماندگارکردن `/home/node` در یک حجم نام‌گذاری‌شدهٔ Docker                                                                     |
| `OPENCLAW_SANDBOX`                              | انتخاب راه‌اندازی اولیهٔ سندباکس (`1`، `true`، `yes`، `on`)                                                            |
| `OPENCLAW_SKIP_ONBOARDING`                      | صرف‌نظر از مرحلهٔ تعاملی راه‌اندازی اولیه (`1`، `true`، `yes`، `on`)                                                   |
| `OPENCLAW_DOCKER_SOCKET`                        | بازنویسی مسیر سوکت Docker                                                                                         |
| `OPENCLAW_DISABLE_BONJOUR`                      | اجبار تبلیغ Bonjour/mDNS به حالت روشن (`0`) یا خاموش (`1`)؛ به [Bonjour / mDNS](#bonjour--mdns) مراجعه کنید                        |
| `OPENCLAW_DISABLE_BUNDLED_SOURCE_OVERLAYS`      | غیرفعال‌کردن هم‌پوشانی‌های اتصال bind منبع Pluginهای همراه                                                       |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                   | نقطهٔ پایانی گردآورندهٔ مشترک OTLP/HTTP برای خروجی OpenTelemetry                                                 |
| `OTEL_EXPORTER_OTLP_*_ENDPOINT`                 | نقاط پایانی OTLP مختص سیگنال برای ردگیری‌ها، سنجه‌ها یا گزارش‌ها                                                |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                   | بازنویسی پروتکل OTLP. در حال حاضر فقط `http/protobuf` پشتیبانی می‌شود                                                   |
| `OTEL_SERVICE_NAME`                             | نام سرویس مورد استفاده برای منابع OpenTelemetry                                                                  |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                 | انتخاب جدیدترین ویژگی‌های معنایی آزمایشی GenAI                                                                   |
| `OPENCLAW_OTEL_PRELOADED`                       | جلوگیری از راه‌اندازی دومین SDK مربوط به OpenTelemetry هنگامی که یکی از قبل بارگذاری شده است                    |

ایمیج رسمی شامل Homebrew نیست. هنگام راه‌اندازی اولیه، OpenClaw نصب‌کننده‌های وابستگی Skills مختص brew را در یک کانتینر Linux بدون `brew` پنهان می‌کند؛ این وابستگی‌ها را از طریق یک ایمیج سفارشی فراهم کنید یا به‌صورت دستی نصب کنید. برای وابستگی‌های بسته‌بندی‌شدهٔ Debian از `OPENCLAW_IMAGE_APT_PACKAGES` و برای وابستگی‌های Python از `OPENCLAW_IMAGE_PIP_PACKAGES` استفاده کنید (`python3 -m pip install --break-system-packages` را هنگام ساخت اجرا می‌کند؛ بنابراین نسخه‌ها را سنجاق کنید و فقط از ایندکس‌هایی استفاده کنید که به آن‌ها اعتماد دارید).

اگر Docker خطاهای `ResourceExhausted` یا `cannot allocate memory` را گزارش کرد، یا هنگام `tsdown` متوقف شد، محدودیت حافظهٔ سازندهٔ Docker را افزایش دهید یا با heapهای صریح کوچک‌تر دوباره تلاش کنید:

```bash
OPENCLAW_DOCKER_BUILD_NODE_OPTIONS=--max-old-space-size=4096 OPENCLAW_DOCKER_BUILD_TSDOWN_MAX_OLD_SPACE_MB=4096
```

### ایمیج‌های ساخته‌شده از منبع با Pluginهای منتخب

`OPENCLAW_EXTENSIONS` شناسه‌های مانیفست Plugin را از checkout منبع انتخاب می‌کند؛
نام‌های موجود دایرکتوری منبع نیز، در صورت متفاوت‌بودن، پذیرفته می‌شوند. فرایند ساخت Docker
انتخاب را یک‌بار به دایرکتوری‌های منبع نگاشت می‌کند، وابستگی‌های production
را نصب می‌کند و، وقتی Plugin انتخاب‌شده جداگانه با
`openclaw.build.bundledDist: false` منتشر شده باشد، runtime آن را در dist همراه‌شده
ریشه کامپایل می‌کند. این بسته‌بندی مختص Docker قرارداد artifact مربوط به npm یا ClawHub
این Plugin را تغییر نمی‌دهد. شناسه‌های ناشناخته، نامعتبر یا مبهم باعث شکست ساخت image می‌شوند.
شناسه‌های شناخته‌شده‌ای که فقط برای وابستگی/منبع هستند، staging فعلی منبع و وابستگی
خود را بدون دریافت ورودی dist کامپایل‌شده در ریشه حفظ می‌کنند. یک Plugin انتخاب‌شده با
ورودی‌های ساخت یکپارچه باید با موفقیت کامپایل شود؛ منبع و خروجی runtime مربوط به Pluginهای
خارجی انتخاب‌نشده حذف می‌شوند.

برای نمونه، این فرمان‌ها imageهای مستقل و چندمعماریِ جداگانه‌ای از
Gatewayهای FakeCo برای ClickClack، Slack و Microsoft Teams می‌سازند. ClawRouter از قبل
بخشی از runtime ریشه OpenClaw است، بنابراین image مربوط به ClickClack فقط
`clickclack` را انتخاب می‌کند. آرگومان صریحاً خالی مرورگر باعث می‌شود image پیش‌فرض
فاقد Chromium بماند:

```bash
SOURCE_SHA="$(git rev-parse HEAD)"
BUILD_TIMESTAMP="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
REGISTRY="registry.example.com/fakeco"

build_gateway_image() {
  gateway="$1"
  selected_plugin="$2"
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    --build-arg "GIT_COMMIT=${SOURCE_SHA}" \
    --build-arg "OPENCLAW_BUILD_TIMESTAMP=${BUILD_TIMESTAMP}" \
    --build-arg "OPENCLAW_EXTENSIONS=${selected_plugin}" \
    --build-arg OPENCLAW_INSTALL_BROWSER= \
    --provenance=mode=max \
    --sbom=true \
    --tag "${REGISTRY}/openclaw-${gateway}:${SOURCE_SHA}" \
    --push \
    .
}

build_gateway_image clickclack clickclack
build_gateway_image slack slack
build_gateway_image teams msteams
```

برای یک ساخت محلی native منفرد، از `--platform linux/arm64 --load` یا `--platform linux/amd64 --load` استفاده کنید.
خروجی چندپلتفرمی و SBOM/provenance پیوست‌شده
به یک registry یا خروجی دیگری از Buildx نیاز دارند که attestationها را حفظ کند. پس از
push، مانیفست را بررسی کنید و به‌جای tag تغییرپذیر source-SHA،
digest تغییرناپذیر را مستقر کنید:

```bash
docker buildx imagetools inspect \
  "${REGISTRY}/openclaw-clickclack:${SOURCE_SHA}"
# استقرار: registry.example.com/fakeco/openclaw-clickclack@sha256:<manifest-digest>
```

این imageها برای Gatewayهای مستقل مبتنی بر OCI و کاربران عمومی Docker هستند.
Gatewayهای مدیریت‌شده با Crabhelm از آن‌ها استفاده نمی‌کنند: آن مسیر تحویل،
یک archive مجزای appliance برای x86_64 می‌سازد که حاوی tarball مربوط به npm از OpenClaw است و
digestهای Node، archive و مانیفست را ثابت می‌کند. آن appliance را مستقلاً
از همان منبع نهایی‌شده OpenClaw بسازید.

برای آزمایش منبع Plugin همراه‌شده در برابر یک image بسته‌بندی‌شده، یک دایرکتوری منبع Plugin را روی مسیر منبع بسته‌بندی‌شده آن mount کنید؛ برای مثال `OPENCLAW_EXTRA_MOUNTS=/path/to/fork/extensions/synology-chat:/app/extensions/synology-chat:ro`. این کار bundle کامپایل‌شده متناظر `/app/dist/extensions/synology-chat` را برای همان شناسه Plugin بازنویسی می‌کند.

### مشاهده‌پذیری

ارسال OpenTelemetry از کانتینر Gateway به collector مربوط به OTLP شما خروجی است؛ به هیچ پورت منتشرشده Docker نیاز ندارد. برای گنجاندن exporter همراه‌شده در یک image ساخته‌شده به‌صورت محلی:

```bash
export OPENCLAW_EXTENSIONS="diagnostics-otel"
export OTEL_EXPORTER_OTLP_ENDPOINT="http://otel-collector:4318"
export OTEL_SERVICE_NAME="openclaw-gateway"
./scripts/docker/setup.sh
```

imageهای رسمی ازپیش‌ساخته‌شده از قبل `diagnostics-otel` را همراه دارند؛ فقط اگر آن را حذف کرده‌اید، خودتان `clawhub:@openclaw/diagnostics-otel` را نصب کنید. برای فعال‌کردن ارسال، Plugin مربوط به `diagnostics-otel` را در پیکربندی مجاز و فعال کنید، سپس `diagnostics.otel.enabled=true` را تنظیم کنید (نمونه کامل را در [ارسال OpenTelemetry](/fa/gateway/opentelemetry) ببینید). هدرهای احراز هویت collector از طریق `diagnostics.otel.headers` ارائه می‌شوند، نه متغیرهای محیطی Docker.

معیارهای Prometheus از همان پورت ازقبل‌منتشرشده Gateway استفاده می‌کنند. `clawhub:@openclaw/diagnostics-prometheus` را نصب و Plugin مربوط به `diagnostics-prometheus` را فعال کنید، سپس scrape کنید:

```text
http://<gateway-host>:18789/api/diagnostics/prometheus
```

این route با احراز هویت Gateway محافظت می‌شود؛ یک پورت عمومی جداگانه `/metrics` یا مسیر reverse proxy بدون احراز هویت در معرض دسترس قرار ندهید. [معیارهای Prometheus](/fa/gateway/prometheus) را ببینید.

### بررسی‌های سلامت

endpointهای probe کانتینر (بدون نیاز به احراز هویت):

```bash
curl -fsS http://127.0.0.1:18789/healthz   # زنده‌بودن
curl -fsS http://127.0.0.1:18789/readyz     # آمادگی
```

`HEALTHCHECK` داخلی image، `/healthz` را ping می‌کند؛ شکست‌های تکراری کانتینر را `unhealthy` علامت‌گذاری می‌کنند تا orchestratorها بتوانند آن را راه‌اندازی مجدد یا جایگزین کنند.

snapshot عمیق سلامت با احراز هویت:

```bash
docker compose exec openclaw-gateway node dist/index.js health --token "$OPENCLAW_GATEWAY_TOKEN"
```

### LAN در برابر loopback

`scripts/docker/setup.sh` به‌طور پیش‌فرض `OPENCLAW_GATEWAY_BIND=lan` را تنظیم می‌کند تا `http://127.0.0.1:18789` روی میزبان با انتشار پورت Docker کار کند.

- `lan` (پیش‌فرض): مرورگر میزبان و CLI میزبان می‌توانند به پورت منتشرشده Gateway دسترسی پیدا کنند.
- `loopback`: فقط پردازه‌های داخل namespace شبکه کانتینر می‌توانند مستقیماً به Gateway دسترسی پیدا کنند.

<Note>
از مقادیر حالت bind در `gateway.bind` (`lan` / `loopback` / `custom` / `tailnet` / `auto`) استفاده کنید، نه aliasهای میزبان مانند `0.0.0.0` یا `127.0.0.1`.
</Note>

### ارائه‌دهندگان محلی میزبان

درون کانتینر، `127.0.0.1` خود کانتینر است، نه میزبان. برای ارائه‌دهندگانی که روی میزبان اجرا می‌شوند از `host.docker.internal` استفاده کنید:

| ارائه‌دهنده | URL پیش‌فرض میزبان | URL راه‌اندازی Docker |
| --------- | ------------------------ | ----------------------------------- |
| LM Studio | `http://127.0.0.1:1234`  | `http://host.docker.internal:1234`  |
| Ollama    | `http://127.0.0.1:11434` | `http://host.docker.internal:11434` |

راه‌اندازی همراه‌شده از آن URLها به‌عنوان پیش‌فرض‌های onboarding برای LM Studio/Ollama استفاده می‌کند و `docker-compose.yml`، مقدار `host.docker.internal` را در Docker Engine روی Linux به Gateway میزبان نگاشت می‌کند (Docker Desktop همین alias را در macOS/Windows فراهم می‌کند). سرویس‌های میزبان باید روی آدرسی گوش دهند که Docker بتواند به آن دسترسی پیدا کند:

```bash
lms server start --port 1234 --bind 0.0.0.0
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

از فایل Compose خودتان یا `docker run` استفاده می‌کنید؟ همان نگاشت را خودتان اضافه کنید؛ برای مثال `--add-host=host.docker.internal:host-gateway`.

### backend مربوط به Claude CLI در Docker

image رسمی، Claude Code را از پیش نصب نمی‌کند. آن را در کاربر `node` کانتینر نصب کنید و وارد حساب شوید، سپس home کانتینر را ماندگار کنید تا ارتقای image فایل اجرایی یا وضعیت احراز هویت را پاک نکند.

برای نصب جدید، پیش از اجرای راه‌اندازی یک volume ماندگار `/home/node` را فعال کنید:

```bash
export OPENCLAW_IMAGE="ghcr.io/openclaw/openclaw:latest"
export OPENCLAW_HOME_VOLUME="openclaw_home"
./scripts/docker/setup.sh
```

برای نصب موجود، ابتدا stack را متوقف و مقادیر فعلی `.env` را دوباره بارگذاری کنید — اسکریپت راه‌اندازی همیشه `.env` را بر اساس shell فعلی و مقادیر پیش‌فرض بازنویسی می‌کند و فایل را به‌تنهایی نمی‌خواند:

```bash
set -a
. ./.env
set +a
export OPENCLAW_HOME_VOLUME="${OPENCLAW_HOME_VOLUME:-openclaw_home}"
./scripts/docker/setup.sh
```

اگر `.env` حاوی مقادیری است که shell شما نمی‌تواند source کند، ابتدا مواردی را که به آن‌ها متکی هستید به‌صورت دستی دوباره export کنید (`OPENCLAW_IMAGE`، پورت‌ها، حالت bind، مسیرهای سفارشی، `OPENCLAW_EXTRA_MOUNTS`، sandbox، ردکردن onboarding). overlay تولیدشده، volume مربوط به home را برای هر دو `openclaw-gateway` و `openclaw-cli` mount می‌کند؛ فرمان‌های باقی‌مانده را با همان overlay اجرا کنید (و اگر از `docker-compose.override.yml` استفاده می‌کنید، ابتدا آن را نیز اضافه کنید):

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint sh openclaw-cli -lc \
  'curl -fsSL https://claude.ai/install.sh | bash'
```

نصب‌کننده native، `claude` را در `/home/node/.local/bin/claude` می‌نویسد. image
مربوط به OpenClaw شامل `/home/node/.local/bin` در `PATH` است، بنابراین Plugin همراه‌شده
Anthropic بدون بازنویسی پیکربندی adapter آن را resolve می‌کند.

با همان home ماندگار وارد شوید و بررسی کنید:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth login
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  --entrypoint /home/node/.local/bin/claude openclaw-cli auth status --text
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models auth login \
  --provider anthropic --method cli --set-default
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli models list --provider anthropic
```

سپس از backend همراه‌شده `claude-cli` استفاده کنید:

```bash
docker compose -f docker-compose.yml -f docker-compose.extra.yml run --rm \
  openclaw-cli agent \
  --agent main \
  --model claude-cli/claude-sonnet-4-6 \
  --message "از Docker Claude CLI سلام کن"
```

`OPENCLAW_HOME_VOLUME` نصب native را در `/home/node/.local/bin` و `/home/node/.local/share/claude`، و همچنین تنظیمات/احراز هویت Claude Code را در `/home/node/.claude` و `/home/node/.claude.json` ماندگار می‌کند. ماندگارکردن فقط `/home/node/.openclaw` کافی نیست؛ اگر به‌جای volume مربوط به home از `OPENCLAW_EXTRA_MOUNTS` استفاده می‌کنید، همه آن مسیرهای Claude را در هر دو سرویس mount کنید.

<Note>
برای اتوماسیون production مشترک یا صورت‌حساب قابل‌پیش‌بینی Anthropic، مسیر کلید API مربوط به Anthropic را ترجیح دهید. استفاده مجدد از Claude CLI تابع نسخه نصب‌شده Claude Code، ورود به حساب، صورت‌حساب و رفتار به‌روزرسانی آن است.
</Note>

### Bonjour / mDNS

شبکه‌سازی bridge در Docker معمولاً multicast مربوط به Bonjour/mDNS (`224.0.0.251:5353`) را به‌طور قابل‌اعتماد forward نمی‌کند. وقتی `OPENCLAW_DISABLE_BONJOUR` تنظیم نشده باشد، Plugin همراه‌شده Bonjour پس از تشخیص اجرا در کانتینر، تبلیغ LAN را به‌طور خودکار غیرفعال می‌کند تا برای multicastای که bridge کنار می‌گذارد وارد چرخه crash و retry نشود. برای غیرفعال‌کردن اجباری آن، مستقل از تشخیص، `OPENCLAW_DISABLE_BONJOUR=1` را تنظیم کنید؛ یا برای فعال‌کردن اجباری آن `0` را تنظیم کنید (فقط در شبکه‌سازی میزبان، macvlan یا شبکه دیگری که کارکرد multicast مربوط به mDNS در آن تأیید شده است).

در غیر این صورت برای میزبان‌های Docker از URL منتشرشده Gateway،‏ Tailscale یا DNS-SD گسترده استفاده کنید. برای نکات ظریف و عیب‌یابی، [کشف Bonjour](/fa/gateway/bonjour) را ببینید.

### ذخیره‌سازی و ماندگاری

Docker Compose،‏ `OPENCLAW_CONFIG_DIR` را به `/home/node/.openclaw`،‏ `OPENCLAW_WORKSPACE_DIR` را به `/home/node/.openclaw/workspace` و `OPENCLAW_AUTH_PROFILE_SECRET_DIR` را به `/home/node/.config/openclaw` به‌صورت bind mount متصل می‌کند تا این مسیرها پس از جایگزینی کانتینر باقی بمانند. وقتی متغیری تنظیم نشده باشد، `docker-compose.yml` به مسیری زیر `${HOME}` بازمی‌گردد، یا اگر خود `HOME` نیز وجود نداشته باشد به `/tmp`، تا `docker compose up` هرگز در محیط‌های خام یک مشخصه volume با منبع خالی تولید نکند.

دایرکتوری پیکربندی mountشده شامل موارد زیر است:

- `openclaw.json` برای پیکربندی رفتار
- `agents/<agentId>/agent/auth-profiles.json` برای احراز هویت ذخیره‌شده ارائه‌دهنده با OAuth/کلید API
- `.env` برای secretهای runtime مبتنی بر env مانند `OPENCLAW_GATEWAY_TOKEN`

دایرکتوری secret مربوط به پروفایل احراز هویت، کلید رمزنگاری محلی مواد token پروفایل احراز هویت مبتنی بر OAuth را ذخیره می‌کند. آن را همراه وضعیت میزبان Docker خود نگه دارید، اما از `OPENCLAW_CONFIG_DIR` جدا کنید.

Pluginهای دانلودشدنی نصب‌شده، وضعیت package را زیر home مربوط به OpenClaw که mount شده است ذخیره می‌کنند؛ بنابراین رکوردهای نصب و ریشه‌های package پس از جایگزینی کانتینر باقی می‌مانند. راه‌اندازی Gateway، درخت‌های وابستگی Pluginهای همراه‌شده را دوباره تولید نمی‌کند.

برای جزئیات کامل ماندگاری VM،‏ [runtime ماشین مجازی Docker ــ چه چیزی کجا ماندگار می‌شود](/fa/install/docker-vm-runtime#what-persists-where) را ببینید.

**نقاط اصلی رشد دیسک:** `media/`، پایگاه‌های داده SQLite هر agent، transcriptهای قدیمی session با قالب JSONL، پایگاه داده SQLite مشترک وضعیت، ریشه‌های package مربوط به Pluginهای نصب‌شده و logهای چرخشی فایل زیر `/tmp/openclaw/`.

### ابزارهای کمکی shell (اختیاری)

برای کوتاه‌ترشدن فرمان‌های روزمره، [ClawDock](/fa/install/clawdock) را نصب کنید:

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

اگر از مسیر قدیمی‌تر `scripts/shell-helpers/clawdock-helpers.sh` نصب کرده‌اید، فرمان بالا را دوباره اجرا کنید تا ابزار کمکی محلی شما مسیر فعلی را دنبال کند. سپس از `clawdock-start`،‏ `clawdock-stop`،‏ `clawdock-dashboard` و موارد دیگر استفاده کنید (برای فهرست کامل `clawdock-help` را اجرا کنید).

<AccordionGroup>
  <Accordion title="فعال‌سازی سندباکس عامل برای Gateway در Docker">
    ```bash
    export OPENCLAW_SANDBOX=1
    ./scripts/docker/setup.sh
    ```

    مسیر سفارشی سوکت (برای نمونه، Docker بدون روت):

    ```bash
    export OPENCLAW_SANDBOX=1
    export OPENCLAW_DOCKER_SOCKET=/run/user/1000/docker.sock
    ./scripts/docker/setup.sh
    ```

    اسکریپت تنها پس از برآورده‌شدن پیش‌نیازهای سندباکس، `docker.sock` را متصل می‌کند. اگر راه‌اندازی سندباکس کامل نشود، `agents.defaults.sandbox.mode` را به `off` بازنشانی می‌کند. حالت کد Codex در نوبت‌هایی که سندباکس OpenClaw فعال است غیرفعال می‌شود (به [سندباکس‌سازی § بک‌اند Docker](/fa/gateway/sandboxing#docker-backend) مراجعه کنید)؛ هرگز سوکت Docker میزبان را داخل کانتینرهای سندباکس عامل متصل نکنید.

  </Accordion>

  <Accordion title="خودکارسازی / CI (غیرتعاملی)">
    تخصیص شبه-TTY در Compose را با `-T` غیرفعال کنید:

    ```bash
    docker compose run -T --rm openclaw-cli gateway probe
    docker compose run -T --rm openclaw-cli devices list --json
    ```

  </Accordion>

  <Accordion title="نکته امنیتی شبکه مشترک">
    `openclaw-cli` از `network_mode: "service:openclaw-gateway"` استفاده می‌کند تا فرمان‌های CLI بتوانند از طریق `127.0.0.1` به Gateway دسترسی پیدا کنند. این وضعیت را یک مرز اعتماد مشترک در نظر بگیرید. پیکربندی Compose، قابلیت‌های `NET_RAW`/`NET_ADMIN` را حذف و `no-new-privileges` را هم در `openclaw-gateway` و هم در `openclaw-cli` فعال می‌کند.
  </Accordion>

  <Accordion title="خطاهای DNS در Docker Desktop برای openclaw-cli">
    در برخی پیکربندی‌های Docker Desktop، جست‌وجوی DNS از سایدکار شبکه مشترک `openclaw-cli` پس از حذف `NET_RAW` ناموفق می‌شود و هنگام اجرای فرمان‌های مبتنی بر npm مانند `openclaw plugins install` به‌صورت `EAI_AGAIN` ظاهر می‌شود. برای عملکرد عادی، فایل Compose سخت‌سازی‌شده پیش‌فرض را حفظ کنید. بازنویسی زیر قابلیت‌های پیش‌فرض را فقط برای کانتینر `openclaw-cli` بازیابی می‌کند — آن را فقط برای فرمان یک‌باره‌ای به‌کار ببرید که به دسترسی رجیستری نیاز دارد، نه به‌عنوان اجرای پیش‌فرض:

    ```bash
    printf '%s\n' \
      'services:' \
      '  openclaw-cli:' \
      '    cap_drop: !reset []' \
      > docker-compose.cli-no-dropped-caps.local.yml

    docker compose -f docker-compose.yml -f docker-compose.cli-no-dropped-caps.local.yml run --rm openclaw-cli plugins install <package>
    ```

    اگر پیش‌تر یک کانتینر طولانی‌مدت `openclaw-cli` ساخته‌اید، آن را با همان بازنویسی دوباره ایجاد کنید — `docker compose exec`/`docker exec` نمی‌تواند قابلیت‌های Linux را در کانتینری که از قبل ایجاد شده است تغییر دهد.

  </Accordion>

  <Accordion title="مجوزها و EACCES">
    ایمیج با کاربر `node` (uid 1000) اجرا می‌شود. اگر در `/home/node/.openclaw` خطای مجوز مشاهده می‌کنید، مطمئن شوید اتصال‌های bind میزبان در مالکیت uid 1000 هستند:

    ```bash
    sudo chown -R 1000:1000 /path/to/openclaw-config /path/to/openclaw-workspace
    ```

    همین عدم تطابق ممکن است به‌صورت `blocked plugin candidate: suspicious ownership (... uid=1000, expected uid=0 or root)` و سپس `plugin present but blocked` ظاهر شود — uid فرایند با مالک دایرکتوری Plugin متصل‌شده متفاوت است. اجرای فرایند با uid پیش‌فرض 1000 و اصلاح مالکیت اتصال bind را ترجیح دهید. تنها در صورتی مالکیت `/path/to/openclaw-config/npm` را به `root:root` تغییر دهید که عمداً می‌خواهید OpenClaw را در بلندمدت با کاربر روت اجرا کنید.

  </Accordion>

  <Accordion title="بازسازی‌های سریع‌تر">
    Dockerfile خود را طوری مرتب کنید که لایه‌های وابستگی در کش باقی بمانند و تا زمانی که lockfileها تغییر نکرده‌اند، از اجرای دوباره `pnpm install` جلوگیری شود:

    ```dockerfile
    FROM node:24-bookworm
    RUN curl -fsSL https://bun.sh/install | bash
    ENV PATH="/root/.bun/bin:${PATH}"
    RUN corepack enable
    WORKDIR /app
    COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
    COPY ui/package.json ./ui/package.json
    COPY scripts ./scripts
    RUN pnpm install --frozen-lockfile
    COPY . .
    RUN pnpm build
    RUN pnpm ui:install
    RUN pnpm ui:build
    ENV NODE_ENV=production
    CMD ["node","dist/index.js"]
    ```

  </Accordion>

  <Accordion title="گزینه‌های کانتینر برای کاربران حرفه‌ای">
    ایمیج پیش‌فرض با اولویت امنیت طراحی شده و به‌صورت `node` غیرروت اجرا می‌شود. برای کانتینری با امکانات کامل‌تر:

    1. **ماندگارکردن `/home/node`**: `export OPENCLAW_HOME_VOLUME="openclaw_home"`
    2. **گنجاندن وابستگی‌های سیستمی در ایمیج**: `export OPENCLAW_IMAGE_APT_PACKAGES="git curl jq"`
    3. **گنجاندن وابستگی‌های Python در ایمیج**: `export OPENCLAW_IMAGE_PIP_PACKAGES="requests==2.32.5 humanize==4.14.0"`
    4. **گنجاندن Playwright Chromium در ایمیج**: `export OPENCLAW_INSTALL_BROWSER=1`، یا استفاده از تگ رسمی ایمیج `-browser`
    5. **یا نصب مرورگرهای Playwright در یک volume ماندگار**:
       ```bash
       docker compose run --rm openclaw-cli \
         node /app/node_modules/playwright-core/cli.js install chromium
       ```
    6. **ماندگارکردن دانلودهای مرورگر**: از `OPENCLAW_HOME_VOLUME` یا `OPENCLAW_EXTRA_MOUNTS` استفاده کنید. OpenClaw در Linux، Chromium مدیریت‌شده با Playwright در ایمیج را به‌طور خودکار شناسایی می‌کند.

  </Accordion>

  <Accordion title="OpenAI Codex OAuth (Docker بدون رابط گرافیکی)">
    اگر در راهنما OpenAI Codex OAuth را انتخاب کنید، یک URL مرورگر باز می‌شود. در Docker یا پیکربندی‌های بدون رابط گرافیکی، URL کامل تغییرمسیری را که به آن می‌رسید کپی کنید و برای تکمیل احراز هویت دوباره در راهنما جای‌گذاری کنید.
  </Accordion>

  <Accordion title="فراداده ایمیج پایه">
    ایمیج زمان اجرا از `node:24-bookworm-slim` استفاده می‌کند و `tini` را به‌عنوان PID 1 اجرا می‌کند تا فرایندهای زامبی جمع‌آوری و سیگنال‌ها در کانتینرهای طولانی‌مدت به‌درستی مدیریت شوند. این ایمیج حاشیه‌نویسی‌های ایمیج پایه OCI، از جمله `org.opencontainers.image.base.name` و `org.opencontainers.image.source` را منتشر می‌کند. Dependabot دایجست سنجاق‌شده ایمیج پایه Node را به‌روزرسانی می‌کند؛ بیلدهای انتشار یک لایه ارتقای توزیع جداگانه اجرا نمی‌کنند. به [حاشیه‌نویسی‌های ایمیج OCI](https://github.com/opencontainers/image-spec/blob/main/annotations.md) مراجعه کنید.
  </Accordion>
</AccordionGroup>

### اجرا روی VPS؟

برای مراحل استقرار در VM مشترک، از جمله گنجاندن فایل‌های اجرایی در ایمیج، ماندگاری و به‌روزرسانی‌ها، به [Hetzner (Docker VPS)](/fa/install/hetzner) و [زمان اجرای VM در Docker](/fa/install/docker-vm-runtime) مراجعه کنید.

## سندباکس عامل

وقتی `agents.defaults.sandbox` با بک‌اند Docker فعال باشد، Gateway اجرای ابزارهای عامل (پوسته، خواندن/نوشتن فایل و غیره) را در کانتینرهای مجزای Docker انجام می‌دهد، درحالی‌که خود Gateway روی میزبان باقی می‌ماند — دیواری سخت در اطراف نشست‌های عامل نامطمئن یا چندمستاجری، بدون کانتینری‌کردن کل Gateway.

دامنه سندباکس می‌تواند برای هر عامل (پیش‌فرض)، هر نشست یا به‌صورت مشترک باشد؛ هر دامنه فضای کاری مختص خود را دارد که در `/workspace` متصل می‌شود. همچنین می‌توانید سیاست‌های مجاز/غیرمجاز ابزارها، جداسازی شبکه، محدودیت منابع و کانتینرهای مرورگر را پیکربندی کنید.

برای پیکربندی کامل، ایمیج‌ها، نکات امنیتی و پروفایل‌های چندعاملی:

- [سندباکس‌سازی](/fa/gateway/sandboxing) -- مرجع کامل سندباکس
- [OpenShell](/fa/gateway/openshell) -- دسترسی تعاملی پوسته به کانتینرهای سندباکس
- [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) -- بازنویسی‌های مختص هر عامل

### فعال‌سازی سریع

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off | non-main | all
        scope: "agent", // session | agent | shared
      },
    },
  },
}
```

ایمیج پیش‌فرض سندباکس را بسازید (از یک checkout کد منبع):

```bash
scripts/sandbox-setup.sh
```

برای نصب‌های npm بدون checkout کد منبع، برای فرمان‌های درون‌خطی `docker build` به [سندباکس‌سازی § ایمیج‌ها و راه‌اندازی](/fa/gateway/sandboxing#images-and-setup) مراجعه کنید.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="ایمیج موجود نیست یا کانتینر سندباکس راه‌اندازی نمی‌شود">
    ایمیج سندباکس را با [`scripts/sandbox-setup.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/sandbox-setup.sh) (checkout کد منبع) یا فرمان درون‌خطی `docker build` از [سندباکس‌سازی § ایمیج‌ها و راه‌اندازی](/fa/gateway/sandboxing#images-and-setup) (نصب npm) بسازید، یا `agents.defaults.sandbox.docker.image` را روی ایمیج سفارشی خود تنظیم کنید. کانتینرها هنگام نیاز برای هر نشست به‌طور خودکار ایجاد می‌شوند.
  </Accordion>

  <Accordion title="خطاهای مجوز در سندباکس">
    `docker.user` را روی UID:GID منطبق با مالکیت فضای کاری متصل‌شده تنظیم کنید، یا مالکیت پوشه فضای کاری را تغییر دهید.
  </Accordion>

  <Accordion title="ابزارهای سفارشی در سندباکس پیدا نمی‌شوند">
    OpenClaw فرمان‌ها را با `sh -lc` (پوسته ورود) اجرا می‌کند که `/etc/profile` را بارگذاری می‌کند و ممکن است PATH را بازنشانی کند. `docker.env.PATH` را طوری تنظیم کنید که مسیر ابزارهای سفارشی شما را به ابتدای مسیر اضافه کند، یا اسکریپتی را در Dockerfile زیر `/etc/profile.d/` اضافه کنید.
  </Accordion>

  <Accordion title="توقف بر اثر OOM هنگام ساخت ایمیج (خروج 137)">
    VM به حداقل 2 GB RAM نیاز دارد. از کلاس ماشین بزرگ‌تری استفاده کنید و دوباره تلاش کنید.
  </Accordion>

  <Accordion title="غیرمجاز یا نیازمند جفت‌سازی در رابط کاربری کنترل">
    یک پیوند تازه داشبورد دریافت کنید و دستگاه مرورگر را تأیید کنید:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    جزئیات بیشتر: [داشبورد](/fa/web/dashboard)، [دستگاه‌ها](/fa/cli/devices).

  </Accordion>

  <Accordion title="هدف Gateway در Docker CLI نشانی ws://172.x.x.x یا خطاهای جفت‌سازی را نشان می‌دهد">
    حالت و bind مربوط به Gateway را بازنشانی کنید:

    ```bash
    docker compose run --rm openclaw-cli config set --batch-json '[{"path":"gateway.mode","value":"local"},{"path":"gateway.bind","value":"lan"}]'
    docker compose run --rm openclaw-cli devices list --url ws://127.0.0.1:18789
    ```

  </Accordion>
</AccordionGroup>

## مرتبط

- [نمای کلی نصب](/fa/install) — همه روش‌های نصب
- [Podman](/fa/install/podman) — جایگزین Podman برای Docker
- [ClawDock](/fa/install/clawdock) — راه‌اندازی اجتماعی Docker Compose
- [به‌روزرسانی](/fa/install/updating) — به‌روز نگه‌داشتن OpenClaw
- [پیکربندی](/fa/gateway/configuration) — پیکربندی Gateway پس از نصب
