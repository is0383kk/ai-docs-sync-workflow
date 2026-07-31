---
read_when:
    - می‌خواهید Prometheus، Grafana، VictoriaMetrics یا اسکرپر دیگری متریک‌های Gateway در OpenClaw را جمع‌آوری کند
    - برای داشبوردها یا هشدارها به نام‌های متریک Prometheus و خط‌مشی برچسب‌ها نیاز دارید
    - می‌خواهید بدون اجرای گردآورندهٔ OpenTelemetry، متریک‌ها را دریافت کنید
sidebarTitle: Prometheus
summary: ارائهٔ داده‌های تشخیصی OpenClaw به‌صورت معیارهای متنی Prometheus از طریق Plugin diagnostics-prometheus
title: معیارهای Prometheus
x-i18n:
    generated_at: "2026-07-27T15:31:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d04a46bdb401df3cdd2571b973f2a60f264862cf74da02c5a9cfa1de6ea9ffe
    source_path: gateway/prometheus.md
    workflow: 16
---

OpenClaw می‌تواند سنجه‌های عیب‌یابی را از طریق Plugin رسمی
`diagnostics-prometheus` ارائه کند. این Plugin به عیب‌یابی‌های مورد اعتماد و رویدادهای
عیب‌یابیِ دارای برچسب داخلی و تحت مالکیت توزیع‌کننده (سیگنال‌های صف، حافظه و
بازیابی نشست) گوش می‌دهد و یک نقطه پایانی متنی Prometheus را در نشانی زیر ارائه می‌کند:

```text
GET /api/diagnostics/prometheus
```

نوع محتوا `text/plain; version=0.0.4; charset=utf-8`، یعنی قالب استاندارد
ارائه Prometheus است.

<Warning>
این مسیر از احراز هویت Gateway استفاده می‌کند (محدوده اپراتور، سطح اپراتور مورد اعتماد). آن را به‌عنوان یک نقطه پایانی عمومی و بدون احراز هویت `/metrics` در دسترس قرار ندهید. آن را از طریق همان مسیر احراز هویتی جمع‌آوری کنید که برای سایر APIهای اپراتور استفاده می‌کنید.
</Warning>

برای ردگیری‌ها، گزارش‌ها، ارسال OTLP و ویژگی‌های معنایی OpenTelemetry GenAI، به [خروجی OpenTelemetry](/fa/gateway/opentelemetry) مراجعه کنید.

## شروع سریع

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install clawhub:@openclaw/diagnostics-prometheus
    ```
  </Step>
  <Step title="فعال‌کردن Plugin">
    <Tabs>
      <Tab title="پیکربندی">
        ```json5
        {
          plugins: {
            allow: ["diagnostics-prometheus"],
            entries: {
              "diagnostics-prometheus": { enabled: true },
            },
          },
          diagnostics: {
            enabled: true,
          },
        }
        ```
      </Tab>
      <Tab title="CLI">
        ```bash
        openclaw plugins enable diagnostics-prometheus
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="راه‌اندازی مجدد Gateway">
    مسیر HTTP هنگام راه‌اندازی Plugin ثبت می‌شود؛ بنابراین پس از فعال‌سازی، آن را بارگذاری مجدد کنید.
  </Step>
  <Step title="جمع‌آوری از مسیر محافظت‌شده">
    همان احراز هویت Gateway را ارسال کنید که کلاینت‌های اپراتور شما استفاده می‌کنند:

    ```bash
    curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
      http://127.0.0.1:18789/api/diagnostics/prometheus
    ```

  </Step>
  <Step title="اتصال Prometheus">
    ```yaml
    # prometheus.yml
    scrape_configs:
      - job_name: openclaw
        scrape_interval: 30s
        metrics_path: /api/diagnostics/prometheus
        authorization:
          credentials_file: /etc/prometheus/openclaw-gateway-token
        static_configs:
          - targets: ["openclaw-gateway:18789"]
    ```
  </Step>
</Steps>

<Note>
مقدار پیش‌فرض `diagnostics.enabled` برابر با `true` است؛ آن را فقط در محیط‌هایی با محدودیت‌های سخت‌گیرانه روی `false` تنظیم کنید. اگر مقدار آن `false` باشد، Plugin همچنان مسیر HTTP را ثبت می‌کند، اما هیچ رویداد عیب‌یابی وارد صادرکننده نمی‌شود؛ بنابراین پاسخ خالی است.
</Note>

## سنجه‌های صادرشده

| سنجه                                            | نوع       | برچسب‌ها                                                                                  |
| ------------------------------------------------ | --------- | ----------------------------------------------------------------------------------------- |
| `openclaw_run_completed_total`                   | شمارنده   | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_run_duration_seconds`                  | هیستوگرام | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_model_call_total`                      | شمارنده   | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_call_duration_seconds`           | هیستوگرام | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_failover_total`                  | شمارنده   | `from_model`, `from_provider`, `lane`, `reason`, `suspended`, `to_model`, `to_provider`   |
| `openclaw_model_tokens_total`                    | شمارنده   | `agent`, `channel`, `model`, `provider`, `token_type`                                     |
| `openclaw_gen_ai_client_token_usage`             | هیستوگرام | `model`, `provider`, `token_type`                                                         |
| `openclaw_model_cost_usd_total`                  | شمارنده   | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_model_usage_duration_seconds`          | هیستوگرام | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_skill_used_total`                      | شمارنده   | `activation`, `agent`, `skill`, `source`                                                  |
| `openclaw_tool_execution_total`                  | شمارنده   | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_duration_seconds`       | هیستوگرام | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_blocked_total`          | شمارنده   | `denied_reason`, `params_kind`, `tool`, `tool_owner`, `tool_source`                       |
| `openclaw_harness_run_total`                     | شمارنده   | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_harness_run_duration_seconds`          | هیستوگرام | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_webhook_received_total`                | شمارنده   | `channel`, `webhook`                                                                      |
| `openclaw_webhook_error_total`                   | شمارنده   | `channel`, `webhook`                                                                      |
| `openclaw_webhook_duration_seconds`              | هیستوگرام | `channel`, `webhook`                                                                      |
| `openclaw_message_received_total`                | شمارنده   | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_started_total`        | شمارنده   | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_completed_total`      | شمارنده   | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_dispatch_duration_seconds`     | هیستوگرام | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_processed_total`               | شمارنده   | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_processed_duration_seconds`    | هیستوگرام | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_delivery_started_total`        | شمارنده   | `channel`, `delivery_kind`                                                                |
| `openclaw_message_delivery_total`                | شمارنده   | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_message_delivery_duration_seconds`     | هیستوگرام | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_talk_event_total`                      | شمارنده   | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_event_duration_seconds`           | هیستوگرام | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_audio_bytes`                      | هیستوگرام | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_queue_lane_size`                       | سنجه      | `lane`                                                                                    |
| `openclaw_queue_lane_wait_seconds`               | هیستوگرام | `lane`                                                                                    |
| `openclaw_session_state_total`                   | شمارنده   | `reason`, `state`                                                                         |
| `openclaw_session_queue_depth`                   | سنجه      | `state`                                                                                   |
| `openclaw_session_turn_created_total`            | شمارنده   | `agent`, `channel`, `trigger`                                                             |
| `openclaw_session_stuck_total`                   | شمارنده   | `reason`, `state`                                                                         |
| `openclaw_session_stuck_age_seconds`             | هیستوگرام | `reason`, `state`                                                                         |
| `openclaw_session_recovery_total`                | شمارنده   | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_session_recovery_age_seconds`          | هیستوگرام | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_liveness_warning_total`                | شمارنده   | `reason`                                                                                  |
| `openclaw_liveness_sessions`                     | سنجه      | `state`                                                                                   |
| `openclaw_liveness_event_loop_delay_p99_seconds` | هیستوگرام | `reason`                                                                                  |
| `openclaw_liveness_event_loop_delay_max_seconds` | هیستوگرام | `reason`                                                                                  |
| `openclaw_liveness_event_loop_utilization_ratio` | هیستوگرام | `reason`                                                                                  |
| `openclaw_liveness_cpu_core_ratio`               | هیستوگرام | `reason`                                                                                  |
| `openclaw_payload_large_total`                   | شمارنده   | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_payload_large_bytes`                   | هیستوگرام | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_memory_bytes`                          | سنجه      | `kind`                                                                                    |
| `openclaw_memory_rss_bytes`                      | هیستوگرام | هیچ‌کدام                                                                                  |
| `openclaw_memory_pressure_total`                 | شمارنده   | `level`, `reason`                                                                         |
| `openclaw_telemetry_exporter_total`              | شمارنده   | `exporter`, `reason`, `signal`, `status`                                                  |
| `openclaw_prometheus_series_dropped_total`       | شمارنده   | هیچ‌کدام                                                                                  |
| `openclaw_diagnostic_async_queue_dropped_total`  | شمارنده   | `drop_class`                                                                              |
| `openclaw_diagnostic_async_queue_length`         | سنجه      | هیچ‌کدام                                                                                  |

برای سنجه‌های فراخوانی مدل، `observation_unit="request"` یک درخواست مشاهده‌پذیر
به ارائه‌دهنده را اندازه‌گیری می‌کند. `observation_unit="turn"` یک نوبت مصنوعی عامل Claude Code
یا Codex CLI را اندازه‌گیری می‌کند که می‌تواند شامل چندین درخواست پنهان به ارائه‌دهنده باشد.
هنگام مقایسه تأخیر، این سری‌ها را جدا نگه دارید.

## سیاست برچسب‌ها

<AccordionGroup>
  <Accordion title="برچسب‌های محدود با کاردینالیتی پایین">
    برچسب‌های Prometheus محدود و دارای کاردینالیتی پایین باقی می‌مانند. صادرکننده شناسه‌های تشخیصی خام مانند `runId`، `sessionKey`، `sessionId`، `callId`، `toolCallId`، شناسه‌های پیام، شناسه‌های گفت‌وگو یا شناسه‌های درخواست ارائه‌دهنده را منتشر نمی‌کند.

    مقادیر برچسب‌ها حذف حساسیت می‌شوند و باید با سیاست نویسه‌های دارای کاردینالیتی پایین OpenClaw مطابقت داشته باشند. مقادیری که با این سیاست مطابقت ندارند، بسته به سنجه با `unknown`، `other` یا `none` جایگزین می‌شوند. برچسب‌هایی که شبیه کلیدهای نشست عاملِ دارای دامنه هستند نیز با `unknown` جایگزین می‌شوند.

  </Accordion>
  <Accordion title="سقف سری‌ها و محاسبه سرریز">
    صادرکننده تعداد سری‌های زمانی نگه‌داری‌شده در حافظه را در مجموع شمارنده‌ها، پیمانه‌ها و هیستوگرام‌ها به **2048** سری محدود می‌کند. سری‌های جدید فراتر از این سقف حذف می‌شوند و `openclaw_prometheus_series_dropped_total` هر بار یک واحد افزایش می‌یابد.

    این شمارنده را به‌عنوان نشانه‌ای قطعی زیر نظر بگیرید که یک ویژگی در بالادست مقادیر با کاردینالیتی بالا نشت می‌دهد. صادرکننده هرگز سقف را به‌طور خودکار افزایش نمی‌دهد؛ اگر مقدار آن بالا می‌رود، به‌جای غیرفعال‌کردن سقف، منبع را اصلاح کنید.

  </Accordion>
  <Accordion title="مواردی که هرگز در خروجی Prometheus ظاهر نمی‌شوند">
    - متن پرامپت، متن پاسخ، ورودی‌های ابزار، خروجی‌های ابزار، پرامپت‌های سیستمی
    - رونوشت‌های Talk، محتوای صوتی، شناسه‌های تماس، شناسه‌های اتاق، توکن‌های واگذاری، شناسه‌های نوبت و شناسه‌های خام نشست
    - شناسه‌های خام درخواست ارائه‌دهنده (فقط هش‌های محدود، در صورت کاربرد، روی spanها — هرگز روی سنجه‌ها)
    - کلیدهای نشست و شناسه‌های نشست
    - نام‌های میزبان، مسیرهای فایل، مقادیر محرمانه

  </Accordion>
</AccordionGroup>

## دستورهای PromQL

```promql
# توکن‌ها در دقیقه، تفکیک‌شده بر اساس ارائه‌دهنده
sum by (provider) (rate(openclaw_model_tokens_total[1m]))

# هزینه (USD) در یک ساعت گذشته، بر اساس مدل
sum by (model) (increase(openclaw_model_cost_usd_total[1h]))

# صدک ۹۵ مدت اجرای مدل
histogram_quantile(
  0.95,
  sum by (le, provider, model)
    (rate(openclaw_run_duration_seconds_bucket[5m]))
)

# SLO زمان انتظار صف (صدک ۹۵ کمتر از ۲ ثانیه)
histogram_quantile(
  0.95,
  sum by (le, lane) (rate(openclaw_queue_lane_wait_seconds_bucket[5m]))
) < 2

# استفاده از Skill، تفکیک‌شده بر اساس منبع محدود
sum by (skill, source) (increase(openclaw_skill_used_total[24h]))

# سری‌های حذف‌شده Prometheus (هشدار کاردینالیتی)
increase(openclaw_prometheus_series_dropped_total[15m]) > 0
```

<Tip>
برای داشبوردهای میان‌ارائه‌دهنده، `gen_ai_client_token_usage` را ترجیح دهید: این مورد از قراردادهای معنایی GenAI در OpenTelemetry پیروی می‌کند و با سنجه‌های سرویس‌های GenAI غیر OpenClaw سازگار است.
</Tip>

## انتخاب میان خروجی Prometheus و OpenTelemetry

OpenClaw از هر دو سطح به‌طور مستقل پشتیبانی می‌کند. می‌توانید یکی، هر دو یا هیچ‌کدام را اجرا کنید.

<Tabs>
  <Tab title="diagnostics-prometheus">
    - مدل **Pull**: ‏Prometheus نشانی `/api/diagnostics/prometheus` را جمع‌آوری می‌کند.
    - به گردآورنده خارجی نیازی نیست.
    - احراز هویت از طریق احراز هویت معمول Gateway انجام می‌شود.
    - این سطح فقط شامل سنجه‌ها است (بدون ردیابی یا گزارش‌ها).
    - بهترین گزینه برای پشته‌هایی که از قبل بر Prometheus + Grafana استاندارد شده‌اند.

  </Tab>
  <Tab title="diagnostics-otel">
    - مدل **Push**: ‏OpenClaw داده‌های OTLP/HTTP را به یک گردآورنده یا پشتیبان سازگار با OTLP ارسال می‌کند.
    - این سطح شامل سنجه‌ها، ردیابی‌ها و گزارش‌ها است.
    - در صورت نیاز به هر دو، از طریق OpenTelemetry Collector (صادرکننده `prometheus` یا `prometheusremotewrite`) به Prometheus متصل می‌شود.
    - برای فهرست کامل، [خروجی OpenTelemetry](/fa/gateway/opentelemetry) را ببینید.

  </Tab>
</Tabs>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="بدنه پاسخ خالی">
    - بررسی کنید که `diagnostics.enabled` در پیکربندی روی `false` تنظیم نشده باشد (مقدار پیش‌فرض آن `true` است).
    - با استفاده از `openclaw plugins list --enabled` تأیید کنید که Plugin فعال و بارگذاری شده است.
    - مقداری ترافیک ایجاد کنید؛ شمارنده‌ها و هیستوگرام‌ها فقط پس از وقوع حداقل یک رویداد، خط خروجی منتشر می‌کنند.

  </Accordion>
  <Accordion title="401 / غیرمجاز">
    نقطه پایانی به دامنه اپراتور Gateway نیاز دارد (`auth: "gateway"` با `gatewayRuntimeScopeSurface: "trusted-operator"`). از همان توکن یا گذرواژه‌ای استفاده کنید که Prometheus برای هر مسیر اپراتوری دیگر Gateway استفاده می‌کند. هیچ حالت عمومیِ بدون احراز هویت وجود ندارد.
  </Accordion>
  <Accordion title="`openclaw_prometheus_series_dropped_total` در حال افزایش است">
    یک ویژگی جدید از سقف **2048** سری فراتر می‌رود. سنجه‌های اخیر را برای یافتن برچسبی با کاردینالیتی بالای غیرمنتظره بررسی و آن را در منبع اصلاح کنید. صادرکننده عمداً به‌جای بازنویسی بی‌سروصدای برچسب‌ها، سری‌های جدید را حذف می‌کند.
  </Accordion>
  <Accordion title="Prometheus پس از راه‌اندازی مجدد سری‌های قدیمی را نشان می‌دهد">
    Plugin وضعیت را فقط در حافظه نگه می‌دارد. پس از راه‌اندازی مجدد Gateway، شمارنده‌ها به صفر بازنشانی می‌شوند و پیمانه‌ها از مقدار گزارش‌شده بعدی خود دوباره آغاز می‌شوند. برای مدیریت صحیح بازنشانی‌ها از `rate()` و `increase()` در PromQL استفاده کنید.
  </Accordion>
</AccordionGroup>

## مرتبط

- [خروجی عیب‌یابی](/fa/gateway/diagnostics) — فایل فشرده عیب‌یابی محلی برای بسته‌های پشتیبانی
- [سلامت و آمادگی](/fa/gateway/health) — پروب‌های `/healthz` و `/readyz`
- [ثبت گزارش](/fa/logging) — ثبت گزارش مبتنی بر فایل
- [خروجی OpenTelemetry](/fa/gateway/opentelemetry) — ارسال OTLP برای ردیابی‌ها، سنجه‌ها و گزارش‌ها
