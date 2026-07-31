---
read_when: You want a dedicated explanation of sandboxing or need to tune agents.defaults.sandbox.
sidebarTitle: Sandboxing
status: active
summary: 'نحوهٔ عملکرد سندباکس OpenClaw: حالت‌ها، دامنه‌ها، دسترسی به فضای کاری و ایمیج‌ها'
title: سندباکس‌سازی
x-i18n:
    generated_at: "2026-07-27T15:16:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a3668dc512a8ff30732290ee68e9dd29a3a2e9c106e6e39077a97bfbd90098f7
    source_path: gateway/sandboxing.md
    workflow: 16
---

OpenClaw می‌تواند اجرای ابزار را درون یک بک‌اند سندباکس انجام دهد تا دامنهٔ آسیب احتمالی کاهش یابد. سندباکس به‌طور پیش‌فرض غیرفعال است و با `agents.defaults.sandbox` (سراسری) یا `agents.entries.*.sandbox` (برای هر عامل) کنترل می‌شود. فرایند Gateway همیشه روی میزبان باقی می‌ماند؛ فقط اجرای ابزار، در صورت فعال‌بودن، به سندباکس منتقل می‌شود.

<Note>
این یک مرز امنیتی بی‌نقص نیست، اما وقتی مدل کار احمقانه‌ای انجام می‌دهد، دسترسی به سیستم فایل و فرایندها را به‌طور محسوسی محدود می‌کند.
</Note>

## چه چیزهایی در سندباکس اجرا می‌شوند

- اجرای ابزار: `exec`، `read`، `write`، `edit`، `apply_patch`، `process` و غیره.
- مرورگر سندباکس اختیاری (`agents.defaults.sandbox.browser`).

مواردی که در سندباکس اجرا نمی‌شوند:

- خود فرایند Gateway.
- هر ابزاری که صراحتاً از طریق `tools.elevated` مجاز شده باشد بیرون از سندباکس اجرا شود. اجرای ارتقایافته سندباکس را دور می‌زند و در مسیر خروج پیکربندی‌شده اجرا می‌شود (`gateway` به‌طور پیش‌فرض، یا `node` وقتی مقصد اجرا `node` است). اگر سندباکس غیرفعال باشد، `tools.elevated` چیزی را تغییر نمی‌دهد، زیرا اجرا از قبل روی میزبان انجام می‌شود. [حالت ارتقایافته](/fa/tools/elevated) را ببینید.

## حالت‌ها، دامنه و بک‌اند

سه تنظیم مستقل رفتار سندباکس را کنترل می‌کنند:

| تنظیم | کلید                               | مقادیر                       | پیش‌فرض  |
| ------- | --------------------------------- | ---------------------------- | -------- |
| حالت    | `agents.defaults.sandbox.mode`    | `off`، `non-main`، `all`     | `off`    |
| دامنه   | `agents.defaults.sandbox.scope`   | `agent`، `session`، `shared` | `agent`  |
| بک‌اند | `agents.defaults.sandbox.backend` | `docker`، `ssh`، `openshell` | `docker` |

**حالت** زمان اعمال سندباکس را کنترل می‌کند:

- `off`: بدون سندباکس.
- `non-main`: همهٔ نشست‌ها به‌جز نشست اصلی عامل در سندباکس اجرا می‌شوند. کلید نشست اصلی همیشه `agent:<agentId>:main` است (یا وقتی `session.scope` برابر `"global"` باشد، `global`)؛ این مورد قابل پیکربندی نیست. نشست‌های گروه/کانال کلیدهای خود را دارند، بنابراین همیشه غیر اصلی محسوب می‌شوند و در سندباکس اجرا می‌شوند.
- `all`: همهٔ نشست‌ها در سندباکس اجرا می‌شوند.

**دامنه** تعداد کانتینرها/محیط‌های ایجادشده را کنترل می‌کند:

- `agent`: یک کانتینر برای هر عامل.
- `session`: یک کانتینر برای هر نشست.
- `shared`: یک کانتینر مشترک برای همهٔ نشست‌های سندباکس‌شده (نادیده‌گرفتن جایگزین‌های `docker`/`ssh`/`browser` مخصوص هر عامل در این دامنه).

**بک‌اند** مشخص می‌کند کدام محیط اجرا ابزارهای سندباکس‌شده را اجرا کند. پیکربندی مختص SSH زیر `agents.defaults.sandbox.ssh` قرار دارد؛ پیکربندی مختص OpenShell زیر `plugins.entries.openshell.config` قرار دارد.

|                     | Docker                           | SSH                            | OpenShell                                           |
| ------------------- | -------------------------------- | ------------------------------ | --------------------------------------------------- |
| **محل اجرا**   | کانتینر محلی                  | هر میزبان قابل دسترسی با SSH        | سندباکس مدیریت‌شدهٔ OpenShell                           |
| **راه‌اندازی**           | `scripts/sandbox-setup.sh`       | کلید SSH + میزبان مقصد          | Plugin مربوط به OpenShell فعال باشد                            |
| **مدل فضای کاری** | اتصال bind یا کپی               | راه‌دور به‌عنوان مرجع اصلی (یک‌بار مقداردهی اولیه)   | `mirror` یا `remote`                                |
| **کنترل شبکه** | `docker.network` (پیش‌فرض: هیچ‌کدام) | وابسته به میزبان راه‌دور         | وابسته به OpenShell                                |
| **سندباکس مرورگر** | پشتیبانی می‌شود                        | پشتیبانی نمی‌شود                  | هنوز پشتیبانی نمی‌شود                                   |
| **اتصال‌های bind**     | `docker.binds`                   | نامربوط                            | نامربوط                                                 |
| **مناسب برای**        | توسعهٔ محلی، جداسازی کامل        | واگذاری پردازش به یک ماشین راه‌دور | سندباکس‌های راه‌دور مدیریت‌شده با همگام‌سازی دوطرفهٔ اختیاری |

## بک‌اند Docker

پس از فعال‌شدن سندباکس، Docker بک‌اند پیش‌فرض است. ابزارها و مرورگرهای سندباکس را به‌صورت محلی و از طریق سوکت دیمن Docker (`/var/run/docker.sock`) اجرا می‌کند؛ جداسازی توسط فضاهای نام Docker فراهم می‌شود.

پیش‌فرض‌ها: `network: "none"` (بدون خروجی شبکه)، `readOnlyRoot: true`، `capDrop: ["ALL"]`، تصویر `openclaw-sandbox:bookworm-slim`.

برای دردسترس‌قراردادن GPUهای میزبان، `agents.defaults.sandbox.docker.gpus` (یا جایگزین مخصوص هر عامل) را روی مقداری مانند `"all"` یا `"device=GPU-uuid"` تنظیم کنید. این مقدار به پرچم `--gpus` در Docker فرستاده می‌شود و به یک محیط اجرای سازگار روی میزبان، مانند NVIDIA Container Toolkit، نیاز دارد.

<Warning>
**محدودیت‌های Docker-out-of-Docker (DooD)**

اگر خود OpenClaw Gateway را به‌صورت کانتینر Docker مستقر کنید، کانتینرهای سندباکس هم‌سطح را با استفاده از سوکت Docker میزبان هماهنگ می‌کند (DooD). این کار یک محدودیت نگاشت مسیر ایجاد می‌کند:

- **پیکربندی به مسیرهای میزبان نیاز دارد**: `openclaw.json` `workspace` باید شامل **مسیر مطلق میزبان** باشد (برای مثال `/home/user/.openclaw/workspaces`)، نه مسیر داخلی کانتینر Gateway. دیمن Docker مسیرها را نسبت به فضای نام سیستم‌عامل میزبان ارزیابی می‌کند، نه فضای نام خود Gateway.
- **نگاشت volume یکسان الزامی است**: فرایند Gateway فایل‌های Heartbeat و پل را نیز در همان مسیر `workspace` می‌نویسد. یک نگاشت volume یکسان (`-v /home/user/.openclaw:/home/user/.openclaw`) به کانتینر Gateway بدهید تا همان مسیر میزبان از داخل کانتینر Gateway نیز به‌درستی resolve شود. نگاشت‌های ناهماهنگ هنگام تلاش Gateway برای نوشتن Heartbeat خود، به‌شکل `EACCES` ظاهر می‌شوند.
- **حالت کد Codex**: وقتی سندباکس OpenClaw فعال است، OpenClaw حالت کد بومی app-server مربوط به Codex، سرورهای MCP کاربر و اجرای Plugin متکی به برنامه را برای آن نوبت غیرفعال می‌کند (این موارد از فرایند app-server روی میزبان Gateway اجرا می‌شوند، نه از بک‌اند سندباکس OpenClaw)، مگر اینکه سیاست ابزار سندباکس ابزارهای موردنیاز را در دسترس قرار دهد و مسیر آزمایشی exec-server سندباکس را فعال کنید. سپس دسترسی پوسته از طریق ابزارهای متکی به سندباکس OpenClaw، مانند `sandbox_exec` و `sandbox_process`، مسیریابی می‌شود. سوکت Docker میزبان را در کانتینرهای سندباکس عامل یا سندباکس‌های سفارشی Codex mount نکنید. برای رفتار کامل، [هارنس Codex](/fa/plugins/codex-harness) را ببینید.

در میزبان‌های Ubuntu/AppArmor که حالت سندباکس Docker در آن‌ها فعال است، اجرای پوستهٔ `workspace-write` در app-server مربوط به Codex به فضاهای نام کاربر بدون امتیاز درون کانتینر سندباکس نیاز دارد و اگر کاربر سرویس نتواند آن‌ها را ایجاد کند، ممکن است پیش از شروع پوسته شکست بخورد. هنگامی که خروجی شبکهٔ سندباکس Docker غیرفعال است (`network: "none"`، مقدار پیش‌فرض)، این کار به یک فضای نام شبکهٔ بدون امتیاز نیز نیاز دارد. نشانه‌های رایج: `bwrap: setting up uid map: Permission denied` و `bwrap: loopback: Failed RTM_NEWADDR: Operation not permitted`. دستور `openclaw doctor` را اجرا کنید؛ اگر شکست آزمون فضای نام bwrap مربوط به Codex را گزارش کرد، یک پروفایل AppArmor را ترجیح دهید که فضاهای نام موردنیاز را برای فرایند سرویس OpenClaw مجاز کند. `kernel.apparmor_restrict_unprivileged_userns=0` یک راه‌حل جایگزین در سطح کل میزبان با ملاحظات امنیتی است؛ فقط زمانی از آن استفاده کنید که وضعیت امنیتی حاصل برای آن میزبان پذیرفتنی باشد.
</Warning>

### مرورگر سندباکس

- مرورگر سندباکس هنگامی که ابزار مرورگر به آن نیاز دارد، به‌طور خودکار راه‌اندازی می‌شود (تا دسترس‌پذیری CDP تضمین شود). آن را از طریق `agents.defaults.sandbox.browser.autoStart` (پیش‌فرض `true`) و `autoStartTimeoutMs` (پیش‌فرض 12s) پیکربندی کنید.
- کانتینرهای مرورگر سندباکس به‌جای شبکهٔ سراسری `bridge` از یک شبکهٔ اختصاصی Docker (`openclaw-sandbox-browser`) استفاده می‌کنند. آن را با `agents.defaults.sandbox.browser.network` پیکربندی کنید.
- `agents.defaults.sandbox.browser.cdpSourceRange` ورودی CDP در لبهٔ کانتینر را با فهرست مجاز CIDR محدود می‌کند (برای مثال `172.21.0.1/32`).
- دسترسی مشاهده‌گر noVNC به‌طور پیش‌فرض با گذرواژه محافظت می‌شود؛ OpenClaw یک نشانی URL دارای توکن کوتاه‌عمر منتشر می‌کند که یک صفحهٔ راه‌انداز محلی ارائه می‌دهد و noVNC را با گذرواژه در fragment نشانی URL باز می‌کند (نه در رشتهٔ پرس‌وجو یا گزارش‌های header).
- `agents.defaults.sandbox.browser.allowHostControl` (پیش‌فرض `false`) به نشست‌های سندباکس‌شده اجازه می‌دهد مرورگر میزبان را صراحتاً هدف بگیرند.
- فهرست‌های مجاز اختیاری دسترسی به `target: "custom"` را کنترل می‌کنند: `allowedControlUrls`، `allowedControlHosts`، `allowedControlPorts`.

## بک‌اند SSH

برای اجرای سندباکس‌شدهٔ `exec`، ابزارهای فایل و خواندن رسانه روی هر ماشین قابل دسترسی با SSH، از `backend: "ssh"` استفاده کنید.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "ssh",
        scope: "session",
        workspaceAccess: "rw",
        ssh: {
          target: "user@gateway-host:22",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // یا به‌جای فایل‌های محلی از SecretRefs / محتوای درون‌خطی استفاده کنید:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
      },
    },
  },
}
```

پیش‌فرض‌ها: `command: "ssh"`، `workspaceRoot: "/tmp/openclaw-sandboxes"`، `strictHostKeyChecking: true`، `updateHostKeys: true`.

- **چرخهٔ حیات**: OpenClaw یک ریشهٔ راه‌دور برای هر دامنه زیر `sandbox.ssh.workspaceRoot` ایجاد می‌کند. در نخستین استفاده پس از ایجاد یا ایجاد مجدد، فضای کاری راه‌دور را یک‌بار از فضای کاری محلی مقداردهی اولیه می‌کند. پس از آن، `exec`، `read`، `write`، `edit`، `apply_patch`، خواندن رسانه‌های پرامپت و آماده‌سازی رسانه‌های ورودی، مستقیماً از طریق SSH روی فضای کاری راه‌دور اجرا می‌شوند. OpenClaw تغییرات راه‌دور را به‌طور خودکار با فضای کاری محلی همگام نمی‌کند.
- **مواد احراز هویت**: `identityFile`/`certificateFile`/`knownHostsFile` به فایل‌های محلی موجود ارجاع می‌دهند. `identityData`/`certificateData`/`knownHostsData` رشته‌های درون‌خطی یا SecretRefs را می‌پذیرند که از طریق snapshot معمول محیط اجرای اسرار resolve می‌شوند، با حالت `0600` در فایل‌های موقت نوشته می‌شوند و با پایان نشست SSH حذف می‌شوند. اگر برای یک مورد هم گونهٔ `*File` و هم گونهٔ `*Data` تنظیم شده باشند، `*Data` در آن نشست اولویت دارد.
- **پیامدهای راه‌دور به‌عنوان مرجع اصلی**: پس از مقداردهی اولیه، فضای کاری SSH راه‌دور به وضعیت واقعی سندباکس تبدیل می‌شود. ویرایش‌های محلی میزبان که پس از مرحلهٔ مقداردهی اولیه خارج از OpenClaw انجام شوند، تا زمانی که سندباکس را دوباره ایجاد نکنید در راه‌دور قابل مشاهده نیستند. `openclaw sandbox recreate` ریشهٔ راه‌دور هر دامنه را حذف می‌کند و در استفادهٔ بعدی دوباره از فضای محلی مقداردهی اولیه می‌کند. سندباکس مرورگر در این بک‌اند پشتیبانی نمی‌شود و تنظیمات `sandbox.docker.*` برای آن اعمال نمی‌شوند.

## بک‌اند OpenShell

برای اجرای سندباکس‌شدهٔ ابزارها در یک محیط راه‌دور مدیریت‌شده با OpenShell، از `backend: "openshell"` استفاده کنید. OpenShell از همان انتقال SSH و پل سیستم فایل راه‌دور بک‌اند عمومی SSH استفاده می‌کند و چرخهٔ حیات OpenShell (`sandbox create/get/delete/ssh-config`) به‌همراه حالت اختیاری همگام‌سازی فضای کاری `mirror` را به آن می‌افزاید.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        backend: "openshell",
        scope: "session",
        workspaceAccess: "rw",
      },
    },
  },
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          from: "openclaw",
          mode: "remote", // آینه‌ای | راه‌دور
        },
      },
    },
  },
}
```

`mode: "mirror"` (پیش‌فرض) فضای کاری محلی را به‌عنوان منبع مرجع نگه می‌دارد: OpenClaw پیش از `exec` محتوای محلی را با sandbox همگام می‌کند و پس از آن تغییرات را بازمی‌گرداند. `mode: "remote"` یک‌بار فضای کاری راه‌دور را از فضای محلی مقداردهی اولیه می‌کند، سپس `exec`/`read`/`write`/`edit`/`apply_patch` را مستقیماً روی فضای کاری راه‌دور و بدون همگام‌سازی معکوس اجرا می‌کند؛ ویرایش‌های محلی پس از مقداردهی اولیه تا زمانی که `openclaw sandbox recreate` را انجام ندهید، قابل مشاهده نیستند. در `scope: "agent"` یا `scope: "shared"`، آن فضای کاری راه‌دور در همان محدوده مشترک است. محدودیت‌های فعلی: مرورگر sandbox هنوز پشتیبانی نمی‌شود و `sandbox.docker.binds` برای این backend اعمال نمی‌شود.

`openclaw sandbox list`/`recreate`/هرس، همگی runtimeهای OpenShell را مانند runtimeهای Docker در نظر می‌گیرند؛ منطق هرس از backend آگاه است.

برای پیش‌نیازهای کامل، مرجع پیکربندی، مقایسه حالت‌های فضای کاری و جزئیات چرخه حیات، به [OpenShell](/fa/gateway/openshell) مراجعه کنید.

## دسترسی به فضای کاری

`agents.defaults.sandbox.workspaceAccess` تعیین می‌کند sandbox چه چیزهایی را می‌تواند ببیند:

| مقدار            | رفتار                                                                                  |
| ---------------- | ----------------------------------------------------------------------------------------- |
| `none` (پیش‌فرض) | ابزارها یک فضای کاری sandbox ایزوله را در `~/.openclaw/sandboxes` می‌بینند.                    |
| `ro`             | فضای کاری agent را به‌صورت فقط‌خواندنی در `/agent` mount می‌کند (`write`/`edit`/`apply_patch` را غیرفعال می‌کند). |
| `rw`             | فضای کاری agent را به‌صورت خواندنی/نوشتنی در `/workspace` mount می‌کند.                                    |

با backend نوع OpenShell، حالت `mirror` همچنان بین نوبت‌های exec از فضای کاری محلی به‌عنوان منبع مرجع استفاده می‌کند، حالت `remote` پس از مقداردهی اولیه از فضای کاری راه‌دور OpenShell به‌عنوان منبع مرجع استفاده می‌کند و `workspaceAccess: "ro"`/`"none"` همچنان رفتار نوشتن را به همان شیوه محدود می‌کنند.

رسانه ورودی در فضای کاری sandbox فعال کپی می‌شود (`media/inbound/*`).

<Note>
**Skills**: ابزار `read` در ریشه sandbox قرار دارد. با `workspaceAccess: "none"`، OpenClaw مهارت‌های واجد شرایط را در فضای کاری sandbox آینه می‌کند (`.../skills`) تا قابل خواندن باشند. با `"rw"`، مهارت‌های فضای کاری از `/workspace/skills` قابل خواندن‌اند و مهارت‌های مدیریت‌شده، همراه یا Plugin واجد شرایط در مسیر فقط‌خواندنی تولیدشده `/workspace/.openclaw/sandbox-skills/skills` ایجاد می‌شوند.
</Note>

## چند پوشه برای یک agent

وقتی یک agent درون sandbox به بیش از فضای کاری اصلی خود نیاز دارد، از bind mountهای Docker استفاده کنید. هر ورودی یک پوشه میزبان را با یک حالت دسترسی صریح به مسیری در کانتینر نگاشت می‌کند:

```text
host-directory:container-directory:ro
host-directory:container-directory:rw
```

- `ro` پوشه mountشده را درون sandbox فقط‌خواندنی می‌کند.
- `rw` به ابزارها و فرایندهای درون sandbox اجازه می‌دهد پوشه میزبان را تغییر دهند.
- مسیر کانتینر، مسیری است که agent استفاده می‌کند. مسیرهای میزبان به‌طور خودکار آشکار نمی‌شوند.

این نمونه به agent با شناسه `research` یک فضای کاری اصلی قابل‌نوشتن، منابع مرجع فقط‌خواندنی در `/reference` و یک پوشه خروجی قابل‌نوشتن مجزا در `/drafts` می‌دهد:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "all",
        scope: "agent",
      },
    },
    list: [
      {
        id: "research",
        workspace: "/srv/openclaw/research-workspace",
        sandbox: {
          workspaceAccess: "rw",
          docker: {
            binds: ["/srv/shared/reference:/reference:ro", "/srv/shared/drafts:/drafts:rw"],
            // الزامی است، زیرا این منابع خارج از فضای کاری agent قرار دارند.
            dangerouslyAllowExternalBindSources: true,
          },
        },
      },
    ],
  },
}
```

`workspaceAccess` و حالت‌های bind مستقل از یکدیگرند:

| تنظیم                          | کنترل می‌کند                                                                    |
| -------------------------------- | --------------------------------------------------------------------------- |
| `workspaceAccess: "none"`        | از یک فضای کاری sandbox ایزوله استفاده می‌کند؛ فضای کاری agent را آشکار نمی‌کند.    |
| `workspaceAccess: "ro"`          | فضای کاری agent را به‌صورت فقط‌خواندنی در `/agent` mount می‌کند.                           |
| `workspaceAccess: "rw"`          | فضای کاری agent را به‌صورت خواندنی/نوشتنی در `/workspace` mount می‌کند.                      |
| ورودی `docker.binds` با `:ro`/`:rw` | فقط همان پوشه میزبان اضافی را در مسیر کانتینر پیکربندی‌شده آن کنترل می‌کند. |

تغییر `workspaceAccess` یک bind اضافی را از `ro` به `rw` یا برعکس تغییر نمی‌دهد. `docker.binds` سراسری و مختص هر agent با هم ادغام می‌شوند. برای bindهای مختص هر agent، `scope: "agent"` یا `"session"` را نگه دارید؛ `scope: "shared"` تمام overrideهای Docker مختص هر agent را نادیده می‌گیرد و فقط از bindهای سراسری استفاده می‌کند.

Bind mountها مرز پشتیبانی‌شده برای چند پوشه هستند، زیرا Docker نمای سیستم فایل کانتینر را با جداسازی mount می‌سازد و حالت `ro`/`rw` بر همه فرایندهای sandbox اعمال می‌شود. این مرز، `exec`، ابزارهای سیستم فایل، فرایندهای فرزند و کتابخانه‌ها را پوشش می‌دهد، بدون آنکه بررسی‌های مجوز مسیر در هر مسیر کد OpenClaw تکرار شوند. یک فهرست مجاز مسیر در سمت میزبان نمی‌تواند همان مرز کامل را فراهم کند، زیرا یک shell یا وابستگی مجاز می‌تواند مستقیماً به فایل‌ها دسترسی پیدا کند.

`dangerouslyAllowExternalBindSources` اختیاری فقط منابع خارج از ریشه‌های فضای کاری را مجاز می‌کند. این گزینه بررسی‌های OpenClaw برای سیستم مسدودشده، اعتبارنامه‌ها، socket مربوط به Docker، والد symlink یا مقصدهای رزروشده را غیرفعال نمی‌کند. کوچک‌ترین پوشه را ترجیح دهید، مگر زمانی که نوشتن لازم است از `ro` استفاده کنید و پس از تغییر mountها sandbox را دوباره ایجاد کنید:

```bash
openclaw sandbox recreate --agent research
```

### سایر رفتارهای bind

`agents.defaults.sandbox.docker.binds` mountهای سراسری را پیکربندی می‌کند. قالب آن همان فرم `host:container:mode` است (برای نمونه، `"/home/user/source:/source:rw"`).

`agents.defaults.sandbox.browser.binds` دایرکتوری‌های میزبان اضافی را فقط در کانتینر **مرورگر sandbox** mount می‌کند. وقتی تنظیم شده باشد (از جمله `[]`) جایگزین `docker.binds` برای کانتینر مرورگر می‌شود؛ وقتی حذف شده باشد، کانتینر مرورگر به `docker.binds` بازمی‌گردد.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          binds: ["/home/user/source:/source:ro", "/var/data/myapp:/data:ro"],
        },
      },
    },
    list: [
      {
        id: "build",
        sandbox: {
          docker: {
            binds: ["/mnt/cache:/cache:rw"],
          },
        },
      },
    ],
  },
}
```

<Warning>
**امنیت bind**

- Bindها سیستم فایل sandbox را دور می‌زنند: آن‌ها مسیرهای میزبان را با هر حالتی که تنظیم کنید (`:ro` یا `:rw`) آشکار می‌کنند.
- OpenClaw منابع bind خطرناک را به‌طور پیش‌فرض مسدود می‌کند: مسیرهای سیستمی (`/etc`، `/proc`، `/sys`، `/dev`، `/root`، `/boot`)، دایرکتوری‌های socket مربوط به Docker (`/run`، `/var/run` و گونه‌های `docker.sock` آن‌ها) و ریشه‌های رایج اعتبارنامه در دایرکتوری خانه (`~/.aws`، `~/.cargo`، `~/.config`، `~/.docker`، `~/.gnupg`، `~/.netrc`، `~/.npm`، `~/.ssh`).
- اعتبارسنجی ابتدا مسیر منبع را نرمال می‌کند، سپس پیش از بررسی دوباره مسیرهای مسدودشده و ریشه‌های مجاز، آن را از طریق عمیق‌ترین نیای موجود دوباره resolve می‌کند؛ بنابراین گریز از طریق والد symlink حتی وقتی برگ نهایی هنوز وجود ندارد، به‌صورت fail-closed متوقف می‌شود (برای نمونه، اگر `run-link` به آنجا اشاره کند، `/workspace/run-link/new-file` همچنان به‌صورت `/var/run/...` resolve می‌شود).
- مقصدهای bind که نقاط mount رزروشده کانتینر (`/workspace`، `/agent`) را می‌پوشانند نیز به‌طور پیش‌فرض مسدود می‌شوند؛ با `agents.defaults.sandbox.docker.dangerouslyAllowReservedContainerTargets: true` آن را override کنید.
- منابع bind خارج از ریشه‌های مجاز فضای کاری/فضای کاری agent به‌طور پیش‌فرض مسدود می‌شوند؛ با `agents.defaults.sandbox.docker.dangerouslyAllowExternalBindSources: true` آن را override کنید. ریشه‌های مجاز نیز به همان روش canonical می‌شوند؛ بنابراین مسیری که فقط پیش از resolve شدن symlink ظاهراً داخل فهرست مجاز است، همچنان به‌دلیل قرار داشتن خارج از ریشه‌های مجاز رد می‌شود.
- Mountهای حساس (اسرار، کلیدهای SSH، اعتبارنامه‌های سرویس) باید `:ro` باشند، مگر اینکه مطلقاً ضروری باشد.
- اگر فقط به دسترسی خواندن فضای کاری نیاز دارید، آن را با `workspaceAccess: "ro"` ترکیب کنید؛ حالت‌های bind مستقل باقی می‌مانند.
- برای آگاهی از نحوه تعامل bindها با خط‌مشی ابزار و exec ارتقایافته، به [Sandbox در برابر خط‌مشی ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) مراجعه کنید.

</Warning>

## Imageها و راه‌اندازی

Image پیش‌فرض Docker: `openclaw-sandbox:bookworm-slim`

<Note>
**checkout کد منبع در برابر نصب npm**

اسکریپت‌های کمکی `scripts/sandbox-setup.sh`، `scripts/sandbox-common-setup.sh` و `scripts/sandbox-browser-setup.sh` فقط هنگام اجرا از یک [checkout کد منبع](https://github.com/openclaw/openclaw) در دسترس هستند. آن‌ها در بسته npm گنجانده نشده‌اند.

اگر OpenClaw را از طریق `npm install -g openclaw` نصب کرده‌اید، در عوض از فرمان‌های درون‌خطی `docker build` که در ادامه نشان داده شده‌اند استفاده کنید.
</Note>

<Steps>
  <Step title="ساخت image پیش‌فرض">
    از یک checkout کد منبع:

    ```bash
    scripts/sandbox-setup.sh
    ```

    از یک نصب npm (بدون نیاز به checkout کد منبع):

    ```bash
    docker build -t openclaw-sandbox:bookworm-slim - <<'DOCKERFILE'
    FROM debian:bookworm-slim
    ENV DEBIAN_FRONTEND=noninteractive
    RUN apt-get update && apt-get install -y --no-install-recommends \
      bash ca-certificates curl git jq python3 ripgrep \
      && rm -rf /var/lib/apt/lists/*
    RUN useradd --create-home --shell /bin/bash sandbox
    USER sandbox
    WORKDIR /home/sandbox
    CMD ["sleep", "infinity"]
    DOCKERFILE
    ```

    Image پیش‌فرض شامل Node **نیست**. اگر مهارتی به Node (یا runtimeهای دیگر) نیاز دارد، یا یک image سفارشی بسازید یا آن را از طریق `sandbox.docker.setupCommand` نصب کنید (به خروجی شبکه + ریشه قابل‌نوشتن + کاربر root نیاز دارد).

    وقتی `openclaw-sandbox:bookworm-slim` وجود ندارد، OpenClaw به‌طور نامحسوس `debian:bookworm-slim` ساده را جایگزین نمی‌کند. اجراهای sandbox که image پیش‌فرض را هدف می‌گیرند، تا زمانی که آن را بسازید با یک دستورالعمل ساخت فوراً شکست می‌خورند، زیرا image همراه شامل `python3` برای ابزارهای نوشتن/ویرایش sandbox است.

  </Step>
  <Step title="اختیاری: ساخت image عمومی">
    برای یک image کاربردی‌تر sandbox با ابزارهای رایج (برای نمونه `curl`، `jq`، Node 24، pnpm، `python3` و `git`):

    از یک checkout کد منبع:

    ```bash
    scripts/sandbox-common-setup.sh
    ```

    از یک نصب npm، ابتدا image پیش‌فرض را بسازید (بخش بالا را ببینید)، سپس image عمومی را با استفاده از [`scripts/docker/sandbox/Dockerfile.common`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.common) موجود در مخزن روی آن بسازید.

    سپس `agents.defaults.sandbox.docker.image` را روی `openclaw-sandbox-common:bookworm-slim` تنظیم کنید.

  </Step>
  <Step title="اختیاری: ساخت image مرورگر sandbox">
    از یک checkout کد منبع:

    ```bash
    scripts/sandbox-browser-setup.sh
    ```

    از یک نصب npm، با استفاده از [`scripts/docker/sandbox/Dockerfile.browser`](https://github.com/openclaw/openclaw/blob/main/scripts/docker/sandbox/Dockerfile.browser) موجود در مخزن آن را بسازید.

  </Step>
</Steps>

به‌طور پیش‌فرض، کانتینرهای sandbox نوع Docker **هیچ شبکه‌ای** ندارند. با `agents.defaults.sandbox.docker.network` آن را override کنید.

<AccordionGroup>
  <Accordion title="پیش‌فرض‌های Chromium مرورگر sandbox">
    Image همراه مرورگر sandbox، فلگ‌های محافظه‌کارانه راه‌اندازی Chromium را برای بارهای کاری کانتینری اعمال می‌کند:

    - `--remote-debugging-address=127.0.0.1`
    - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
    - `--user-data-dir=${HOME}/.chrome`
    - `--no-first-run`
    - `--no-default-browser-check`
    - `--disable-dev-shm-usage`
    - `--disable-background-networking`
    - `--disable-breakpad`
    - `--disable-crash-reporter`
    - `--no-zygote`
    - `--metrics-recording-only`
    - `--password-store=basic`
    - `--use-mock-keychain`
    - `--headless=new` هنگامی که `browser.headless` فعال است.
    - `--no-sandbox --disable-setuid-sandbox` هنگامی که `browser.noSandbox` فعال است.
    - `--disable-3d-apis`، `--disable-gpu`، `--disable-software-rasterizer` به‌طور پیش‌فرض؛ این پرچم‌های مقاوم‌سازی گرافیکی به کانتینرهای فاقد پشتیبانی GPU کمک می‌کنند. اگر بار کاری به WebGL یا سایر قابلیت‌های سه‌بعدی نیاز دارد، `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` را تنظیم کنید.
    - `--disable-extensions` به‌طور پیش‌فرض؛ برای جریان‌های متکی به افزونه، `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` را تنظیم کنید.
    - `--renderer-process-limit=2` به‌طور پیش‌فرض؛ با `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` کنترل می‌شود که در آن `0` مقدار پیش‌فرض Chromium را حفظ می‌کند.

    اگر به پروفایل زمان اجرای متفاوتی نیاز دارید، از یک ایمیج سفارشی مرورگر استفاده کنید و نقطه ورود خود را ارائه دهید. برای پروفایل‌های محلی Chromium (خارج از کانتینر)، از `browser.extraArgs` برای افزودن پرچم‌های راه‌اندازی بیشتر استفاده کنید.

  </Accordion>
  <Accordion title="پیش‌فرض‌های امنیت شبکه">
    - `network: "host"` مسدود است.
    - `network: "container:<id>"` به‌طور پیش‌فرض مسدود است (خطر دور زدن با پیوستن به فضای نام).
    - نادیده‌گیری اضطراری: `agents.defaults.sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true`.

  </Accordion>
</AccordionGroup>

نصب‌های Docker و Gateway کانتینری در اینجا قرار دارند: [Docker](/fa/install/docker)

برای استقرارهای Docker Gateway، `scripts/docker/setup.sh` می‌تواند پیکربندی سندباکس را راه‌اندازی اولیه کند. برای فعال‌کردن این مسیر، `OPENCLAW_SANDBOX=1` (یا `true`/`yes`/`on`) را تنظیم کنید. مکان سوکت را با `OPENCLAW_DOCKER_SOCKET` بازنویسی کنید. راه‌اندازی کامل و مرجع متغیرهای محیطی: [Docker](/fa/install/docker#agent-sandbox).

## setupCommand (راه‌اندازی یک‌باره کانتینر)

`setupCommand` پس از ایجاد کانتینر سندباکس **یک‌بار** اجرا می‌شود (نه در هر اجرا). این فرمان از طریق `sh -lc` درون کانتینر اجرا می‌شود.

مسیرها:

- سراسری: `agents.defaults.sandbox.docker.setupCommand`
- برای هر عامل: `agents.entries.*.sandbox.docker.setupCommand`

<AccordionGroup>
  <Accordion title="اشتباهات رایج">
    - مقدار پیش‌فرض `docker.network` برابر با `"none"` است (بدون خروجی شبکه)، بنابراین نصب بسته‌ها ناموفق خواهد بود.
    - `docker.network: "container:<id>"` به `dangerouslyAllowContainerNamespaceJoin: true` نیاز دارد و فقط برای شرایط اضطراری است.
    - `readOnlyRoot: true` از نوشتن جلوگیری می‌کند؛ `readOnlyRoot: false` را تنظیم کنید یا یک ایمیج سفارشی بسازید.
    - `user` برای نصب بسته‌ها باید root باشد (`user` را حذف کنید یا `user: "0:0"` را تنظیم کنید).
    - اجرای سندباکس، `process.env` میزبان را به ارث **نمی‌برد**. برای کلیدهای API مهارت‌ها از `agents.defaults.sandbox.docker.env` (یا یک ایمیج سفارشی) استفاده کنید.
    - مقادیر موجود در `agents.defaults.sandbox.docker.env` به‌عنوان متغیرهای محیطی صریح کانتینر Docker ارسال می‌شوند. هر فردی که به دیمون Docker دسترسی داشته باشد می‌تواند آن‌ها را با فرمان‌های فراداده Docker مانند `docker inspect` بررسی کند. اگر این افشای فراداده پذیرفتنی نیست، از یک ایمیج سفارشی، فایل محرمانه متصل‌شده یا مسیر دیگری برای تحویل اطلاعات محرمانه استفاده کنید.

  </Accordion>
</AccordionGroup>

## سیاست ابزار و راه‌های گریز

سیاست‌های مجاز/غیرمجاز ابزار همچنان پیش از قواعد سندباکس اعمال می‌شوند. اگر ابزاری در سطح سراسری یا برای هر عامل غیرمجاز باشد، سندباکس آن را دوباره در دسترس قرار نمی‌دهد.

`tools.elevated` یک راه گریز صریح است که `exec` را خارج از سندباکس اجرا می‌کند (به‌طور پیش‌فرض `gateway`، یا هنگامی که هدف اجرا `node` است، `node`). دستورالعمل‌های `/exec` فقط برای فرستندگان مجاز اعمال می‌شوند و در هر نشست پایدار می‌مانند؛ برای غیرفعال‌سازی قطعی `exec`، از سیاست منع ابزار استفاده کنید (به [سندباکس در برابر سیاست ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) مراجعه کنید).

اشکال‌زدایی:

- `openclaw sandbox list` کانتینرهای سندباکس، وضعیت، تطابق ایمیج، عمر، زمان بیکاری و نشست/عامل مرتبط را نشان می‌دهد.
- `openclaw sandbox explain [--session <key>] [--agent <id>]` حالت مؤثر سندباکس، فضای کاری میزبان، پوشه کاری زمان اجرا، اتصال‌های Docker، سیاست ابزار و کلیدهای پیکربندی اصلاح را بررسی می‌کند. فیلد `workspaceRoot` آن همچنان ریشه پیکربندی‌شده سندباکس است؛ `effectiveHostWorkspaceRoot` نشان می‌دهد فضای کاری فعال واقعاً در کجا قرار دارد.
- `openclaw sandbox recreate [--all | --session <key> | --agent <id>] [--browser] [--force]` کانتینرها/محیط‌ها را حذف می‌کند تا در استفاده بعدی با پیکربندی فعلی دوباره ایجاد شوند.
- برای مدل ذهنی «چرا این مسدود است؟» به [سندباکس در برابر سیاست ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) مراجعه کنید.

## بازنویسی‌های چندعاملی

هر عامل می‌تواند سندباکس و ابزارها را بازنویسی کند: `agents.entries.*.sandbox` و `agents.entries.*.tools` (به‌علاوه `agents.entries.*.tools.sandbox.tools` برای سیاست ابزار سندباکس). برای تقدم، به [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) مراجعه کنید.

## نمونه حداقلی فعال‌سازی

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main",
        scope: "session",
        workspaceAccess: "none",
      },
    },
  },
}
```

## مرتبط

- [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) -- بازنویسی‌های مختص هر عامل و تقدم
- [OpenShell](/fa/gateway/openshell) -- راه‌اندازی بک‌اند مدیریت‌شده سندباکس، حالت‌های فضای کاری و مرجع پیکربندی
- [پیکربندی سندباکس](/fa/gateway/config-agents#agentsdefaultssandbox)
- [سندباکس در برابر سیاست ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) -- اشکال‌زدایی «چرا این مسدود است؟»
- [امنیت](/fa/gateway/security)
