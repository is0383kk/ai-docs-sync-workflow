---
read_when:
    - می‌خواهید OpenClaw را روی یک کلاستر Kubernetes اجرا کنید
    - می‌خواهید OpenClaw را در یک محیط Kubernetes آزمایش کنید
summary: استقرار Gateway‏ OpenClaw در یک کلاستر Kubernetes با Kustomize
title: کوبرنتیز
x-i18n:
    generated_at: "2026-07-27T14:16:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c05eb0eb923fa1f515aca1f6dcb6073aba69af0bdf30233243027edfedd45a39
    source_path: install/kubernetes.md
    workflow: 16
---

نقطه شروعی حداقلی برای اجرای OpenClaw روی Kubernetes، نه استقراری آماده برای محیط عملیاتی. این راهنما منابع اصلی را پوشش می‌دهد و برای سازگارشدن با محیط شما در نظر گرفته شده است.

## چرا Helm نه

OpenClaw یک کانتینر منفرد با چند فایل پیکربندی است. سفارشی‌سازی مهم در محتوای عامل (فایل‌های Markdown، مهارت‌ها و بازنویسی‌های پیکربندی) انجام می‌شود، نه در قالب‌بندی زیرساخت. Kustomize هم‌پوشانی‌ها را بدون سربار یک چارت Helm مدیریت می‌کند. اگر استقرار شما پیچیده‌تر شد، یک چارت Helm را روی این مانیفست‌ها قرار دهید.

## موارد موردنیاز

- یک کلاستر Kubernetes در حال اجرا (AKS، EKS، GKE، k3s، kind، OpenShift و غیره)
- `kubectl` متصل به کلاستر شما
- یک کلید API برای دست‌کم یک ارائه‌دهنده مدل

## شروع سریع

```bash
# با ارائه‌دهنده خود جایگزین کنید: ANTHROPIC، GEMINI، OPENAI یا OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

`deploy.sh` به‌طور پیش‌فرض احراز هویت مبتنی بر توکن ایجاد می‌کند. توکن Gateway تولیدشده را برای رابط کاربری کنترل دریافت کنید:

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

برای اشکال‌زدایی محلی، `./scripts/k8s/deploy.sh --show-token` پس از استقرار توکن را چاپ می‌کند.

## آزمایش محلی با Kind

اگر کلاستر ندارید، با [Kind](https://kind.sigs.k8s.io/) یکی را به‌صورت محلی ایجاد کنید:

```bash
./scripts/k8s/create-kind.sh           # docker یا podman را به‌طور خودکار تشخیص می‌دهد
./scripts/k8s/create-kind.sh --delete  # برچیدن
```

سپس طبق معمول با `./scripts/k8s/deploy.sh` استقرار را انجام دهید.

## گام‌به‌گام

### 1) استقرار

**گزینه A: کلید API در محیط (یک مرحله)**

```bash
# با ارائه‌دهنده خود جایگزین کنید: ANTHROPIC، GEMINI، OPENAI یا OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

اسکریپت یک Secret در Kubernetes با کلید API و یک توکن Gateway تولیدشده به‌صورت خودکار ایجاد می‌کند و سپس استقرار را انجام می‌دهد. اگر Secret از قبل وجود داشته باشد، توکن Gateway فعلی و همه کلیدهای ارائه‌دهندگانی را که تغییر نمی‌کنند حفظ می‌کند.

**گزینه B: ایجاد Secret به‌صورت جداگانه**

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

برای چاپ توکن در stdout جهت آزمایش محلی، `--show-token` را به هرکدام از فرمان‌ها اضافه کنید.

### 2) دسترسی به Gateway

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## مواردی که مستقر می‌شوند

```text
فضای نام: openclaw (قابل پیکربندی از طریق OPENCLAW_NAMESPACE)
├── Deployment/openclaw        # یک پاد، کانتینر راه‌انداز + Gateway
├── Service/openclaw           # ClusterIP روی پورت 18789
├── PersistentVolumeClaim      # 10Gi برای وضعیت و پیکربندی عامل
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # توکن Gateway + کلیدهای API
```

## سفارشی‌سازی

### دستورالعمل‌های عامل

`AGENTS.md` را در `scripts/k8s/manifests/configmap.yaml` ویرایش و دوباره مستقر کنید:

```bash
./scripts/k8s/deploy.sh
```

### پیکربندی Gateway

`openclaw.json` را در `scripts/k8s/manifests/configmap.yaml` ویرایش کنید. برای مرجع کامل، به [پیکربندی Gateway](/fa/gateway/configuration) مراجعه کنید.

### افزودن ارائه‌دهندگان

با صادرکردن کلیدهای بیشتر، دوباره اجرا کنید:

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

کلیدهای ارائه‌دهندگان موجود در Secret باقی می‌مانند، مگر اینکه آن‌ها را بازنویسی کنید.

یا Secret را مستقیماً وصله کنید:

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### فضای نام سفارشی

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### ایمیج سفارشی

فیلد `image` را در `scripts/k8s/manifests/deployment.yaml` ویرایش کنید:

```yaml
image: ghcr.io/openclaw/openclaw:slim # اصلی؛ آینه رسمی Docker Hub: openclaw/openclaw
```

### در معرض دسترسی قرار دادن فراتر از انتقال پورت

مانیفست‌های پیش‌فرض، Gateway را داخل پاد به رابط حلقه‌بازگشت متصل می‌کنند. این تنظیم با `kubectl port-forward` کار می‌کند، اما با مسیر Kubernetes `Service` یا Ingress که باید مستقیماً به IP پاد دسترسی پیدا کند، کار نمی‌کند.

برای در معرض دسترسی قرار دادن Gateway از طریق Ingress یا متعادل‌کننده بار:

- اتصال Gateway را در `scripts/k8s/manifests/configmap.yaml` از `loopback` به یک اتصال غیرحلقه‌بازگشت که با مدل استقرار شما مطابقت دارد تغییر دهید.
- احراز هویت Gateway را فعال نگه دارید و از یک نقطه ورود مناسب با خاتمه TLS استفاده کنید.
- رابط کاربری کنترل را با استفاده از مدل امنیتی پشتیبانی‌شده وب برای دسترسی راه دور پیکربندی کنید (برای نمونه HTTPS/Tailscale Serve و مبدأهای مجاز صریح در صورت نیاز).

## استقرار مجدد

```bash
./scripts/k8s/deploy.sh
```

این کار همه مانیفست‌ها را اعمال و پاد را راه‌اندازی مجدد می‌کند تا هرگونه تغییر پیکربندی یا Secret اعمال شود.

## برچیدن

```bash
./scripts/k8s/deploy.sh --delete
```

این فرمان فضای نام و همه منابع درون آن، از جمله PVC، را حذف می‌کند.

## نکات معماری

- Gateway به‌طور پیش‌فرض داخل پاد به رابط حلقه‌بازگشت متصل می‌شود؛ بنابراین راه‌اندازی ارائه‌شده برای `kubectl port-forward` است.
- هیچ منبعی در سطح کلاستر وجود ندارد؛ همه‌چیز در یک فضای نام واحد قرار دارد.
- سخت‌سازی امنیتی: قابلیت‌های `readOnlyRootFilesystem`، `drop: ALL`، کاربر غیرریشه (UID 1000).
- پیکربندی پیش‌فرض، رابط کاربری کنترل را در مسیر امن‌تر دسترسی محلی نگه می‌دارد: اتصال حلقه‌بازگشت به‌همراه `kubectl port-forward` روی `http://127.0.0.1:18789`.
- اگر از دسترسی localhost فراتر می‌روید، از مدل راه دور پشتیبانی‌شده استفاده کنید: HTTPS/Tailscale به‌همراه تنظیمات مناسب اتصال Gateway و مبدأ رابط کاربری کنترل.
- Secretها در یک دایرکتوری موقت تولید و مستقیماً روی کلاستر اعمال می‌شوند؛ هیچ داده محرمانه‌ای در نسخه کاری مخزن نوشته نمی‌شود.

## ساختار فایل‌ها

```text
scripts/k8s/
├── deploy.sh                   # فضای نام و Secret را ایجاد می‌کند و از طریق kustomize مستقر می‌کند
├── create-kind.sh              # کلاستر محلی Kind (تشخیص خودکار docker/podman)
└── manifests/
    ├── kustomization.yaml      # پایه Kustomize
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # مشخصات پاد با سخت‌سازی امنیتی
    ├── pvc.yaml                # فضای ذخیره‌سازی پایدار 10Gi
    └── service.yaml            # ClusterIP روی 18789
```

## مرتبط

- [Docker](/fa/install/docker)
- [محیط اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime)
- [نمای کلی نصب](/fa/install)
