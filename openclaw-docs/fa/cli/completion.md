---
read_when:
    - تکمیل خودکار پوسته برای zsh/bash/fish/PowerShell می‌خواهید
    - باید اسکریپت‌های تکمیل را در وضعیت OpenClaw ذخیره‌سازی موقت کنید
summary: مرجع CLI برای `openclaw completion` (تولید/نصب اسکریپت‌های تکمیل خودکار پوسته)
title: تکمیل
x-i18n:
    generated_at: "2026-07-27T15:00:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 67cb52a47036745150887c752d18e2dfa84fab2722c27c696142d23080bb2efd
    source_path: cli/completion.md
    workflow: 16
---

# `openclaw completion`

اسکریپت‌های تکمیل پوسته را تولید کنید، آن‌ها را در وضعیت OpenClaw کش کنید و در صورت تمایل در پروفایل پوستهٔ خود نصب کنید.

## نحوهٔ استفاده

```bash
openclaw completion                          # اسکریپت zsh را در خروجی استاندارد چاپ می‌کند
openclaw completion --shell fish             # اسکریپت fish را چاپ می‌کند
openclaw completion --write-state            # اسکریپت‌های همهٔ پوسته‌ها را کش می‌کند
openclaw completion --write-state --install  # کش و سپس در یک مرحله نصب می‌کند
openclaw completion --shell bash --write-state
```

## گزینه‌ها

- `-s, --shell <shell>`: پوستهٔ مقصد (`zsh`، `bash`، `powershell`، `fish`؛ پیش‌فرض: `zsh`)
- `-i, --install`: تکمیل را با افزودن یک خط source برای اسکریپت کش‌شده به پروفایل پوستهٔ خود نصب کنید
- `--write-state`: اسکریپت(های) تکمیل را بدون چاپ در خروجی استاندارد در `$OPENCLAW_STATE_DIR/completions` (پیش‌فرض `~/.openclaw/completions`) بنویسید؛ همراه با `--shell` فقط برای همان پوسته می‌نویسد، در غیر این صورت برای هر چهار پوسته
- `-y, --yes`: از اعلان‌های تأیید نصب صرف‌نظر کنید (غیرتعاملی)

## روند نصب

`--install` پروفایل شما را به اسکریپت کش‌شده ارجاع می‌دهد، بنابراین کش باید ابتدا وجود داشته باشد: اگر موجود نباشد، فرمان ناموفق می‌شود و از شما می‌خواهد `openclaw completion --write-state` را اجرا کنید. برای انجام هر دو کار در یک مرحله، آن را با `--write-state --install` ترکیب کنید. بدون `--shell`، ‏`--install` پوسته را از `$SHELL` تشخیص می‌دهد (و در صورت ناموفق‌بودن، از zsh استفاده می‌کند).

نصب، یک بلوک کوچک `# OpenClaw Completion` را در پروفایل پوستهٔ شما می‌نویسد و هر خط قدیمی و کند `source <(openclaw completion ...)` را با خط source کش‌شده جایگزین می‌کند:

| پوسته      | پروفایل                                                                                                                                                                                    |
| ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| bash       | `~/.bashrc` (وقتی `~/.bashrc` موجود نیست، از `~/.bash_profile` استفاده می‌کند)                                                                                                                  |
| fish       | `~/.config/fish/config.fish`                                                                                                                                                               |
| powershell | `~/.config/powershell/Microsoft.PowerShell_profile.ps1` (در Windows: ‏`Documents/PowerShell/Microsoft.PowerShell_profile.ps1`، یا `Documents/WindowsPowerShell/...` برای Windows PowerShell) |
| zsh        | `~/.zshrc`                                                                                                                                                                                 |

## نکات

- بدون `--install` یا `--write-state`، فرمان اسکریپت را در خروجی استاندارد چاپ می‌کند.
- تولید تکمیل، کل درخت فرمان را از پیش بارگیری می‌کند، از جمله فرمان‌های CLI مربوط به Plugin، بنابراین زیرفرمان‌های تودرتو نیز گنجانده می‌شوند.
- `openclaw update` پس از به‌روزرسانی موفق، کش تکمیل را به‌طور خودکار تازه‌سازی می‌کند؛ `openclaw doctor` می‌تواند تنظیمات تکمیل مفقود یا منسوخ را ترمیم کند.

## مرتبط

- [مرجع CLI](/fa/cli)
