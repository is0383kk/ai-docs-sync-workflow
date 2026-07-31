---
read_when:
    - İlk aracı çalıştırmada neler olduğunu anlama
    - Önyükleme dosyalarının nerede bulunduğunu açıklama
    - İlk katılım kimlik kurulumu hatalarını ayıklama
sidebarTitle: Bootstrapping
summary: Çalışma alanı ve kimlik dosyalarını başlangıç verileriyle dolduran ajan önyükleme ritüeli
title: Aracı önyükleme
x-i18n:
    generated_at: "2026-07-27T00:17:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: efb47e1a6a86d68aef1aa1662fe9c5def9a4e5b45649b84aeb9060bfcba21a5d
    source_path: start/bootstrapping.md
    workflow: 16
---

Önyükleme, yeni bir agent çalışma alanını başlangıç verileriyle dolduran ve
agent'ın bir kimlik seçmesini sağlayan ilk çalıştırma ritüelidir. Katılım
işleminden hemen sonra, agent'ın ilk gerçek turunda bir kez çalışır.

## Neler gerçekleşir?

Yepyeni bir çalışma alanında (varsayılan `~/.openclaw/workspace`) ilk kez çalıştırıldığında
OpenClaw:

- `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md` ve `BOOTSTRAP.md` dosyalarını başlangıç verileriyle doldurur.
- Agent'ın en fazla üç adımdan oluşan bir doğuş dizisini izlemesini sağlar: ona
  ne ad vermek istediğinizi sorar, ruhunu/havasını anlatan kısa bir cümle paylaşır
  ve önerilen asgari plugin kümesini mi yoksa azami kullanım kolaylığını mı
  istediğinizi sorar.
- Üzerinde anlaşılan kimliği iki şekilde kalıcı hâle getirir: `IDENTITY.md` ve `SOUL.md` içine
  (agent'ın kendisi hakkında okudukları) ve `openclaw agents set-identity` aracılığıyla (kanalların
  ve kullanıcı arayüzünün gösterdikleri).
- Katılım sırasında önceden kaydedilmiş uygulama önerilerini yeniden taramadan okur.
  Resmî pluginler `openclaw plugins install <id>` kullanır; üçüncü taraf ClawHub
  Skills açıkça etkinleştirilmeyi gerektirmeye devam eder. Seçim işlendikten sonra agent,
  bir daha sormamak üzere kayıtlı önerinin alındığını onaylar.
- Çalışma alanı yapılandırılmış göründüğünde `BOOTSTRAP.md` dosyasını siler; böylece ritüel yalnızca bir kez çalışır.

`SOUL.md`, `IDENTITY.md` veya `USER.md` başlangıç şablonundan
farklılaştığında ya da bir `memory/` klasörü bulunduğunda çalışma alanı yapılandırılmış sayılır.

<Note>
`BOOTSTRAP.md`, kimlik konuşmasının tamamını kapsar. İçeriğini
[BOOTSTRAP.md şablonu](/tr/reference/templates/BOOTSTRAP) sayfasında görebilirsiniz.
</Note>

## Gömülü ve yerel model çalıştırmaları

OpenClaw, gömülü veya yerel model çalıştırmalarında `BOOTSTRAP.md` dosyasını
ayrıcalıklı sistem bağlamının dışında tutar. Birincil etkileşimli ilk çalıştırmada
dosyanın içeriğini yine kullanıcı istemi üzerinden iletir; böylece
`read` aracını güvenilir biçimde çağırmayan modeller de ritüeli tamamlayabilir.
Mevcut çalıştırma çalışma alanına güvenli biçimde erişemiyorsa agent, genel bir
karşılama yerine kısa bir sınırlı önyükleme notu alır.

## Önyüklemeyi atlama

Önceden başlangıç verileriyle doldurulmuş bir çalışma alanında bunu atlamak için şu komutu çalıştırın:

```bash
openclaw onboard --skip-bootstrap
```

## Nerede çalışır?

Önyükleme her zaman gateway ana makinesinde çalışır. macOS uygulaması uzak bir
Gateway'e bağlanırsa çalışma alanı ve önyükleme dosyaları Mac'te değil, bu uzak
makinede bulunur.

<Note>
Gateway başka bir makinede çalıştığında çalışma alanı dosyalarını gateway
ana makinesinde (örneğin `user@gateway-host:~/.openclaw/workspace`) düzenleyin.
</Note>

## İlgili belgeler

- macOS uygulaması katılımı: [Katılım](/tr/start/onboarding)
- Çalışma alanı düzeni: [Agent çalışma alanı](/tr/concepts/agent-workspace)
- Şablon içeriği: [BOOTSTRAP.md şablonu](/tr/reference/templates/BOOTSTRAP)
