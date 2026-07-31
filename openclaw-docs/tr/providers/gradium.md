---
read_when:
    - Metin okuma için Gradium kullanmak istiyorsunuz
    - Gradium API anahtarı, ses veya yönerge belirteci yapılandırması gereklidir
summary: OpenClaw'da Gradium metinden sese dönüştürmeyi kullanma
title: Gradium
x-i18n:
    generated_at: "2026-07-27T00:15:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5536426eb6d3c8f24c04643b033ebb519a1f2f9df9d97c917ced1c7e23ad180d
    source_path: providers/gradium.md
    workflow: 16
---

[Gradium](https://gradium.ai), OpenClaw için bir metinden konuşmaya sağlayıcısıdır. Standart sesli yanıtlar (WAV), sesli notlarla uyumlu Opus çıktısı ve telefon arayüzleri için 8 kHz u-law ses üretir.

| Özellik       | Değer                                |
| ------------- | ------------------------------------ |
| Sağlayıcı kimliği | `gradium`                            |
| Kimlik doğrulama | `GRADIUM_API_KEY` veya `apiKey` yapılandırması |
| Temel URL     | `https://api.gradium.ai` (varsayılan)   |
| Varsayılan ses | `Emma` (`YTpq7expH9539ERJ`)          |

## Plugin'i yükleme

Gradium, resmî bir harici Plugin'dir. Plugin'i yükleyin, ardından Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/gradium-speech
openclaw gateway restart
```

## Kurulum

Bir Gradium API anahtarı oluşturun, ardından bunu bir ortam değişkeni veya yapılandırma anahtarıyla kullanıma açın. Yapılandırma, ortam değişkenine göre önceliklidir.

<Tabs>
  <Tab title="Ortam değişkeni">
    ```bash
    export GRADIUM_API_KEY="gsk_..."
    ```
  </Tab>

  <Tab title="Yapılandırma anahtarı">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "gradium",
        providers: {
          gradium: {
            apiKey: "${GRADIUM_API_KEY}",
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## Yapılandırma

```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        speakerVoiceId: "YTpq7expH9539ERJ",
        // apiKey: "${GRADIUM_API_KEY}",
        // baseUrl: "https://api.gradium.ai",
      },
    },
  },
}
```

| Anahtar                                | Tür    | Açıklama                                                                                             |
| -------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| `tts.providers.gradium.apiKey`         | dize | Çözümlenmiş API anahtarı. `${ENV}` ve gizli bilgi referanslarını destekler.                                                    |
| `tts.providers.gradium.baseUrl`        | dize | `api.gradium.ai` üzerindeki HTTPS Gradium API URL'si. Sondaki eğik çizgiler kaldırılır. Varsayılan: `https://api.gradium.ai`. |
| `tts.providers.gradium.speakerVoiceId` | dize | Direktifle geçersiz kılma belirtilmediğinde kullanılan varsayılan ses kimliği.                                            |

Çıktı biçimi hedef arayüze göre otomatik olarak seçilir (bkz. [Çıktı](#output)) ve `openclaw.json` içinde yapılandırılamaz.

## Sesler

| Ad                 | Ses kimliği        |
| ------------------ | ------------------ |
| Arthur             | `3jUdJyOi9pgbxBTK` |
| Christina          | `2H4HY2CBNyJHBCrP` |
| Emma **(varsayılan)** | `YTpq7expH9539ERJ` |
| John               | `KWJiFWu2O9nMPYcR` |
| Kent               | `LFZvm12tW_z0xfGo` |
| Sydney             | `jtEKaLYNn6iif5PR` |
| Tiffany            | `Eu9iL_CYe8N-Gkx_` |

### Mesaj başına sesi geçersiz kılma

Etkin konuşma ilkesi sesin geçersiz kılınmasına izin verdiğinde, bir direktif belirteciyle satır içinde sesler arasında geçiş yapın (bunların tümü eşdeğerdir ve sağlayıcıya özgü bir ses kimliği alır):

```text
/voice:LFZvm12tW_z0xfGo
/voice_id:LFZvm12tW_z0xfGo
/voiceid:LFZvm12tW_z0xfGo
/gradium_voice:LFZvm12tW_z0xfGo
/gradiumvoice:LFZvm12tW_z0xfGo
```

Konuşma ilkesi sesin geçersiz kılınmasını devre dışı bırakırsa direktif işlenir ancak yok sayılır.

## Çıktı

Çıktı biçimi hedef arayüze göre seçilir; sağlayıcı başka biçimler üretmez.

| Hedef          | Biçim       | Dosya uzantısı | Örnekleme hızı | Sesle uyumluluk işareti |
| -------------- | ----------- | -------------- | ------------- | ----------------------- |
| Standart ses   | `wav`       | `.wav`   | sağlayıcı     | hayır                   |
| Sesli not      | `opus`      | `.opus`  | sağlayıcı     | evet                    |
| Telefon        | `ulaw_8000` | yok            | 8 kHz         | yok                     |

## Otomatik seçim sırası

Yapılandırılmış TTS sağlayıcıları arasında Gradium'un otomatik seçim sırası `30` şeklindedir. `tts.provider` sabitlenmediğinde OpenClaw'ın etkin sağlayıcıyı nasıl seçtiğini öğrenmek için [Metinden Konuşmaya](/tr/tools/tts) bölümüne bakın.

## İlgili

- [Metinden Konuşmaya](/tr/tools/tts)
- [Medyaya Genel Bakış](/tr/tools/media-overview)
