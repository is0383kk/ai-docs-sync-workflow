---
read_when:
    - macOS Skills ayarları kullanıcı arayüzünü güncelleme
    - Skills geçiş denetimini veya kurulum davranışını değiştirme
summary: macOS Skills ayarları kullanıcı arayüzü ve Gateway destekli durum
title: Skills (macOS)
x-i18n:
    generated_at: "2026-07-26T22:52:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

macOS uygulaması, OpenClaw Skills'ı Gateway üzerinden sunar; Skills'ı yerel olarak ayrıştırmaz.

## Veri kaynağı

- `skills.status` (Gateway), paketlenmiş Skills için izin verilenler listesi engelleri de dahil olmak üzere tüm Skills'ı uygunluk ve eksik gereksinimlerle birlikte döndürür.
- Gereksinimler, her `SKILL.md` içindeki `metadata.openclaw.requires` kaynağından gelir.

## Yükleme eylemleri

- `metadata.openclaw.install`, yükleme seçeneklerini (brew/node/go/uv/download) tanımlar.
- Uygulama, yükleyicileri Gateway ana makinesinde çalıştırmak için `skills.install` çağrısını yapar.
- Operatörün yönettiği `security.installPolicy` (`enabled`, `targets`, `exec`), yükleyici meta verileri çalıştırılmadan önce Gateway destekli Skill yüklemelerini engelleyebilir. Yerleşik tehlikeli kod taraması (Plugin yüklemelerinde kullanılır), Skill yükleme akışına bağlanmamıştır.
- Her yükleme seçeneği `download` ise Gateway, tüm indirme seçeneklerini sunar.
- Aksi takdirde Gateway, mevcut yükleme tercihlerini (`skills.install.preferBrew`, `skills.install.nodeManager`) ve ana makine ikili dosyalarını kullanarak tercih edilen bir yükleyici seçer: `preferBrew` etkin ve `brew` mevcutsa önce Homebrew, ardından `uv`, sonra yapılandırılmış node yöneticisi, ardından mevcutsa (`preferBrew` olmasa bile) yeniden Homebrew, sonra `go` ve son olarak `download`.
- Node yükleme etiketleri, `yarn` dahil olmak üzere yapılandırılmış node yöneticisini yansıtır.

## Ortam/API anahtarları

- Uygulama, anahtarları `~/.openclaw/openclaw.json` içinde `skills.entries.<skillKey>` altında saklar.
- `skills.update`; `enabled`, `apiKey` ve `env` üzerinde yama uygular.

## Uzak mod

- Yükleme ve yapılandırma güncellemeleri yerel Mac'te değil, Gateway ana makinesinde gerçekleşir.

## İlgili

- [Skills](/tr/tools/skills)
- [macOS uygulaması](/tr/platforms/macos)
