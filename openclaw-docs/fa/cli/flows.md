---
read_when:
    - در مستندات قدیمی‌تر یا یادداشت‌های انتشار با `openclaw flows` مواجه می‌شوید
    - یک مرجع سریع برای بررسی TaskFlow می‌خواهید
summary: 'تغییر مسیر: فرمان‌های جریان در `openclaw tasks flow` قرار دارند'
title: جریان‌ها (تغییر مسیر)
x-i18n:
    generated_at: "2026-07-27T14:59:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 05d27154190d6087649612d81ce15f0cbc9459aa89ab22211582c18f4fc2943c
    source_path: cli/flows.md
    workflow: 16
---

# `openclaw tasks flow`

هیچ فرمان سطح‌بالای `openclaw flows` وجود ندارد. بازرسی پایدار TaskFlow در زیرمجموعهٔ `openclaw tasks flow` قرار دارد.

## زیرفرمان‌ها

```bash
openclaw tasks flow list   [--json] [--status <name>]
openclaw tasks flow show   <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

| زیرفرمان | توضیحات                | آرگومان‌ها / گزینه‌ها                                                                   |
| ---------- | -------------------------- | ------------------------------------------------------------------------------------- |
| `list`     | TaskFlowهای ردیابی‌شده را فهرست می‌کند.    | خروجی قابل‌خواندن برای ماشین با `--json`؛ فیلتر `--status <name>` (مقادیر وضعیت را در ادامه ببینید). |
| `show`     | یک TaskFlow را نمایش می‌دهد.         | شناسهٔ جریان یا کلید مالک در `<lookup>`؛ خروجی قابل‌خواندن برای ماشین با `--json`.                    |
| `cancel`   | یک TaskFlow در حال اجرا را لغو می‌کند. | شناسهٔ جریان یا کلید مالک در `<lookup>`.                                                      |

`<lookup>` یا یک شناسهٔ جریان (که توسط `list` / `show` بازگردانده می‌شود) یا کلید مالک جریان (شناسهٔ پایداری که زیرسامانهٔ مالک برای ردیابی جریان استفاده می‌کند) را می‌پذیرد.

### مقادیر فیلتر وضعیت

`--status` در `list` یکی از این مقادیر را می‌پذیرد: `queued`، `running`، `waiting`، `blocked`، `succeeded`، `failed`، `cancelled`، `lost`.

## مثال‌ها

```bash
openclaw tasks flow list
openclaw tasks flow list --status running
openclaw tasks flow list --json
openclaw tasks flow show flow_abc123
openclaw tasks flow show flow_abc123 --json
openclaw tasks flow cancel flow_abc123
```

برای مفاهیم و نگارش TaskFlow، به [TaskFlow](/fa/automation/taskflow) مراجعه کنید. برای فرمان والد `tasks`، [مرجع CLI مربوط به tasks](/fa/cli/tasks) را ببینید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [خودکارسازی](/fa/automation)
- [TaskFlow](/fa/automation/taskflow)
