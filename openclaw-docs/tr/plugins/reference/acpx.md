---
read_when:
    - acpx pluginini kuruyor, yapılandırıyor veya denetliyorsunuz
summary: Plugin tarafından yönetilen oturum ve aktarım yönetimine sahip OpenClaw ACP çalışma zamanı arka ucu.
title: ACPx plugin'i
x-i18n:
    generated_at: "2026-07-26T23:31:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9816ca3ada81eb44883b641f3d761b76f894bd83c8aa978c516125c77842f664
    source_path: plugins/reference/acpx.md
    workflow: 16
---

# ACPx plugin'i

Plugin'e ait oturum ve aktarım yönetimine sahip OpenClaw ACP çalışma zamanı arka ucu.

## Dağıtım

- Paket: `@openclaw/acpx`
- Yükleme yolu: npm; ClawHub

## Yüzey

Skills

<!-- openclaw-plugin-reference:manual-start -->

## Pi yerel oturumları

Paketle birlikte gelen çalışma zamanı, Gateway ve eşleştirilmiş
Node'larda Pi'nin oturum deposunu otomatik olarak algılar. Depolanan oturumlar, Pi'nin belgelenmiş JSONL oturum biçiminden
salt okunur transkriptlere göz atma özelliğiyle **Pi** oturumları kenar çubuğu grubunda görünür. Katalog, proje ve genel `settings.json` oturum dizinlerinin yanı sıra
`PI_CODING_AGENT_DIR` ve `PI_CODING_AGENT_SESSION_DIR` öğelerini dikkate alır. Göreli yollar,
`settings.json` dosyalarını içeren dizinden çözümlenir.

Keşfi devre dışı bırakmak için **Config > Plugins > ACPX Runtime** altındaki
**Pi Session Catalog** seçeneğini kapatın. Varsayılan olarak etkindir.

<!-- openclaw-plugin-reference:manual-end -->

## İlgili belgeler

- [acpx](/tr/tools/acp-agents-setup)
