---
read_when:
    - به روشی برای نصب غیر از راه‌اندازی سریع «شروع به کار» نیاز دارید
    - می‌خواهید روی یک پلتفرم ابری مستقر کنید
    - باید به‌روزرسانی، مهاجرت یا حذف نصب انجام دهید
summary: نصب OpenClaw — اسکریپت نصب، npm/pnpm/bun، از کد منبع، Docker و موارد دیگر
title: نصب
x-i18n:
    generated_at: "2026-07-27T16:41:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: dc6c6c33294852c90d2d2904b78ff8b0483b8e72a380d5835c5bdda67547de0c
    source_path: install/index.md
    workflow: 16
---

## نیازمندی‌های سیستم

- **Node 22.22.3+، 24.15+ یا 25.9+** - Node 24 هدف پیش‌فرض است؛ اسکریپت نصب این مورد را به‌طور خودکار مدیریت می‌کند.
- **macOS، Linux یا Windows** - کاربران Windows می‌توانند کار را با برنامه بومی Windows Hub، نصب‌کننده CLI مبتنی بر PowerShell یا یک Gateway در WSL2 آغاز کنند. به [Windows](/fa/platforms/windows) مراجعه کنید.
- `pnpm` فقط در صورت ساخت از کد منبع لازم است.

## روش پیشنهادی: اسکریپت نصب

سریع‌ترین روش نصب است. سیستم‌عامل را تشخیص می‌دهد، در صورت نیاز Node را نصب می‌کند، OpenClaw را نصب می‌کند و فرایند راه‌اندازی اولیه را اجرا می‌کند.

<Note>
کاربران دسکتاپ Windows می‌توانند برنامه همراه بومی [Windows Hub](/fa/platforms/windows#recommended-windows-hub) را نیز نصب کنند که شامل راه‌اندازی، وضعیت سینی سیستم، گفت‌وگو، حالت Node و حالت MCP محلی است.
</Note>

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    iwr -useb https://openclaw.ai/install.ps1 | iex
    ```
  </Tab>
</Tabs>

برای نصب بدون اجرای راه‌اندازی اولیه:

<Tabs>
  <Tab title="macOS / Linux / WSL2">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
    ```
  </Tab>
  <Tab title="Windows (PowerShell)">
    ```powershell
    & ([scriptblock]::Create((iwr -useb https://openclaw.ai/install.ps1))) -NoOnboard
    ```
  </Tab>
</Tabs>

برای همه پرچم‌ها و گزینه‌های CI/خودکارسازی، به [جزئیات داخلی نصب‌کننده](/fa/install/installer) مراجعه کنید.

## روش‌های جایگزین نصب

### نصب‌کننده با پیشوند محلی (`install-cli.sh`)

زمانی از این روش استفاده کنید که می‌خواهید OpenClaw و Node زیر یک پیشوند محلی مانند
`~/.openclaw` نگهداری شوند، بدون آنکه به نصب سراسری Node در سیستم وابسته باشید:

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

این روش به‌طور پیش‌فرض از نصب‌های npm و همچنین نصب از نسخه دریافت‌شده با git در همان
جریان پیشوند پشتیبانی می‌کند. مرجع کامل: [جزئیات داخلی نصب‌کننده](/fa/install/installer#install-clish).

قبلاً نصب کرده‌اید؟ با
`openclaw update --channel dev` و `openclaw update --channel stable` بین نصب‌های بسته‌ای و git جابه‌جا شوید. به
[به‌روزرسانی](/fa/install/updating#switch-between-npm-and-git-installs) مراجعه کنید.

### npm، pnpm یا bun

اگر Node را خودتان مدیریت می‌کنید:

<Tabs>
  <Tab title="npm">
    ```bash
    npm install -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    نصب‌کننده میزبانی‌شده، فیلترهای تازگی npm مانند `min-release-age`
    را برای نصب بسته OpenClaw پاک می‌کند. اگر با npm به‌صورت دستی نصب کنید، سیاست
    npm خودتان همچنان اعمال می‌شود.
    </Note>

  </Tab>
  <Tab title="pnpm">
    ```bash
    pnpm add -g openclaw@latest
    pnpm approve-builds -g
    openclaw onboard --install-daemon
    ```

    <Note>
    pnpm برای بسته‌های دارای اسکریپت ساخت به تأیید صریح نیاز دارد. پس از نخستین نصب، `pnpm approve-builds -g` را اجرا کنید.
    </Note>

  </Tab>
  <Tab title="bun">
    ```bash
    bun add -g openclaw@latest
    openclaw onboard --install-daemon
    ```

    <Note>
    Bun می‌تواند بسته سراسری را نصب کند، اما فایل اجرایی `openclaw` حاصل به یک محیط اجرای پشتیبانی‌شده Node نیاز دارد، زیرا وضعیت OpenClaw از `node:sqlite` استفاده می‌کند.
    </Note>

  </Tab>
</Tabs>

### از کد منبع

برای مشارکت‌کنندگان یا هر کسی که می‌خواهد از یک نسخه دریافت‌شده محلی اجرا کند:

```bash
git clone https://github.com/openclaw/openclaw.git
cd openclaw
pnpm install && pnpm build && pnpm ui:build
pnpm link --global
openclaw onboard --install-daemon
```

یا از پیوند صرف‌نظر کنید و `pnpm openclaw ...` را از داخل مخزن به‌کار ببرید. برای جریان‌های کامل توسعه به [راه‌اندازی](/fa/start/setup) مراجعه کنید.

### نصب از نسخه شاخه اصلی GitHub

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --version main
```

### کانتینرها و مدیران بسته

<CardGroup cols={2}>
  <Card title="Docker" href="/fa/install/docker" icon="container">
    استقرارهای کانتینری یا بدون رابط گرافیکی.
  </Card>
  <Card title="Podman" href="/fa/install/podman" icon="container">
    جایگزین کانتینری بدون دسترسی root برای Docker.
  </Card>
  <Card title="Nix" href="/fa/install/nix" icon="snowflake">
    نصب اعلانی از طریق Nix flake.
  </Card>
  <Card title="Ansible" href="/fa/install/ansible" icon="server">
    آماده‌سازی خودکار ناوگان.
  </Card>
  <Card title="Bun" href="/fa/install/bun" icon="zap">
    نصب‌کننده اختیاری وابستگی‌ها و اجراکننده اسکریپت‌های بسته.
  </Card>
</CardGroup>

## تأیید نصب

```bash
openclaw --version      # تأیید کنید CLI در دسترس است
openclaw doctor         # مشکلات پیکربندی را بررسی کنید
openclaw gateway status # بررسی کنید Gateway در حال اجرا است
```

اگر پس از نصب، راه‌اندازی مدیریت‌شده می‌خواهید:

- macOS: عامل راه‌اندازی LaunchAgent از طریق `openclaw onboard --install-daemon` یا `openclaw gateway install`
- Linux/WSL2: سرویس کاربری systemd از طریق همان فرمان‌ها
- Windows بومی: ابتدا Scheduled Task و اگر ایجاد وظیفه مجاز نباشد، یک مورد ورود به سیستم مختص هر کاربر در پوشه Startup به‌عنوان روش جایگزین

## میزبانی و استقرار

OpenClaw را روی سرور ابری یا VPS مستقر کنید. برای گزینش کامل
ارائه‌دهندگان (DigitalOcean، Hetzner، Hostinger، Fly.io، GCP، Azure، Railway،
Northflank، Oracle Cloud، Raspberry Pi و موارد دیگر) به [سرور Linux](/fa/vps) مراجعه کنید، یا استقرار را به‌صورت اعلانی روی
[Render](/fa/install/render) انجام دهید.

<CardGroup cols={3}>
  <Card title="VPS" href="/fa/vps">
    یک ارائه‌دهنده انتخاب کنید.
  </Card>
  <Card title="ماشین مجازی Docker" href="/fa/install/docker-vm-runtime">
    مراحل مشترک Docker.
  </Card>
  <Card title="Kubernetes" href="/fa/install/kubernetes">
    استقرار K8s.
  </Card>
</CardGroup>

## به‌روزرسانی، مهاجرت یا حذف نصب

<CardGroup cols={3}>
  <Card title="به‌روزرسانی" href="/fa/install/updating" icon="refresh-cw">
    OpenClaw را به‌روز نگه دارید.
  </Card>
  <Card title="مهاجرت" href="/fa/install/migrating" icon="arrow-right">
    به یک دستگاه جدید منتقل شوید.
  </Card>
  <Card title="حذف نصب" href="/fa/install/uninstall" icon="trash-2">
    OpenClaw را به‌طور کامل حذف کنید.
  </Card>
</CardGroup>

## عیب‌یابی: `openclaw` یافت نشد

تقریباً همیشه مشکل از PATH است: پوشه باینری سراسری npm در `PATH` پوسته شما قرار ندارد. برای راه‌حل کامل، از جمله مسیر Windows، به [عیب‌یابی Node.js](/fa/install/node#troubleshooting) مراجعه کنید.

```bash
node -v           # آیا Node نصب شده است؟
npm prefix -g     # بسته‌های سراسری کجا هستند؟
echo "$PATH"      # آیا پوشه باینری سراسری در PATH است؟
```
