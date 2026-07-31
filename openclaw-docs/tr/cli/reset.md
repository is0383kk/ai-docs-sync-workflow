---
read_when:
    - CLI'yi yüklü tutarken yerel durumu silmek istiyorsunuz
    - Nelerin kaldırılacağının deneme çalışmasını istiyorsunuz
summary: '`openclaw reset` için CLI başvurusu (yerel durumu/yapılandırmayı sıfırla)'
title: Sıfırla
x-i18n:
    generated_at: "2026-07-26T23:36:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

Yerel yapılandırmayı/durumu sıfırlayın (CLI yüklü kalır).

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## Seçenekler

- `--scope <scope>`: `config`, `config+creds+sessions` veya `full`
- `--yes`: onay istemlerini atlar
- `--non-interactive`: istemleri devre dışı bırakır; `--scope` ve `--yes` gerektirir
- `--dry-run`: dosyaları kaldırmadan eylemleri yazdırır

## Kapsamlar

| Kapsam                  | Kaldırılanlar                                                               | Önce Gateway'i durdurur |
| ----------------------- | --------------------------------------------------------------------------- | ----------------------- |
| `config`                | yalnızca yapılandırma dosyası                                                | hayır                   |
| `config+creds+sessions` | yapılandırma dosyası, OAuth/kimlik bilgileri dizini, aracı başına oturum dizinleri | evet                    |
| `full`                  | durum dizini (paylaşılan SQLite veritabanı dâhil) ve çalışma alanı dizinleri | evet                    |

`config+creds+sessions` ve `full`, durumu silmeden önce çalışan yönetilen Gateway hizmetini durdurur.

## Notlar

- Yerel durumu kaldırmadan önce geri yüklenebilir bir anlık görüntü oluşturmak için ilk olarak `openclaw backup create` komutunu çalıştırın.
- Çalışma alanı kurulum durumu ve tasdikler, paylaşılan SQLite veritabanında satırlar hâlindedir; bu nedenle `full`, bunları durum diziniyle birlikte kaldırır. Ayrı olarak kaldırılacak güncel tasdik yardımcı dosyaları yoktur.
- `--scope` olmadan `openclaw reset`, kaldırılacak kapsamı etkileşimli olarak sorar.
- `--non-interactive`, yalnızca hem `--scope` hem de `--yes` ayarlandığında geçerlidir.
- `config+creds+sessions` ve `full`, tamamlandığında `Next: openclaw onboard --install-daemon` yazdırır.

## İlgili

- [CLI başvurusu](/tr/cli)
