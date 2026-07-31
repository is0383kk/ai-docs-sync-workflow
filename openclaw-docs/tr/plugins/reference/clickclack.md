---
read_when:
    - clickclack pluginini kuruyor, yapılandırıyor veya denetliyorsunuz
summary: OpenClaw mesajlarını göndermek ve almak için Clickclack kanal yüzeyini ekler.
title: Clickclack Plugin'i
x-i18n:
    generated_at: "2026-07-27T00:10:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fcb39341009946dc38a12cc24496e65fd704ed3f2f9aff44bb2dd29fdedaef26
    source_path: plugins/reference/clickclack.md
    workflow: 16
---

# Clickclack plugin'i

OpenClaw mesajlarını gönderip almak için Clickclack kanal yüzeyini ekler.

## Dağıtım

- Paket: `@openclaw/clickclack`
- Kurulum yolu: npm; ClawHub: `clawhub:@openclaw/clickclack`

## Yüzey

kanallar: `clickclack`; sözleşmeler: `tools`

<!-- openclaw-plugin-reference:manual-start -->

Plugin, isteğe bağlı olarak her OpenClaw oturumu için yaşam döngüsüyle eşzamanlı bir ClickClack kanalı oluşturabilir. Yönetilen tartışma kanalları, gözlem ve aktarım için aynı ajana ait bir yan oturum kullanırken bağlı ana oturum yalnızca çekme işlevli bir `discussion` aracı alır. Yapılandırma ve oturum aracı görünürlüğü gereksinimleri için [ClickClack oturum tartışmaları](/tr/channels/clickclack#session-discussions) bölümüne bakın.

<!-- openclaw-plugin-reference:manual-end -->

## İlgili belgeler

- [clickclack](/tr/channels/clickclack)
