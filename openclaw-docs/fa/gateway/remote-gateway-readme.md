---
read_when: Connecting the macOS app to a remote gateway over SSH
summary: راه‌اندازی تونل SSH برای اتصال OpenClaw.app به یک Gateway راه دور
title: راه‌اندازی Gateway راه‌دور
x-i18n:
    generated_at: "2026-07-27T16:35:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 842578eb74e99d115b04abff5e9673a6454fa6d2cf7905d056999469e1c6b66d
    source_path: gateway/remote-gateway-readme.md
    workflow: 16
---

<Note>
این محتوا اکنون در [دسترسی از راه دور](/fa/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) قرار دارد. برای راهنمای فعلی از آن صفحه استفاده کنید؛ این صفحه به‌عنوان مقصد تغییرمسیر باقی می‌ماند.
</Note>

# اجرای OpenClaw.app با یک Gateway راه دور

OpenClaw.app از طریق یک تونل SSH به Gateway راه دور دسترسی پیدا می‌کند: یک SSH `LocalForward` یک درگاه محلی را به درگاه WebSocket متعلق به Gateway روی میزبان راه دور نگاشت می‌کند.

```mermaid
flowchart TB
    subgraph Client["دستگاه کارخواه"]
        direction TB
        A["OpenClaw.app"]
        B["ws://127.0.0.1:18789\n(درگاه محلی)"]
        T["تونل SSH"]

        A --> B
        B --> T
    end
    subgraph Remote["دستگاه راه دور"]
        direction TB
        C["WebSocket متعلق به Gateway"]
        D["ws://127.0.0.1:18789"]

        C --> D
    end
    T --> C
```

## راه‌اندازی

1. یک ورودی پیکربندی SSH با `LocalForward 18789 127.0.0.1:18789` اضافه کنید (برای بلوک کامل پیکربندی، به [دسترسی از راه دور](/fa/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) مراجعه کنید).
2. کلید SSH خود را با `ssh-copy-id` در میزبان راه دور کپی کنید.
3. مقدار `gateway.remote.token` (یا `gateway.remote.password`) را از طریق `openclaw config set gateway.remote.token "<your-token>"` تنظیم کنید.
4. تونل را راه‌اندازی کنید: `ssh -N remote-gateway &`.
5. از OpenClaw.app خارج شوید و دوباره آن را باز کنید.

برای تونلی که پس از راه‌اندازی مجدد سیستم همچنان فعال می‌ماند و به‌طور خودکار دوباره متصل می‌شود، به‌جای `ssh -N` دستی، از راه‌اندازی LaunchAgent در صفحه [دسترسی از راه دور](/fa/gateway/remote#macos-persistent-ssh-tunnel-via-launchagent) استفاده کنید.

## نحوه کارکرد

| مؤلفه                            | کاری که انجام می‌دهد                                                  |
| ------------------------------------ | ------------------------------------------------------------- |
| `LocalForward 18789 127.0.0.1:18789` | درگاه محلی 18789 را به درگاه راه دور 18789 هدایت می‌کند                |
| `ssh -N`                             | SSH بدون اجرای فرمان‌های راه دور (فقط هدایت درگاه)  |
| `KeepAlive`                          | در صورت از کار افتادن تونل، آن را به‌طور خودکار دوباره راه‌اندازی می‌کند (LaunchAgent) |
| `RunAtLoad`                          | هنگام بارگذاری LaunchAgent، تونل را راه‌اندازی می‌کند (LaunchAgent)    |

OpenClaw.app در دستگاه کارخواه به `ws://127.0.0.1:18789` متصل می‌شود. تونل آن اتصال را به درگاه 18789 در میزبان راه دوری که Gateway را اجرا می‌کند هدایت می‌کند.

## مرتبط

- [دسترسی از راه دور](/fa/gateway/remote)
- [Tailscale](/fa/gateway/tailscale)
