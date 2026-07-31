---
read_when:
    - Prometheus, Grafana, VictoriaMetrics veya başka bir kazıyıcının OpenClaw Gateway metriklerini toplamasını istiyorsunuz
    - Panolar veya uyarılar için Prometheus metrik adlarına ve etiket politikasına ihtiyacınız var
    - OpenTelemetry toplayıcısı çalıştırmadan metrikler istiyorsunuz
sidebarTitle: Prometheus
summary: OpenClaw tanılamalarını diagnostics-prometheus Plugin’i aracılığıyla Prometheus metin metrikleri olarak sunun
title: Prometheus metrikleri
x-i18n:
    generated_at: "2026-07-26T23:41:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d04a46bdb401df3cdd2571b973f2a60f264862cf74da02c5a9cfa1de6ea9ffe
    source_path: gateway/prometheus.md
    workflow: 16
---

OpenClaw, resmi `diagnostics-prometheus` plugin aracılığıyla tanılama metriklerini sunabilir. Güvenilir tanılamaların yanı sıra dahili olarak etiketlenmiş, dağıtıcıya ait tanılama olaylarını (kuyruk, bellek ve oturum kurtarma sinyalleri) dinler ve şu adreste bir Prometheus metin uç noktası oluşturur:

```text
GET /api/diagnostics/prometheus
```

İçerik türü, standart Prometheus sunum biçimi olan `text/plain; version=0.0.4; charset=utf-8` değeridir.

<Warning>
Rota, Gateway kimlik doğrulamasını (operatör kapsamı, güvenilir operatör yüzeyi) kullanır. Bunu kimlik doğrulaması olmayan herkese açık bir `/metrics` uç noktası olarak sunmayın. Diğer operatör API'leri için kullandığınız kimlik doğrulama yolu üzerinden veri toplayın.
</Warning>

İzler, günlükler, OTLP gönderimi ve OpenTelemetry GenAI semantik öznitelikleri için [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) bölümüne bakın.

## Hızlı başlangıç

<Steps>
  <Step title="Plugin'i yükleyin">
    ```bash
    openclaw plugins install clawhub:@openclaw/diagnostics-prometheus
    ```
  </Step>
  <Step title="Plugin'i etkinleştirin">
    <Tabs>
      <Tab title="Yapılandırma">
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
  <Step title="Gateway'i yeniden başlatın">
    HTTP rotası Plugin başlatılırken kaydedilir; bu nedenle etkinleştirdikten sonra yeniden yükleyin.
  </Step>
  <Step title="Korumalı rotadan veri toplayın">
    Operatör istemcilerinizin kullandığı Gateway kimlik doğrulamasının aynısını gönderin:

    ```bash
    curl -H "Authorization: Bearer $OPENCLAW_GATEWAY_TOKEN" \
      http://127.0.0.1:18789/api/diagnostics/prometheus
    ```

  </Step>
  <Step title="Prometheus'u bağlayın">
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
`diagnostics.enabled` varsayılan olarak `true` değerindedir; yalnızca sıkı biçimde kısıtlanmış ortamlarda `false` olarak ayarlayın. `false` ise Plugin HTTP rotasını kaydetmeye devam eder ancak dışa aktarıcıya hiçbir tanılama olayı ulaşmaz; bu nedenle yanıt boş olur.
</Note>

## Dışa aktarılan metrikler

| Metrik                                           | Tür       | Etiketler                                                                                 |
| ------------------------------------------------ | --------- | ----------------------------------------------------------------------------------------- |
| `openclaw_run_completed_total`                   | sayaç     | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_run_duration_seconds`                  | histogram | `channel`, `model`, `outcome`, `provider`, `trigger`                                      |
| `openclaw_model_call_total`                      | sayaç     | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_call_duration_seconds`           | histogram | `api`, `error_category`, `model`, `observation_unit`, `outcome`, `provider`, `transport`  |
| `openclaw_model_failover_total`                  | sayaç     | `from_model`, `from_provider`, `lane`, `reason`, `suspended`, `to_model`, `to_provider`   |
| `openclaw_model_tokens_total`                    | sayaç     | `agent`, `channel`, `model`, `provider`, `token_type`                                     |
| `openclaw_gen_ai_client_token_usage`             | histogram | `model`, `provider`, `token_type`                                                         |
| `openclaw_model_cost_usd_total`                  | sayaç     | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_model_usage_duration_seconds`          | histogram | `agent`, `channel`, `model`, `provider`                                                   |
| `openclaw_skill_used_total`                      | sayaç     | `activation`, `agent`, `skill`, `source`                                                  |
| `openclaw_tool_execution_total`                  | sayaç     | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_duration_seconds`       | histogram | `error_category`, `outcome`, `params_kind`, `tool`, `tool_owner`, `tool_source`           |
| `openclaw_tool_execution_blocked_total`          | sayaç     | `denied_reason`, `params_kind`, `tool`, `tool_owner`, `tool_source`                       |
| `openclaw_harness_run_total`                     | sayaç     | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_harness_run_duration_seconds`          | histogram | `channel`, `error_category`, `harness`, `model`, `outcome`, `phase`, `plugin`, `provider` |
| `openclaw_webhook_received_total`                | sayaç     | `channel`, `webhook`                                                                      |
| `openclaw_webhook_error_total`                   | sayaç     | `channel`, `webhook`                                                                      |
| `openclaw_webhook_duration_seconds`              | histogram | `channel`, `webhook`                                                                      |
| `openclaw_message_received_total`                | sayaç     | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_started_total`        | sayaç     | `channel`, `source`                                                                       |
| `openclaw_message_dispatch_completed_total`      | sayaç     | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_dispatch_duration_seconds`     | histogram | `channel`, `outcome`, `reason`, `source`                                                  |
| `openclaw_message_processed_total`               | sayaç     | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_processed_duration_seconds`    | histogram | `channel`, `outcome`, `reason`                                                            |
| `openclaw_message_delivery_started_total`        | sayaç     | `channel`, `delivery_kind`                                                                |
| `openclaw_message_delivery_total`                | sayaç     | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_message_delivery_duration_seconds`     | histogram | `channel`, `delivery_kind`, `error_category`, `outcome`                                   |
| `openclaw_talk_event_total`                      | sayaç     | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_event_duration_seconds`           | histogram | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_talk_audio_bytes`                      | histogram | `brain`, `event_type`, `mode`, `provider`, `transport`                                    |
| `openclaw_queue_lane_size`                       | gösterge  | `lane`                                                                                    |
| `openclaw_queue_lane_wait_seconds`               | histogram | `lane`                                                                                    |
| `openclaw_session_state_total`                   | sayaç     | `reason`, `state`                                                                         |
| `openclaw_session_queue_depth`                   | gösterge  | `state`                                                                                   |
| `openclaw_session_turn_created_total`            | sayaç     | `agent`, `channel`, `trigger`                                                             |
| `openclaw_session_stuck_total`                   | sayaç     | `reason`, `state`                                                                         |
| `openclaw_session_stuck_age_seconds`             | histogram | `reason`, `state`                                                                         |
| `openclaw_session_recovery_total`                | sayaç     | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_session_recovery_age_seconds`          | histogram | `action`, `active_work_kind`, `state`, `status`                                           |
| `openclaw_liveness_warning_total`                | sayaç     | `reason`                                                                                  |
| `openclaw_liveness_sessions`                     | gösterge  | `state`                                                                                   |
| `openclaw_liveness_event_loop_delay_p99_seconds` | histogram | `reason`                                                                                  |
| `openclaw_liveness_event_loop_delay_max_seconds` | histogram | `reason`                                                                                  |
| `openclaw_liveness_event_loop_utilization_ratio` | histogram | `reason`                                                                                  |
| `openclaw_liveness_cpu_core_ratio`               | histogram | `reason`                                                                                  |
| `openclaw_payload_large_total`                   | sayaç     | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_payload_large_bytes`                   | histogram | `action`, `channel`, `plugin`, `reason`, `surface`                                        |
| `openclaw_memory_bytes`                          | gösterge  | `kind`                                                                                    |
| `openclaw_memory_rss_bytes`                      | histogram | yok                                                                                       |
| `openclaw_memory_pressure_total`                 | sayaç     | `level`, `reason`                                                                         |
| `openclaw_telemetry_exporter_total`              | sayaç     | `exporter`, `reason`, `signal`, `status`                                                  |
| `openclaw_prometheus_series_dropped_total`       | sayaç     | yok                                                                                       |
| `openclaw_diagnostic_async_queue_dropped_total`  | sayaç     | `drop_class`                                                                              |
| `openclaw_diagnostic_async_queue_length`         | gösterge  | yok                                                                                       |

Model çağrısı metrikleri için `observation_unit="request"`, gözlemlenebilir tek bir
sağlayıcı isteğini ölçer. `observation_unit="turn"`, birden fazla gizli sağlayıcı isteği
içerebilen sentetik bir Claude Code veya Codex CLI aracı dönüşünü ölçer.
Gecikmeyi karşılaştırırken bu serileri ayrı tutun.

## Etiket politikası

<AccordionGroup>
  <Accordion title="Sınırlı, düşük kardinaliteli etiketler">
    Prometheus etiketleri sınırlı ve düşük kardinaliteli kalır. Dışa aktarıcı; `runId`, `sessionKey`, `sessionId`, `callId`, `toolCallId`, mesaj kimlikleri, sohbet kimlikleri veya sağlayıcı istek kimlikleri gibi ham tanılama tanımlayıcılarını yayımlamaz.

    Etiket değerleri sansürlenir ve OpenClaw'ın düşük kardinaliteli karakter politikasıyla eşleşmelidir. Politikaya uymayan değerler, metriğe bağlı olarak `unknown`, `other` veya `none` ile değiştirilir. Kapsamlı aracı oturumu anahtarlarına benzeyen etiketler de `unknown` ile değiştirilir.

  </Accordion>
  <Accordion title="Seri sınırı ve taşma hesabı">
    Dışa aktarıcı, sayaçlar, göstergeler ve histogramların toplamında bellekte tutulan zaman serilerini **2048** seriyle sınırlar. Bu sınırın üzerindeki yeni seriler bırakılır ve her seferinde `openclaw_prometheus_series_dropped_total` bir artırılır.

    Bu sayacı, yukarı akıştaki bir özniteliğin yüksek kardinaliteli değerler sızdırdığına ilişkin kesin bir sinyal olarak izleyin. Dışa aktarıcı sınırı hiçbir zaman otomatik olarak yükseltmez; değer artarsa sınırı devre dışı bırakmak yerine kaynağı düzeltin.

  </Accordion>
  <Accordion title="Prometheus çıktısında hiçbir zaman görünmeyenler">
    - istem metni, yanıt metni, araç girdileri, araç çıktıları, sistem istemleri
    - Konuşma dökümleri, ses yükleri, çağrı kimlikleri, oda kimlikleri, devir belirteçleri, dönüş kimlikleri ve ham oturum kimlikleri
    - ham sağlayıcı istek kimlikleri (uygun olduğu durumlarda yalnızca span'lerde sınırlı karmalar — metriklerde hiçbir zaman bulunmaz)
    - oturum anahtarları ve oturum kimlikleri
    - ana makine adları, dosya yolları, gizli değerler

  </Accordion>
</AccordionGroup>

## PromQL tarifleri

```promql
# Sağlayıcıya göre ayrılmış, dakika başına belirteçler
sum by (provider) (rate(openclaw_model_tokens_total[1m]))

# Modele göre son bir saatteki harcama (USD)
sum by (model) (increase(openclaw_model_cost_usd_total[1h]))

# Model çalıştırma süresinin 95. yüzdelik dilimi
histogram_quantile(
  0.95,
  sum by (le, provider, model)
    (rate(openclaw_run_duration_seconds_bucket[5m]))
)

# Kuyruk bekleme süresi SLO'su (95p, 2 sn'nin altında)
histogram_quantile(
  0.95,
  sum by (le, lane) (rate(openclaw_queue_lane_wait_seconds_bucket[5m]))
) < 2

# Sınırlı kaynağa göre ayrılmış Skills kullanımı
sum by (skill, source) (increase(openclaw_skill_used_total[24h]))

# Bırakılan Prometheus serileri (kardinalite alarmı)
increase(openclaw_prometheus_series_dropped_total[15m]) > 0
```

<Tip>
Sağlayıcılar arası panolar için `gen_ai_client_token_usage` tercih edin: OpenTelemetry GenAI anlamsal kurallarını izler ve OpenClaw dışındaki GenAI hizmetlerinden gelen metriklerle tutarlıdır.
</Tip>

## Prometheus ile OpenTelemetry dışa aktarımı arasında seçim yapma

OpenClaw her iki yüzeyi de birbirinden bağımsız olarak destekler. İkisinden birini, ikisini birden veya hiçbirini çalıştırabilirsiniz.

<Tabs>
  <Tab title="diagnostics-prometheus">
    - **Çekme** modeli: Prometheus, `/api/diagnostics/prometheus` uç noktasını tarar.
    - Harici toplayıcı gerekmez.
    - Normal Gateway kimlik doğrulamasıyla doğrulanır.
    - Yüzey yalnızca metrikleri içerir (iz veya günlük içermez).
    - Prometheus + Grafana üzerinde zaten standartlaşmış yığınlar için en uygunudur.

  </Tab>
  <Tab title="diagnostics-otel">
    - **Gönderme** modeli: OpenClaw, OTLP/HTTP aracılığıyla bir toplayıcıya veya OTLP uyumlu arka uca veri gönderir.
    - Yüzey; metrikleri, izleri ve günlükleri içerir.
    - Her ikisine de ihtiyaç duyduğunuzda bir OpenTelemetry Collector (`prometheus` veya `prometheusremotewrite` dışa aktarıcısı) aracılığıyla Prometheus ile köprü kurar.
    - Tam katalog için [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) sayfasına bakın.

  </Tab>
</Tabs>

## Sorun giderme

<AccordionGroup>
  <Accordion title="Boş yanıt gövdesi">
    - Yapılandırmada `diagnostics.enabled` değerinin `false` olarak ayarlanmadığını kontrol edin (varsayılanı `true` değeridir).
    - Plugin'in etkinleştirildiğini ve `openclaw plugins list --enabled` ile yüklendiğini doğrulayın.
    - Bir miktar trafik oluşturun; sayaçlar ve histogramlar yalnızca en az bir olaydan sonra satır yayımlar.

  </Accordion>
  <Accordion title="401 / yetkisiz">
    Uç nokta, Gateway operatör kapsamını (`auth: "gateway"` ile `gatewayRuntimeScopeSurface: "trusted-operator"`) gerektirir. Prometheus'un diğer Gateway operatör rotaları için kullandığı belirteç veya parolanın aynısını kullanın. Kimlik doğrulaması gerektirmeyen herkese açık bir mod yoktur.
  </Accordion>
  <Accordion title="`openclaw_prometheus_series_dropped_total` artıyor">
    Yeni bir öznitelik, **2048** serilik sınırı aşıyor. Beklenmedik ölçüde yüksek kardinaliteli bir etiket olup olmadığını görmek için son metrikleri inceleyin ve sorunu kaynağında düzeltin. Dışa aktarıcı, etiketleri sessizce yeniden yazmak yerine yeni serileri bilinçli olarak bırakır.
  </Accordion>
  <Accordion title="Prometheus yeniden başlatmadan sonra eski seriler gösteriyor">
    Plugin, durumu yalnızca bellekte tutar. Gateway yeniden başlatıldıktan sonra sayaçlar sıfırlanır ve göstergeler bildirilen sonraki değerlerinden yeniden başlar. Sıfırlamaları sorunsuz işlemek için PromQL `rate()` ve `increase()` kullanın.
  </Accordion>
</AccordionGroup>

## İlgili içerikler

- [Tanılama dışa aktarımı](/tr/gateway/diagnostics) — destek paketleri için yerel tanılama zip dosyası
- [Sistem durumu ve hazır olma](/tr/gateway/health) — `/healthz` ve `/readyz` yoklamaları
- [Günlük kaydı](/tr/logging) — dosya tabanlı günlük kaydı
- [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry) — izler, metrikler ve günlükler için OTLP gönderimi
