---
read_when:
    - Bağımlılıkları yüklemek veya paket betiklerini Bun ile çalıştırmak istiyorsunuz
    - Bun kurulum/yama/yaşam döngüsü betiği sorunlarıyla karşılaşıyorsunuz
summary: Kurulumlar ve paket betikleri için Bun iş akışı; çalışma zamanında Node gereklidir
title: Bun
x-i18n:
    generated_at: "2026-07-26T23:23:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b822f700123b91c785eb881ebf28a63e77915b46dfd44beb9dbf63fb71aaa0d2
    source_path: install/bun.md
    workflow: 16
---

<Warning>
Bun, gerekli `node:sqlite` API'sini sağlamadığı için OpenClaw CLI'yi veya Gateway'i çalıştıramaz. Tüm OpenClaw çalışma zamanı komutları için desteklenen bir Node sürümü yükleyin.
</Warning>

Bun, isteğe bağlı bir bağımlılık yükleyicisi ve paket betiği çalıştırıcısı olarak kullanılmaya devam edebilir. Varsayılan paket yöneticisi, tam olarak desteklenen ve dokümantasyon araçları tarafından kullanılan `pnpm` olmaya devam eder. Bun, `pnpm-lock.yaml` kullanamaz ve bunu yok sayar.

## Kurulum

<Steps>
  <Step title="Bağımlılıkları yükleyin">
    ```sh
    bun install
    ```

    `bun.lock` / `bun.lockb` git tarafından yok sayılır, dolayısıyla depoda değişiklik oluşmaz. Kilit dosyasına yazmayı tamamen atlamak için:

    ```sh
    bun install --no-save
    ```

  </Step>
  <Step title="Derleyin ve test edin">
    ```sh
    bun run build
    bun run vitest run
    ```

    OpenClaw'ın kendisini başlatan komutlar yine de Node üzerinden çalıştırılmalıdır.

  </Step>
</Steps>

## Yaşam döngüsü betikleri

Bun, açıkça güvenilir olarak işaretlenmedikçe bağımlılık yaşam döngüsü betiklerini engeller. Bu depo için yaygın olarak engellenen betikler gerekli değildir:

- `baileys` `preinstall`: Node ana sürümünün >= 20 olduğunu denetler (OpenClaw; Node 24 önerilmek üzere Node 22.22.3+, 24.15+ veya 25.9+ gerektirir)
- `protobufjs` `postinstall`: uyumsuz sürüm şemaları hakkında uyarılar verir (derleme yapıtı yoktur)

Bu betikleri gerektiren bir çalışma zamanı sorunuyla karşılaşırsanız bunları açıkça güvenilir olarak işaretleyin:

```sh
bun pm trust baileys protobufjs
```

## Uyarılar

Bazı paket betikleri, dahili olarak `pnpm` değerini sabit kodlar (örneğin `check:docs`, `ui:*`, `protocol:check`). Bunları `bun run` üzerinden çalıştırmak yine de `pnpm` çağırır; bu nedenle söz konusu betikleri doğrudan `pnpm` üzerinden çalıştırın.

## İlgili konular

- [Kuruluma genel bakış](/tr/install)
- [Node.js](/tr/install/node)
- [Güncelleme](/tr/install/updating)
