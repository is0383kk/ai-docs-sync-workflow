---
read_when:
    - OpenClaw'da Zalo Personal (resmî olmayan) desteği istiyorsunuz
    - zalouser pluginini yapılandırıyor veya geliştiriyorsunuz
summary: 'Zalo Personal plugin’i: yerel zca-js aracılığıyla QR ile oturum açma + mesajlaşma (plugin kurulumu + kanal yapılandırması + araç)'
title: Zalo kişisel plugin'i
x-i18n:
    generated_at: "2026-07-27T00:14:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb0bdaa10340b5d78dc32abf6b0520fda6cf5f65e2e17b551b4e9bd72acfbbf2
    source_path: plugins/zalouser.md
    workflow: 16
---

Yerel `zca-js` kullanarak normal bir Zalo kullanıcı hesabını otomatikleştiren bir plugin aracılığıyla OpenClaw için Zalo Personal desteği. Harici bir `zca`/`openzca` CLI ikili dosyası
gerekmez.

<Warning>
Resmî olmayan otomasyon, hesabın askıya alınmasına veya yasaklanmasına yol açabilir. Kullanım riski size aittir.
</Warning>

## Adlandırma

Kanal kimliği, bunun **kişisel bir Zalo kullanıcı hesabını** (resmî olmayan) otomatikleştirdiğini açıkça belirtmek için `zalouser` olarak belirlenmiştir. Ayrı `zalo` kanal kimliği, resmî ve
pakete dâhil Zalo Bot/Webhook entegrasyonudur. Bkz. [Zalo](/tr/channels/zalo).

## Nerede çalışır

Bu plugin **Gateway işlemi içinde** çalışır. Uzak bir Gateway için
plugin'i o ana makineye kurup yapılandırın, ardından Gateway'i yeniden başlatın.

## Kurulum

### npm'den

```bash
openclaw plugins install @openclaw/zalouser
```

Güncel resmî sürüm etiketini takip etmek için yalnızca paket adını kullanın; tam bir
sürümü yalnızca tekrarlanabilir bir kurulum gerektiğinde sabitleyin. Ardından Gateway'i
yeniden başlatın.

### Yerel klasörden (geliştirme)

```bash
PLUGIN_SRC=./path/to/local/zalouser-plugin
openclaw plugins install "$PLUGIN_SRC"
cd "$PLUGIN_SRC" && pnpm install
```

Ardından Gateway'i yeniden başlatın.

## Yapılandırma

Kanal yapılandırması `channels.zalouser` altında bulunur (`plugins.entries.*` altında değil):

```json5
{
  channels: {
    zalouser: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

DM/grup erişim denetimi, çoklu hesap kurulumu, ortam değişkenleri ve sorun giderme
için [Zalo kişisel kanal yapılandırması](/tr/channels/zalouser) bölümüne bakın.

## CLI

```bash
openclaw channels login --channel zalouser
openclaw channels login --channel zalouser --account <name>
openclaw channels logout --channel zalouser
openclaw channels status --probe
openclaw message send --channel zalouser --target <threadId> --message "Hello from OpenClaw"
openclaw directory self --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory groups list --channel zalouser --query "name"
openclaw directory groups members --channel zalouser --group-id <id>
```

## Agent aracı

Araç adı: `zalouser`

Eylemler: `send`, `image`, `link`, `friends`, `groups`, `me`, `status`

Kanal mesajı eylemleri (agent aracı değil), mesaj
tepkileri için `react` eylemini de destekler.

## İlgili

- [Zalo kişisel kanal yapılandırması](/tr/channels/zalouser)
- [Zalo (resmî Bot/Webhook kanalı)](/tr/channels/zalo)
- [Plugin oluşturma](/tr/plugins/building-plugins)
- [ClawHub](/clawhub)
