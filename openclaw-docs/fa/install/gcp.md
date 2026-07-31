---
read_when:
    - می‌خواهید OpenClaw به‌صورت ۲۴/۷ روی GCP اجرا شود
    - یک Gateway همیشه‌فعال و در سطح تولید روی ماشین مجازی خود می‌خواهید
    - می‌خواهید کنترل کاملی بر ماندگاری، فایل‌های اجرایی و رفتار راه‌اندازی مجدد داشته باشید
summary: اجرای ۲۴/۷ Gateway ‏OpenClaw روی ماشین مجازی GCP Compute Engine ‏(Docker) با وضعیت پایدار
title: GCP
x-i18n:
    generated_at: "2026-07-27T15:20:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ca46b2ee78731162261cae6ea5a26b718be6035b998fa92e4ee5c9ea2e7ae07
    source_path: install/gcp.md
    workflow: 16
---

یک Gateway پایدار OpenClaw را با استفاده از Docker روی یک ماشین مجازی GCP Compute Engine اجرا کنید؛ به‌گونه‌ای که وضعیت ماندگار، فایل‌های اجرایی تعبیه‌شده و رفتار راه‌اندازی مجدد ایمن داشته باشد.

قیمت‌گذاری بر اساس نوع ماشین و منطقه متفاوت است؛ کوچک‌ترین ماشین مجازی متناسب با بار کاری خود را انتخاب کنید و اگر با خطاهای کمبود حافظه مواجه شدید، منابع را افزایش دهید.

می‌توان از لپ‌تاپ از طریق انتقال پورت SSH به Gateway دسترسی پیدا کرد، یا اگر دیواره آتش و توکن‌ها را خودتان مدیریت می‌کنید، پورت را مستقیماً در دسترس قرار داد.

این راهنما از Debian روی GCP Compute Engine استفاده می‌کند. Ubuntu نیز کار می‌کند؛ بسته‌ها را متناسب با آن جایگزین کنید. برای روند عمومی Docker، به [Docker](/fa/install/docker) مراجعه کنید.

## پیش‌نیازها

- حساب GCP (`e2-micro` واجد شرایط استفاده از سطح رایگان است)
- CLI ‏`gcloud`، یا [Cloud Console](https://console.cloud.google.com)
- دسترسی SSH از لپ‌تاپ
- Docker و Docker Compose
- اعتبارنامه‌های احراز هویت مدل
- اعتبارنامه‌های اختیاری ارائه‌دهنده (کد QR ‏WhatsApp، توکن ربات Telegram، ‏OAuth ‏Gmail)
- حدود 20-30 دقیقه

## مسیر سریع

1. یک پروژه GCP ایجاد و صورت‌حساب و Compute Engine API را فعال کنید
2. یک ماشین مجازی Compute Engine ایجاد کنید (`e2-small`، ‏Debian 12، ‏20GB)
3. از طریق SSH وارد ماشین مجازی شوید و Docker را نصب کنید
4. مخزن OpenClaw را کلون کنید
5. دایرکتوری‌های ماندگار میزبان را ایجاد کنید
6. `.env` و `docker-compose.yml` را پیکربندی کنید
7. فایل‌های اجرایی لازم را در ایمیج تعبیه کنید، بسازید و راه‌اندازی کنید

<Steps>
  <Step title="نصب gcloud CLI (یا استفاده از Console)">
    آن را از [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install) نصب کنید، سپس:

    ```bash
    gcloud init
    gcloud auth login
    ```

    یا تمام مراحل زیر را از طریق رابط وب [Cloud Console](https://console.cloud.google.com) انجام دهید.

  </Step>

  <Step title="ایجاد پروژه GCP">
    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    gcloud services enable compute.googleapis.com
    ```

    صورت‌حساب را در [console.cloud.google.com/billing](https://console.cloud.google.com/billing) فعال کنید (برای Compute Engine الزامی است).

    معادل آن در Console: ‏IAM & Admin > Create Project، صورت‌حساب را فعال کنید، سپس APIs & Services > Enable APIs > "Compute Engine API" > Enable.

  </Step>

  <Step title="ایجاد ماشین مجازی">
    | نوع       | مشخصات                   | هزینه              | توضیحات                                              |
    | --------- | ------------------------ | ------------------ | ---------------------------------------------------- |
    | e2-medium | 2 vCPU، ‏4GB RAM         | ~$25/mo            | مطمئن‌ترین گزینه برای ساخت محلی Docker               |
    | e2-small  | 2 vCPU، ‏2GB RAM         | ~$12/mo            | حداقل گزینه توصیه‌شده برای ساخت Docker               |
    | e2-micro  | 2 vCPU (اشتراکی)، ‏1GB RAM | واجد شرایط سطح رایگان | اغلب ساخت Docker به‌دلیل کمبود حافظه شکست می‌خورد (خروجی 137) |

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

  </Step>

  <Step title="ورود به ماشین مجازی از طریق SSH">
    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    در Console: در داشبورد Compute Engine، کنار ماشین مجازی روی "SSH" کلیک کنید.

    انتشار کلید SSH پس از ایجاد ماشین مجازی ممکن است 1-2 دقیقه طول بکشد؛ اگر اتصال رد شد، صبر کنید و دوباره تلاش کنید.

  </Step>

  <Step title="نصب Docker (روی ماشین مجازی)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    برای اعمال تغییر گروه، خارج شوید و دوباره وارد شوید، سپس مجدداً از طریق SSH متصل شوید:

    ```bash
    exit
    ```

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    بررسی کنید:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="کلون‌کردن مخزن OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    این راهنما یک ایمیج سفارشی می‌سازد تا همه فایل‌های اجرایی تعبیه‌شده در راه‌اندازی‌های مجدد باقی بمانند.

  </Step>

  <Step title="ایجاد دایرکتوری‌های ماندگار میزبان">
    کانتینرهای Docker موقتی هستند؛ تمام وضعیت بلندمدت باید روی میزبان نگهداری شود.

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="پیکربندی متغیرهای محیطی">
    فایل `.env` را در ریشه مخزن ایجاد کنید:

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    برای مدیریت توکن پایدار Gateway از طریق
    `.env`، مقدار `OPENCLAW_GATEWAY_TOKEN` را تنظیم کنید؛ در غیر این صورت، پیش از اتکا به کلاینت‌ها
    در راه‌اندازی‌های مجدد، `gateway.auth.token` را پیکربندی کنید. اگر هیچ‌کدام تنظیم نشده باشد، OpenClaw برای
    همان راه‌اندازی از توکنی استفاده می‌کند که فقط در زمان اجرا معتبر است. برای `GOG_KEYRING_PASSWORD` یک گذرواژه keyring ایجاد کنید:

    ```bash
    openssl rand -hex 32
    ```

    **این فایل را commit نکنید.** این فایل شامل متغیرهای محیطی کانتینر/زمان اجرا مانند
    `OPENCLAW_GATEWAY_TOKEN` است. احراز هویت ذخیره‌شده OAuth/کلید API ارائه‌دهنده در
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` مونت‌شده قرار دارد.

  </Step>

  <Step title="پیکربندی Docker Compose">
    فایل `docker-compose.yml` را ایجاد یا به‌روزرسانی کنید:

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # توصیه می‌شود Gateway را فقط روی loopback ماشین مجازی نگه دارید و از طریق تونل SSH به آن دسترسی پیدا کنید.
          # برای دسترسی عمومی، پیشوند `127.0.0.1:` را حذف و دیواره آتش را متناسب با آن پیکربندی کنید.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` فقط برای سهولت راه‌اندازی اولیه است و جایگزین پیکربندی واقعی Gateway نیست. همچنان احراز هویت (`gateway.auth.token` یا گذرواژه) و حالت اتصال ایمن را برای استقرار خود تنظیم کنید.

  </Step>

  <Step title="مراحل مشترک زمان اجرای ماشین مجازی Docker">
    برای روند مشترک میزبان Docker، راهنمای زمان اجرای مشترک را دنبال کنید:

    - [تعبیه فایل‌های اجرایی لازم در ایمیج](/fa/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [ساخت و راه‌اندازی](/fa/install/docker-vm-runtime#build-and-launch)
    - [محل ماندگاری هر بخش](/fa/install/docker-vm-runtime#what-persists-where)
    - [به‌روزرسانی‌ها](/fa/install/docker-vm-runtime#updates)

  </Step>

  <Step title="نکات راه‌اندازی ویژه GCP">
    اگر ساخت هنگام `pnpm install --frozen-lockfile` با `Killed` یا `exit code 137` شکست خورد، حافظه ماشین مجازی تمام شده است. حداقل از `e2-small` یا برای ساخت اولیه مطمئن‌تر از `e2-medium` استفاده کنید.

    هنگام اتصال به LAN ‏(`OPENCLAW_GATEWAY_BIND=lan`)، پیش از ادامه یک origin مرورگر مورد اعتماد پیکربندی کنید:

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    اگر پورت را تغییر داده‌اید، `18789` را با پورت پیکربندی‌شده خود جایگزین کنید.

  </Step>

  <Step title="دسترسی از لپ‌تاپ">
    برای انتقال پورت Gateway، یک تونل SSH ایجاد کنید:

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    `http://127.0.0.1:18789/` را در مرورگر خود باز کنید.

    یک پیوند پاک داشبورد را دوباره نمایش دهید:

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    اگر رابط کاربری برای احراز هویت با راز مشترک درخواست داد، توکن یا
    گذرواژه پیکربندی‌شده را در تنظیمات Control UI وارد کنید (این روند Docker به‌طور
    پیش‌فرض یک توکن می‌نویسد؛ اگر به احراز هویت با گذرواژه تغییر داده‌اید، به‌جای آن
    از گذرواژه پیکربندی‌شده استفاده کنید).

    اگر Control UI عبارت `unauthorized` یا `disconnected (1008): pairing required` را نشان داد، دستگاه مرورگر را تأیید کنید:

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    برای نقشه ماندگاری مشترک و [روند به‌روزرسانی](/fa/install/docker-vm-runtime#updates)، به [زمان اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime#what-persists-where) مراجعه کنید.

  </Step>
</Steps>

## عیب‌یابی

**اتصال SSH رد شد**

انتشار کلید SSH پس از ایجاد ماشین مجازی ممکن است 1-2 دقیقه طول بکشد. صبر کنید و دوباره تلاش کنید.

**مشکلات OS Login**

پروفایل OS Login خود را بررسی کنید:

```bash
gcloud compute os-login describe-profile
```

مطمئن شوید حساب شما مجوزهای IAM لازم را دارد (Compute OS Login یا Compute OS Admin Login).

**کمبود حافظه (OOM)**

اگر ساخت Docker با `Killed` و `exit code 137` شکست خورد، ماشین مجازی به‌دلیل کمبود حافظه متوقف شده است:

```bash
# ابتدا ماشین مجازی را متوقف کنید
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# نوع ماشین را تغییر دهید
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# ماشین مجازی را راه‌اندازی کنید
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

## حساب‌های سرویس (روش برتر امنیتی)

برای استفاده شخصی، حساب کاربری پیش‌فرض شما به‌خوبی کار می‌کند. برای خودکارسازی یا CI/CD، یک حساب سرویس اختصاصی با حداقل مجوزها ایجاد کنید:

```bash
gcloud iam service-accounts create openclaw-deploy \
  --display-name="OpenClaw Deployment"

gcloud projects add-iam-policy-binding my-openclaw-project \
  --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"
```

برای خودکارسازی از نقش Owner استفاده نکنید؛ محدودترین نقشی را به‌کار ببرید که نیاز شما را برآورده می‌کند. به [آشنایی با نقش‌ها](https://cloud.google.com/iam/docs/understanding-roles) مراجعه کنید.

## مراحل بعدی

- کانال‌های پیام‌رسانی را راه‌اندازی کنید: [کانال‌ها](/fa/channels)
- دستگاه‌های محلی را به‌عنوان Node جفت کنید: [Nodeها](/fa/nodes)
- Gateway را پیکربندی کنید: [پیکربندی Gateway](/fa/gateway/configuration)

## مرتبط

- [نمای کلی نصب](/fa/install)
- [Azure](/fa/install/azure)
- [میزبانی VPS](/fa/vps)
