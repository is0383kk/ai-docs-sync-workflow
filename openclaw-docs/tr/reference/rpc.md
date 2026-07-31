---
read_when:
    - Harici CLI entegrasyonları ekleme veya değiştirme
    - RPC bağdaştırıcılarında hata ayıklama (signal-cli, imsg)
summary: Harici CLI'lar (signal-cli, imsg) için RPC adaptörleri ve Gateway kalıpları
title: RPC bağdaştırıcıları
x-i18n:
    generated_at: "2026-07-26T23:35:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7deee8154dc824db4eccca9a26381711693972ba2606aec47d657e3724b3a5dd
    source_path: reference/rpc.md
    workflow: 16
---

OpenClaw, harici CLI'ları JSON-RPC aracılığıyla entegre eder. Günümüzde iki model kullanılmaktadır.

## Model A: HTTP daemon'ı (signal-cli)

- `signal-cli`, HTTP üzerinden JSON-RPC ile bir daemon olarak çalışır.
- Olay akışı SSE'dir (`/api/v1/events`).
- Durum yoklaması: `/api/v1/check`.
- `channels.signal.transport.kind="managed-native"` olduğunda (varsayılan) yaşam döngüsünü OpenClaw yönetir.

Kurulum ve uç noktalar için [Signal](/tr/channels/signal) bölümüne bakın.

## Model B: stdio alt süreci (imsg)

- OpenClaw, [iMessage](/tr/channels/imessage) için `imsg rpc` öğesini alt süreç olarak başlatır.
- JSON-RPC, stdin/stdout üzerinden satırlarla ayrılır (satır başına bir JSON nesnesi).
- TCP bağlantı noktası ve daemon gerekmez.

Kullanılan temel yöntemler:

- `watch.subscribe` → bildirimler (`method: "message"`)
- `watch.unsubscribe`
- `send`
- `chats.list` (yoklama/tanılama)

Kurulum ve adresleme için [iMessage](/tr/channels/imessage) bölümüne bakın (görüntüleme dizeleri yerine `chat_id` tercih edilir).

## Bağdaştırıcı yönergeleri

- Süreci Gateway yönetir (başlatma/durdurma, sağlayıcının yaşam döngüsüne bağlıdır).
- RPC istemcilerini dayanıklı tutun: zaman aşımları, çıkışta yeniden başlatma.
- Görüntüleme dizeleri yerine kararlı kimlikleri (ör. `chat_id`) tercih edin.

## İlgili konular

- [Gateway protokolü](/tr/gateway/protocol)
