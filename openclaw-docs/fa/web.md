---
read_when:
    - می‌خواهید از طریق Tailscale به Gateway دسترسی پیدا کنید
    - رابط کاربری Control مرورگر و ویرایش پیکربندی را می‌خواهید
summary: 'سطوح وب Gateway: رابط کاربری کنترل، حالت‌های اتصال و امنیت'
title: وب
x-i18n:
    generated_at: "2026-07-27T14:58:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 413fb029d95241f5c6043b28825727cdee52b2fa8cbe998fbbd6e3ff7b81467b
    source_path: web/index.md
    workflow: 16
---

Gateway یک **رابط کاربری کنترل در مرورگر** کوچک (Vite + Lit) را از همان پورت WebSocket مربوط به Gateway ارائه می‌کند:

- پیش‌فرض: `http://<host>:18789/`
- با `gateway.tls.enabled: true`: `https://<host>:18789/`
- پیشوند اختیاری: `gateway.controlUi.basePath` را تنظیم کنید (برای مثال، `/openclaw`)

قابلیت‌ها در [رابط کاربری کنترل](/fa/web/control-ui) توضیح داده شده‌اند. این صفحه حالت‌های اتصال، امنیت و دیگر سطوح تحت وب را پوشش می‌دهد.

## پیکربندی (به‌طور پیش‌فرض فعال)

رابط کاربری کنترل در صورت وجود دارایی‌ها (`dist/control-ui`) **به‌طور پیش‌فرض فعال است**:

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath اختیاری است
  },
}
```

## Webhookها

وقتی `hooks.enabled=true`، Gateway یک نقطه پایانی Webhook را نیز روی همان سرور HTTP ارائه می‌کند. برای احراز هویت و بارهای داده، `hooks` را در [مرجع پیکربندی Gateway](/fa/gateway/configuration-reference#hooks) ببینید.

## RPC مدیریتی HTTP

`POST /api/v1/admin/rpc` روش‌های منتخب صفحه کنترل Gateway را از طریق HTTP ارائه می‌کند. به‌طور پیش‌فرض غیرفعال است و فقط وقتی Plugin مربوط به `admin-http-rpc` فعال باشد ثبت می‌شود. برای مدل احراز هویت، روش‌های مجاز و مقایسه با API وب‌سوکت، [RPC مدیریتی HTTP](/fa/plugins/admin-http-rpc) را ببینید.

## دسترسی Tailscale

<Tabs>
  <Tab title="Serve یکپارچه (توصیه‌شده)">
    Gateway را روی loopback نگه دارید و اجازه دهید Tailscale Serve آن را پروکسی کند:

    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "serve" },
      },
    }
    ```

    Gateway را راه‌اندازی کنید:

    ```bash
    openclaw gateway
    ```

    `https://<magicdns>/` (یا `gateway.controlUi.basePath` پیکربندی‌شده خود) را باز کنید.

  </Tab>
  <Tab title="اتصال Tailnet + توکن">
    ```json5
    {
      gateway: {
        bind: "tailnet",
        controlUi: { enabled: true },
        auth: { mode: "token", token: "your-token" },
      },
    }
    ```

    Gateway را راه‌اندازی کنید (این نمونه غیر-loopback از احراز هویت توکنی با راز مشترک استفاده می‌کند):

    ```bash
    openclaw gateway
    ```

    `http://<tailscale-ip>:18789/` (یا `gateway.controlUi.basePath` پیکربندی‌شده خود) را باز کنید.

  </Tab>
  <Tab title="اینترنت عمومی (Funnel)">
    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "funnel" },
        auth: { mode: "password" }, // یا OPENCLAW_GATEWAY_PASSWORD
      },
    }
    ```

    `tailscale.mode: "funnel"` به `gateway.auth.mode: "password"` نیاز دارد؛ Serve و Funnel هر دو به `gateway.bind: "loopback"` نیاز دارند.

  </Tab>
</Tabs>

## نکات امنیتی

- احراز هویت Gateway به‌طور پیش‌فرض الزامی است: توکن، گذرواژه، پروکسی مورد اعتماد، یا در صورت فعال بودن، سرآیندهای هویت Tailscale Serve.
- اتصال‌های غیر-loopback همچنان به احراز هویت Gateway **نیاز دارند**: احراز هویت با توکن/گذرواژه یا یک پروکسی معکوس آگاه از هویت همراه با `gateway.auth.mode: "trusted-proxy"`.
- ویزارد راه‌اندازی اولیه به‌طور پیش‌فرض احراز هویت با راز مشترک ایجاد می‌کند و معمولاً حتی روی loopback نیز یک توکن Gateway تولید می‌کند.
- در حالت راز مشترک، رابط کاربری هنگام دست‌دهی WebSocket، `connect.params.auth.token` یا `connect.params.auth.password` را ارسال می‌کند.
- با `gateway.tls.enabled: true`، ابزارهای کمکی محلی داشبورد/وضعیت، نشانی‌های `https://` و نشانی‌های WebSocket مربوط به `wss://` را نمایش می‌دهند.
- در حالت‌های دارای هویت (Tailscale Serve، `trusted-proxy`)، بررسی احراز هویت WebSocket به‌جای راز مشترک از طریق سرآیندهای درخواست انجام می‌شود.
- برای استقرارهای عمومی و غیر-loopback رابط کاربری کنترل، `gateway.controlUi.allowedOrigins` را به‌صراحت تنظیم کنید (مبدأهای کامل). بارگذاری‌های خصوصی هم‌مبدأ برای loopback، RFC1918/link-local، `.local`، `.ts.net` و میزبان‌های CGNAT مربوط به Tailscale بدون آن پذیرفته می‌شوند.
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true` بازگشت به مبدأ مبتنی بر سرآیند Host را فعال می‌کند؛ این یک تنزل امنیتی خطرناک است.
- با Serve، وقتی `gateway.auth.allowTailscale: true`، سرآیندهای هویت Tailscale احراز هویت رابط کاربری کنترل/WebSocket را برآورده می‌کنند (بدون نیاز به توکن/گذرواژه). نقاط پایانی API مربوط به HTTP از سرآیندهای هویت Tailscale استفاده نمی‌کنند؛ آن‌ها همیشه از حالت عادی احراز هویت HTTP در Gateway پیروی می‌کنند. برای الزام اعتبارنامه‌های صریح حتی از طریق Serve، `gateway.auth.allowTailscale: false` را تنظیم کنید. این جریان بدون توکن فرض می‌کند خود میزبان Gateway مورد اعتماد است. [Tailscale](/fa/gateway/tailscale) و [امنیت](/fa/gateway/security) را ببینید.

## ساخت رابط کاربری

Gateway فایل‌های ایستا را از `dist/control-ui` ارائه می‌کند:

```bash
pnpm ui:build
```
