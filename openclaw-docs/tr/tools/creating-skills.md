---
read_when:
    - Yeni bir özel skill oluşturuyorsunuz
    - SKILL.md tabanlı Skills için hızlı bir başlangıç iş akışına ihtiyacınız var
    - Skill Workshop'u kullanarak ajan incelemesi için bir beceri önermek istiyorsunuz
sidebarTitle: Creating skills
summary: OpenClaw ajanlarınız için özel SKILL.md çalışma alanı becerileri oluşturun, test edin ve yayımlayın.
title: Skills oluşturma
x-i18n:
    generated_at: "2026-07-26T23:42:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills, ajana araçları nasıl ve ne zaman kullanacağını öğretir. Her skill, YAML frontmatter ve Markdown talimatları içeren bir `SKILL.md` dosyasına sahip bir dizindir.
OpenClaw, skill'leri tanımlı bir [öncelik sırasına](/tr/tools/skills#loading-order) göre çeşitli köklerden yükler.

## İlk skill'inizi oluşturun

<Steps>
  <Step title="Skill dizinini oluşturun">
    Skill'ler, çalışma alanınızdaki `skills/` klasöründe bulunur:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    Düzenleme amacıyla skill'leri alt klasörlerde gruplayabilirsiniz — skill yine de
    klasör yoluyla değil, `SKILL.md` frontmatter alanıyla adlandırılır:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # skill adı hâlâ "hello-world" ve /hello-world olarak çağrılır
    ```

  </Step>

  <Step title="SKILL.md dosyasını yazın">
    Frontmatter meta verileri tanımlar; gövde ise ajana talimatları verir.

    ```markdown
    ---
    name: hello-world
    description: Selamlama yazdıran basit bir skill.
    ---

    # Merhaba Dünya

    Kullanıcı bir selamlama istediğinde şunu çalıştırmak için `exec` aracını kullanın:

    ```bash
    echo "Özel skill'inizden merhaba!"
    ```
    ```

    Adlandırma kuralları:
    - `name` için küçük harfler, rakamlar ve kısa çizgiler kullanın.
    - Dizin adıyla frontmatter içindeki `name` alanını uyumlu tutun.
    - `description`, ajana ve eğik çizgi komutu keşfinde gösterilir —
      tek satırda ve 160 karakterden kısa tutun.

  </Step>

  <Step title="Skill'in yüklendiğini doğrulayın">
    ```bash
    openclaw skills list
    ```

    OpenClaw, varsayılan olarak skill kökleri altındaki `SKILL.md` dosyalarını izler. İzleyici
    devre dışıysa veya mevcut bir oturuma devam ediyorsanız ajanın yenilenmiş listeyi
    alması için yeni bir oturum başlatın:

    ```bash
    # Sohbetten — mevcut oturumu arşivleyip yeni bir oturum başlatın
    /new

    # Veya Gateway'i yeniden başlatın
    openclaw gateway restart
    ```

  </Step>

  <Step title="Test edin">
    ```bash
    openclaw agent --message "bana bir selamlama ver"
    ```

    Alternatif olarak bir sohbet açıp doğrudan ajana sorun. Ada göre açıkça
    çağırmak için `/skill hello-world` kullanın.

  </Step>
</Steps>

## SKILL.md başvurusu

### Zorunlu alanlar

| Alan          | Açıklama                                                        |
| ------------- | --------------------------------------------------------------- |
| `name`        | Küçük harfler, rakamlar ve kısa çizgiler kullanan benzersiz kısa ad |
| `description` | Ajana ve keşif çıktısında gösterilen tek satırlık açıklama       |

### İsteğe bağlı frontmatter anahtarları

| Alan                       | Varsayılan | Açıklama                                                                       |
| -------------------------- | ---------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | Skill'i kullanıcı eğik çizgi komutu olarak sunar                               |
| `disable-model-invocation` | `false` | Skill'i ajanın sistem isteminin dışında tutar (yine de `/skill` aracılığıyla çalışır) |
| `command-dispatch`         | —          | Eğik çizgi komutunu modeli atlayarak doğrudan bir araca yönlendirmek için `tool` olarak ayarlayın |
| `command-tool`             | —          | `command-dispatch: tool` ayarlandığında çağrılacak aracın adı                   |
| `command-arg-mode`         | `raw`   | Araç yönlendirmesinde ham bağımsız değişkenler dizesini araca iletir            |
| `homepage`                 | —          | macOS Skills kullanıcı arayüzünde "Website" olarak gösterilen URL               |

Erişim denetimi alanları (`requires.bins`, `requires.env` vb.) için
[Skills — Erişim denetimi](/tr/tools/skills#gating) bölümüne bakın.

### `{baseDir}` kullanımı

Skill dizini içindeki dosyalara yolları sabit kodlamadan başvurun — ajan
`{baseDir}` değerini skill'in kendi dizinine göre çözümler:

```markdown
`{baseDir}/scripts/run.sh` konumundaki yardımcı betiği çalıştırın.
```

## Koşullu etkinleştirme ekleme

Skill'inizin yalnızca bağımlılıkları kullanılabilir olduğunda yüklenmesi için erişim denetimi uygulayın:

```markdown
---
name: gemini-search
description: Gemini CLI kullanarak arama yapın.
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="Erişim denetimi seçenekleri">
    | Anahtar | Açıklama |
    | --- | --- |
    | `requires.bins` | Tüm ikili dosyalar `PATH` üzerinde bulunmalıdır |
    | `requires.anyBins` | En az bir ikili dosya `PATH` üzerinde bulunmalıdır |
    | `requires.env` | Her ortam değişkeni süreçte veya yapılandırmada bulunmalıdır |
    | `requires.config` | Her `openclaw.json` yolu doğru değerli olmalıdır |
    | `os` | Platform filtresi: `["darwin"]`, `["linux"]`, `["win32"]` |
    | `always` | Tüm erişim denetimlerini atlamak ve skill'i her zaman dahil etmek için `true` olarak ayarlayın |

    Tam başvuru: [Skills — Erişim denetimi](/tr/tools/skills#gating).

  </Accordion>
  <Accordion title="Ortam ve API anahtarları">
    Bir API anahtarını `openclaw.json` içindeki bir skill girdisine bağlayın:

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    Anahtar, yalnızca söz konusu ajan dönüşü için ana sürece enjekte edilir.
    Korumalı alana ulaşmaz — bkz.
    [korumalı alan ortam değişkenleri](/tr/tools/skills-config#sandboxed-skills-and-env-vars).

  </Accordion>
</AccordionGroup>

## Skill Workshop aracılığıyla önerin

Ajan tarafından taslak hâline getirilen skill'ler için veya bir skill kullanıma
girmeden önce operatör incelemesi istiyorsanız doğrudan `SKILL.md`
yazmak yerine [Skill Workshop](/tr/tools/skill-workshop) önerilerini kullanın.

```bash
# Yepyeni bir skill önerin
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "Selamlama yazdıran basit bir skill." \
  --proposal ./PROPOSAL.md

# Mevcut bir skill için güncelleme önerin
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "Güncellenmiş selamlama skill'i"
```

Öneri destek dosyaları içeriyorsa `--proposal-dir` kullanın:

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "Selamlama yazdıran basit bir skill." \
  --proposal-dir ./hello-world-proposal/
```

Dizin, kökünde `PROPOSAL.md` içermelidir. Destek dosyaları
`assets/`, `examples/`, `references/`, `scripts/` veya `templates/` altında bulunur.

İncelemeden sonra:

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

Önerinin tam yaşam döngüsü için [Skill Workshop](/tr/tools/skill-workshop) bölümüne bakın.

## ClawHub'da yayımlama

<Steps>
  <Step title="SKILL.md dosyanızın eksiksiz olduğundan emin olun">
    `name`, `description` ve tüm `metadata.openclaw` erişim denetimi alanlarının
    ayarlandığından emin olun. Bir proje sayfanız varsa `homepage` URL'si ekleyin.
  </Step>
  <Step title="Bağımsız ClawHub CLI'ı yükleyin ve oturum açın">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="Yayımlayın">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    Çıkarılan sürümü geçersiz kılmak veya belirli bir sahip altında yayımlamak
    için `--version <version>` ya da `--owner <owner>` ekleyin. Tam akış, sahip kapsamı ve diğer
    bakım komutları (`clawhub sync`, `clawhub skill rename`, ...) için
    [ClawHub — Yayımlama](/tr/clawhub/publishing) ve
    [ClawHub CLI](/tr/clawhub/cli) bölümlerine bakın.

  </Step>
</Steps>

## En iyi uygulamalar

<Tip>
  - **Kısa ve öz olun** — modele bir yapay zekâ olarak nasıl davranacağını değil, *ne* yapacağını söyleyin.
  - **Önce güvenlik** — skill'iniz `exec` kullanıyorsa istemlerin güvenilmeyen girdilerden
    rastgele komut enjeksiyonuna izin vermediğinden emin olun.
  - **Yerel olarak test edin** — paylaşmadan önce `openclaw agent --message "..."` kullanın.
  - **ClawHub kullanın** — sıfırdan oluşturmadan önce [clawhub.ai](https://clawhub.ai)
    adresindeki topluluk skill'lerine göz atın.
</Tip>

## İlgili konular

<CardGroup cols={2}>
  <Card title="Skills başvurusu" href="/tr/tools/skills" icon="puzzle-piece">
    Yükleme sırası, erişim denetimi, izin listeleri ve SKILL.md biçimi.
  </Card>
  <Card title="Skill Workshop" href="/tr/tools/skill-workshop" icon="flask">
    Ajan tarafından taslak hâline getirilen skill'ler için öneri kuyruğu.
  </Card>
  <Card title="Skills yapılandırması" href="/tr/tools/skills-config" icon="gear">
    Tam `skills.*` yapılandırma şeması.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Herkese açık kayıt defterindeki skill'lere göz atın ve skill yayımlayın.
  </Card>
  <Card title="Plugin oluşturma" href="/tr/plugins/building-plugins" icon="plug">
    Plugin'ler, belgeledikleri araçlarla birlikte skill'ler sunabilir.
  </Card>
</CardGroup>
