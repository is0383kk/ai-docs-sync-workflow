---
read_when:
    - Je wilt OpenClaw verbinden met WeChat of Weixin
    - Je installeert de kanaalplugin openclaw-weixin of lost er problemen mee op
    - Je moet begrijpen hoe externe kanaalplugins naast de Gateway worden uitgevoerd
summary: WeChat-kanaal instellen via de externe openclaw-weixin-plugin
title: WeChat
x-i18n:
    generated_at: "2026-07-27T04:57:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98faf95f9fb76deedb7df9adf3092083722a77bdd793de98c41a6f715cc0d14a
    source_path: channels/wechat.md
    workflow: 16
---

OpenClaw maakt verbinding met WeChat via de externe
`@tencent-weixin/openclaw-weixin`-kanaalplugin van Tencent.

Status: externe plugin, onderhouden door het Tencent Weixin-team. Directe chats en
media worden ondersteund. Groepschats worden niet vermeld in de
pluginmetadata voor mogelijkheden (deze vermeldt alleen directe chats).

## Naamgeving

- **WeChat** is de gebruikersgerichte naam in deze documentatie.
- **Weixin** is de naam die wordt gebruikt door het pakket van Tencent en door de plugin-id.
- `openclaw-weixin` is de OpenClaw-kanaal-id (`weixin` en `wechat` werken als aliassen).
- `@tencent-weixin/openclaw-weixin` is het npm-pakket.

Gebruik `openclaw-weixin` in CLI-opdrachten en configuratiepaden.

## Werking

De WeChat-code bevindt zich niet in de OpenClaw-kernrepository. OpenClaw biedt het
generieke contract voor kanaalplugins en de externe plugin biedt de
WeChat-specifieke runtime:

1. `openclaw plugins install` installeert `@tencent-weixin/openclaw-weixin`.
2. De Gateway ontdekt het pluginmanifest en laadt het ingangspunt van de plugin.
3. De plugin registreert kanaal-id `openclaw-weixin`.
4. `openclaw channels login --channel openclaw-weixin` start het aanmelden via QR-code.
5. De plugin slaat accountreferenties op in de OpenClaw-statusmap
   (standaard `~/.openclaw`).
6. Wanneer de Gateway start, start de plugin zijn Weixin-monitor voor elk
   geconfigureerd account.
7. Inkomende WeChat-berichten worden via het kanaalcontract genormaliseerd, naar
   de geselecteerde OpenClaw-agent gerouteerd en via het uitgaande pad van de plugin teruggestuurd.

Die scheiding is belangrijk: de OpenClaw-kern blijft kanaalonafhankelijk. Aanmelden
bij WeChat, Tencent iLink-API-aanroepen, uploaden/downloaden van media, contexttokens
en accountbewaking vallen onder de verantwoordelijkheid van de externe plugin.

## Installatie

Snelle installatie:

```bash
npx -y @tencent-weixin/openclaw-weixin-cli install
```

Handmatige installatie:

```bash
openclaw plugins install "@tencent-weixin/openclaw-weixin"
openclaw config set plugins.entries.openclaw-weixin.enabled true
```

Start de Gateway na de installatie opnieuw:

```bash
openclaw gateway restart
```

## Aanmelden

Voer de aanmelding via QR-code uit op dezelfde machine waarop de Gateway draait:

```bash
openclaw channels login --channel openclaw-weixin
```

Scan de QR-code met WeChat op je telefoon en bevestig de aanmelding. De plugin slaat
het accounttoken lokaal op na een geslaagde scan.

Voer dezelfde aanmeldopdracht opnieuw uit om nog een WeChat-account toe te voegen. Isoleer bij
meerdere accounts sessies voor directe berichten per account, kanaal en afzender:

```bash
openclaw config set session.dmScope per-account-channel-peer
```

## Toegangsbeheer

Directe berichten gebruiken het normale OpenClaw-model voor koppeling en
toestaanlijsten voor kanaalplugins.

Keur nieuwe afzenders goed:

```bash
openclaw pairing list openclaw-weixin
openclaw pairing approve openclaw-weixin <CODE>
```

Zie [Koppeling](/nl/channels/pairing) voor het volledige toegangsbeheermodel.

## Compatibiliteit

De plugin controleert bij het opstarten de versie van de OpenClaw-host.

| Pluginreeks | OpenClaw-versie                                                | npm-tag  |
| ----------- | --------------------------------------------------------------- | -------- |
| `2.x`       | `>=2026.5.12` (huidige 2.4.6; vroege 2.x accepteerde `>=2026.3.22`) | `latest` |
| `1.x`       | `>=2026.1.0 <2026.3.22`                                         | `legacy` |

Als de plugin meldt dat je OpenClaw-versie te oud is, werk je OpenClaw bij
of installeer je de verouderde pluginreeks:

```bash
openclaw plugins install @tencent-weixin/openclaw-weixin@legacy
```

## Sidecarproces

De WeChat-plugin kan naast de Gateway hulpwerk uitvoeren terwijl deze de
Tencent iLink-API bewaakt. In issue #68451 bracht dat hulppad een fout aan het licht in de
generieke opschoning van verouderde Gateways in OpenClaw: een onderliggend proces kon proberen het bovenliggende
Gateway-proces op te schonen, wat onder procesbeheerders zoals systemd herstartlussen veroorzaakte.

De huidige opschoning bij het opstarten van OpenClaw sluit het huidige proces en de bovenliggende processen ervan uit,
zodat een kanaalhulp het Gateway-proces dat deze heeft gestart niet kan beëindigen. Deze oplossing is
generiek; het is geen WeChat-specifiek pad in de kern.

## Problemen oplossen

Controleer de installatie en status:

```bash
openclaw plugins list
openclaw channels status --probe
openclaw --version
```

Als het kanaal als geïnstalleerd wordt weergegeven maar geen verbinding maakt, controleer dan of de plugin is
ingeschakeld en start opnieuw:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled true
openclaw gateway restart
```

Als de Gateway na het inschakelen van WeChat herhaaldelijk opnieuw start, werk dan zowel OpenClaw als
de plugin bij:

```bash
npm view @tencent-weixin/openclaw-weixin version
openclaw plugins install "@tencent-weixin/openclaw-weixin" --force
openclaw gateway restart
```

Als bij het opstarten wordt gemeld dat het geïnstalleerde pluginpakket `requires compiled runtime
output for TypeScript entry`, is het npm-pakket gepubliceerd zonder de gecompileerde
JavaScript-runtimebestanden die OpenClaw nodig heeft. Werk de plugin bij of installeer deze opnieuw nadat de
uitgever een hersteld pakket heeft uitgebracht, of schakel de plugin tijdelijk uit of verwijder deze.

Tijdelijk uitschakelen:

```bash
openclaw config set plugins.entries.openclaw-weixin.enabled false
openclaw gateway restart
```

## Gerelateerde documentatie

- Kanaaloverzicht: [Chatkanalen](/nl/channels)
- Koppeling: [Koppeling](/nl/channels/pairing)
- Kanaalroutering: [Kanaalroutering](/nl/channels/channel-routing)
- Pluginarchitectuur: [Pluginarchitectuur](/nl/plugins/architecture)
- SDK voor kanaalplugins: [SDK voor kanaalplugins](/nl/plugins/sdk-channel-plugins)
- Extern pakket: [@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
