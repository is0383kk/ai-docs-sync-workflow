---
read_when:
    - OpenClaw'da daha kısa `exec` veya `bash` araç sonuçları istiyorsunuz
    - Tokenjuice Pluginini yüklemek veya etkinleştirmek istiyorsunuz
    - tokenjuice'ın neyi değiştirdiğini ve neyi ham bıraktığını anlamanız gerekir
summary: İsteğe bağlı Tokenjuice plugin'iyle gürültülü exec ve bash aracı sonuçlarını sıkıştırın
title: Tokenjuice
x-i18n:
    generated_at: "2026-07-26T23:44:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 96b110563a2600429dd9f0d38997cf7cc5ae4952b7f146a6ab64c96f2f202440
    source_path: tools/tokenjuice.md
    workflow: 16
---

`tokenjuice`, komut zaten çalıştırıldıktan sonra gürültülü `exec` ve `bash`
araç sonuçlarını sıkıştıran isteğe bağlı bir harici plugindir.

Komutun kendisini değil, döndürülen `tool_result` değerini değiştirir. Tokenjuice;
kabuk girdisini yeniden yazmaz, komutları yeniden çalıştırmaz veya çıkış kodlarını değiştirmez.

Bu özellik şu anda OpenClaw gömülü çalıştırmaları ve Codex app-server
harness'ındaki OpenClaw dinamik araçları için geçerlidir. Tokenjuice, OpenClaw'ın araç sonucu ara yazılımına bağlanır ve
çıktıyı etkin harness oturumuna geri dönmeden önce kırpar.

## Plugini etkinleştirme

Bir kez yükleyin:

```bash
openclaw plugins install clawhub:@openclaw/tokenjuice
```

Ardından etkinleştirin:

```bash
openclaw config set plugins.entries.tokenjuice.enabled true
```

Eşdeğer komut:

```bash
openclaw plugins enable tokenjuice
```

Yapılandırmayı doğrudan düzenlemeyi tercih ediyorsanız:

```json5
{
  plugins: {
    entries: {
      tokenjuice: {
        enabled: true,
      },
    },
  },
}
```

## Tokenjuice neleri değiştirir?

- Gürültülü `exec` ve `bash` sonuçlarını oturuma geri beslenmeden önce sıkıştırır.
- Özgün komut yürütmesini değiştirmeden korur.
- Güvenli envanter ilkesini uygular: tam dosya içeriği okumaları ham kalır, bağımsız depo envanteri komutları sıkıştırılabilir ve güvenli olmayan karma komut dizileri ham kalır.
- İsteğe bağlı kalır: her yerde çıktıyı birebir istiyorsanız plugini devre dışı bırakın.

## Çalıştığını doğrulama

1. Plugini etkinleştirin.
2. `exec` çağrısı yapabilen bir oturum başlatın.
3. `git status` gibi gürültülü bir komut çalıştırın.
4. Döndürülen araç sonucunun ham kabuk çıktısından daha kısa ve daha yapılandırılmış olduğunu kontrol edin.

## Plugini devre dışı bırakma

```bash
openclaw config set plugins.entries.tokenjuice.enabled false
```

Veya:

```bash
openclaw plugins disable tokenjuice
```

## İlgili içerikler

- [Exec aracı](/tr/tools/exec)
- [Düşünme düzeyleri](/tr/tools/thinking)
- [Bağlam motoru](/tr/concepts/context-engine)
