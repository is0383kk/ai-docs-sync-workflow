---
read_when:
    - Çalışma alanını manuel olarak önyükleme
summary: HEARTBEAT.md için çalışma alanı şablonu
title: HEARTBEAT.md şablonu
x-i18n:
    generated_at: "2026-07-27T00:01:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# HEARTBEAT.md şablonu

`HEARTBEAT.md`, agent çalışma alanında bulunur ve periyodik heartbeat kontrol listesini içerir. OpenClaw'ın heartbeat model çağrısını tamamen atlaması için dosyayı boş ya da yalnızca boşluk, Markdown yorumları, ATX başlıkları, boş liste taslakları (`- `, `* [ ]`) veya çit işaretçileri içerecek şekilde tutun (`reason=empty-heartbeat-file`).

Dağıtılan varsayılan içerik:

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# Heartbeat API çağrılarını atlamak için bu dosyayı boş (veya yalnızca yorumlar içerecek şekilde) tutun.

# Heartbeat'in paylaşılan bağlamı incelemesi gerektiğinde aşağıya kısa bir kontrol listesi ekleyin.
```

Yalnızca tek bir heartbeat turunun öğeleri birlikte incelemesi gerektiğinde yorum satırlarının altına kısa bir kontrol listesi ekleyin. Listeyi küçük tutun: heartbeat çalıştırmaları bu dosyayı her tetiklemede (varsayılan olarak her 30 dakikada bir) okur; bu nedenle şişirilmiş talimatlar her uyanmada token tüketir.

Bağımsız olarak zamanlanan veya yalnızca zamanı gelen kontroller için [cron işleri](/tr/automation/cron-jobs) oluşturun. Heartbeat karalama alanı artık zamanlayıcı söz dizimini desteklemiyor. Eski `tasks:` bloklarını dönüştürmek için `openclaw doctor --fix` komutunu çalıştırın.

## İlgili

- [Heartbeat](/tr/gateway/heartbeat)
- [Heartbeat yapılandırması](/tr/gateway/config-agents)
