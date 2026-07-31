---
read_when:
    - Gateway hizmetini ve/veya yerel durumu kaldırmak istiyorsunuz
    - Önce bir deneme çalıştırması yapmak istiyorsunuz
summary: '`openclaw uninstall` için CLI başvurusu (Gateway hizmetini + yerel verileri kaldırma)'
title: Kaldırma
x-i18n:
    generated_at: "2026-07-26T23:14:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1e2e3996cf6d5c0fd11e5054c8fe60f7f8d25047193bb13944ca170bf77b581a
    source_path: cli/uninstall.md
    workflow: 16
---

# `openclaw uninstall`

Gateway hizmetini ve/veya yerel verileri kaldırın. CLI'nin kendisi
kaldırılmaz; onu ayrıca npm/pnpm aracılığıyla kaldırın.

## Seçenekler

| Bayrak                | Varsayılan | Açıklama                                          |
| ------------------- | ------- | ---------------------------------------------------- |
| `--service`         | `false` | Gateway hizmetini kaldırır.                          |
| `--state`           | `false` | Durumu ve yapılandırmayı kaldırır.                             |
| `--workspace`       | `false` | Çalışma alanı dizinlerini kaldırır.                        |
| `--app`             | `false` | macOS uygulamasını kaldırır.                                |
| `--all`             | `false` | `--service --state --workspace --app` için kısa gösterimdir. |
| `--yes`             | `false` | Onay istemlerini atlar.                           |
| `--non-interactive` | `false` | İstemleri devre dışı bırakır; `--yes` gerektirir.                   |
| `--dry-run`         | `false` | Dosyaları kaldırmadan planlanan eylemleri yazdırır.        |

Kapsam bayrağı verilmezse etkileşimli bir çoklu seçim istemi, kaldırılacak
bileşenleri sorar (hizmet, durum ve çalışma alanı varsayılan olarak önceden seçilidir).

## Örnekler

```bash
openclaw backup create
openclaw uninstall
openclaw uninstall --service --yes --non-interactive
openclaw uninstall --state --workspace --yes --non-interactive
openclaw uninstall --all --yes
openclaw uninstall --dry-run
```

## Notlar

- Durumu veya çalışma alanlarını kaldırmadan önce geri yüklenebilir bir anlık görüntü oluşturmak için
  önce `openclaw backup create` komutunu çalıştırın.
- `--state`, `--workspace` de seçilmediği sürece yapılandırılmış çalışma alanı dizinlerini
  korur.

## İlgili

- [CLI başvurusu](/tr/cli)
- [Kaldırma](/tr/install/uninstall)
