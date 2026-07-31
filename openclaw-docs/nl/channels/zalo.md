---
read_when:
    - Werken aan Zalo-functies of webhooks
summary: Ondersteuningsstatus, mogelijkheden en configuratie van Zalo-bots
title: Zalo
x-i18n:
    generated_at: "2026-07-27T05:37:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f3e0bfe6003d3b2f38411fcc5a4e82266733b042693c7853d0b3c8a3864273c5
    source_path: channels/zalo.md
    workflow: 16
---

Status: experimenteel. Directe berichten en groepschats zijn beide geïmplementeerd; de onderstaande tabel [Mogelijkheden](#capabilities) geeft geverifieerd gedrag weer voor Zalo Bot Creator-/Marketplace-bots.

## Gebundelde Plugin

Zalo wordt als gebundelde Plugin meegeleverd in de huidige OpenClaw-releases, dus voor verpakte builds is geen afzonderlijke installatie nodig.

Installeer bij een oudere build of een aangepaste installatie zonder Zalo het npm-pakket rechtstreeks:

- Installeren: `openclaw plugins install @openclaw/zalo`
- Vastgezette versie: `openclaw plugins install @openclaw/zalo@2026.6.11`
- Vanuit een lokale checkout: `openclaw plugins install ./path/to/local/zalo-plugin`
- Details: [Plugins](/nl/tools/plugin)

## Snelle configuratie

1. Maak een bottoken aan op [https://bot.zaloplatforms.com](https://bot.zaloplatforms.com) (meld je aan, maak een bot en configureer de instellingen). Het token is `numeric_id:secret`; voor Marketplace-bots kan het bruikbare runtimetoken in het welkomstbericht van de bot staan.
2. Stel het token in via de omgevingsvariabele `ZALO_BOT_TOKEN=...` (alleen voor het standaardaccount) of in de configuratie.
3. Start de Gateway opnieuw.
4. Keur bij het eerste contact via een direct bericht de koppelingscode goed (het standaardbeleid voor directe berichten is koppeling).

Minimale configuratie:

```json5
{
  channels: {
    zalo: {
      enabled: true,
      accounts: {
        default: {
          botToken: "12345689:abc-xyz",
          dmPolicy: "pairing",
        },
      },
    },
  },
}
```

Meerdere accounts: voeg meer vermeldingen toe onder `channels.zalo.accounts.<id>`, elk met een eigen `botToken`/`name`. `channels.zalo.botToken` (vlak, zonder `accounts`) is een verouderde verkorte notatie voor één account; geef voor nieuwe configuraties de voorkeur aan `accounts.<id>.*`.

## Wat het is

Zalo is een berichtenapp die zich op Vietnam richt. Met de Bot API kan de Gateway een bot uitvoeren voor zowel 1:1-gesprekken als groepschats, met deterministische routering terug naar Zalo (het model kiest nooit kanalen).

Deze pagina behandelt **Zalo Bot Creator-/Marketplace-bots**. **Zalo Official Account-bots (OA)** vormen een ander productoppervlak en kunnen zich anders gedragen; deze pagina behandelt ze niet.

## Hoe het werkt

- Inkomende berichten worden met mediaplaatshouders genormaliseerd naar de gedeelde kanaalenvelop.
- Antwoorden worden altijd teruggeleid naar dezelfde Zalo-chat; geciteerd antwoorden wordt niet gebruikt (`replyToMode` staat permanent uit).
- Standaard wordt long-polling (`getUpdates`) gebruikt; de webhookmodus is beschikbaar via `channels.zalo.webhookUrl`.
- In groepen is een @vermelding vereist om de bot te activeren; dit is niet per kanaal configureerbaar.

## Limieten

| Limiet                        | Waarde                                                                   |
| ----------------------------- | ------------------------------------------------------------------------ |
| Segmentgrootte uitgaande tekst | 2000 tekens (limiet van de Zalo API)                                    |
| Mediagrootte (inkomend/uitgaand) | `channels.zalo.mediaMaxMb`, standaard `5` MB                    |
| Aanvraagbody van webhook      | 1 MB, leestime-out van 30s                                                |
| Snelheidslimiet van webhook   | 120 aanvragen / 60s per pad+client-IP, daarna HTTP 429                   |
| Tombstones voor webhookherhaling | 30 dagen, maximaal 20.000 voltooide gebeurtenissen per account (op bericht-id geïndexeerd) |

## Toegangsbeheer

### Directe berichten

- `channels.zalo.dmPolicy`: `pairing` (standaard) | `allowlist` | `open` | `disabled`.
- Koppeling: onbekende afzenders krijgen een koppelingscode; berichten worden genegeerd totdat deze is goedgekeurd. Codes verlopen na 1 uur.
  - `openclaw pairing list zalo`
  - `openclaw pairing approve zalo <CODE>`
  - Details: [Koppeling](/nl/channels/pairing)
- `channels.zalo.allowFrom` accepteert numerieke Zalo-gebruikers-ID's (geen zoekactie op gebruikersnaam). Voor `open` is `"*"` vereist.

### Groepen

Groepschats worden door de Plugin ondersteund (`chatTypes: ["direct", "group"]`) en worden beperkt door een vermelding en het groepsbeleid:

- `channels.zalo.groupPolicy`: `open` | `allowlist` | `disabled`.
- `channels.zalo.groupAllowFrom` beperkt welke afzender-ID's de bot in groepen kunnen activeren; wanneer dit niet is ingesteld, wordt teruggevallen op `allowFrom`.
- Standaardresolutie: wanneer `channels.zalo` is geconfigureerd, wordt een niet-ingestelde `groupPolicy` opgelost als `open`. Wanneer `channels.zalo` volledig ontbreekt, weigert de runtime standaard toegang via `allowlist`.
- Gemelde praktijkbeperking: bij sommige Marketplace-botconfiguraties kon de bot helemaal niet aan een groep worden toegevoegd. Als dit gebeurt, controleer dan de Zalo Bot Platform-instellingen van je bot; dit is een beperking van het platform, geen beleid van OpenClaw.

## Long-polling versus webhook

- Standaard: long-polling (geen openbare URL vereist).
- Webhookmodus: stel `channels.zalo.webhookUrl` en `channels.zalo.webhookSecret` in.
  - De webhook-URL moet HTTPS gebruiken.
  - Het webhookgeheim moet 8-256 tekens lang zijn.
  - Zalo verzendt gebeurtenissen met een `X-Bot-Api-Secret-Token`-header, die wordt gecontroleerd met een vergelijking met constante uitvoeringstijd.
  - Gateway HTTP verwerkt webhookaanvragen op `channels.zalo.webhookPath` (standaard het pad van de webhook-URL).
  - Aanvragen moeten `Content-Type: application/json` (of een `+json`-mediatype) gebruiken.
  - HTTP 200 wordt pas geretourneerd nadat de onbewerkte gebeurtenis duurzaam is opgeslagen; opslagfouten retourneren HTTP 500.
  - Volgens de documentatie van de Zalo API sluiten getUpdates-polling en een webhook elkaar wederzijds uit.

## Ondersteunde berichttypen

- Tekst: volledig ondersteund, opgesplitst in segmenten van 2000 tekens.
- Media: inkomend/uitgaand, beperkt door `mediaMaxMb`.
- Reacties, threads, peilingen en native opdrachten: niet ondersteund door de Plugin.
- Streaming: de Plugin declareert ondersteuning voor blokstreaming, maar Zalo heeft geen specifieke instelopties voor een uitgaande wachtrij of het samenvoegen van tekst (in tegenstelling tot sommige andere regionale kanalen); verifieer het huidige gedrag in je omgeving als dit voor jouw gebruikssituatie van belang is.

## Mogelijkheden

| Functie                  | Status                            |
| ------------------------ | --------------------------------- |
| Directe berichten        | Ondersteund                       |
| Groepen                  | Ondersteund (vermelding vereist)  |
| Media (inkomend/uitgaand) | Ondersteund, beperkt door `mediaMaxMb` |
| Reacties                 | Niet ondersteund                  |
| Threads                  | Niet ondersteund                  |
| Peilingen                | Niet ondersteund                  |
| Native opdrachten        | Niet ondersteund                  |
| Antwoorden op / citeren  | Niet gebruikt (permanent uit)     |

## Afleveringsdoelen (CLI/cron)

Gebruik een chat-ID als doel:

```bash
openclaw message send --channel zalo --target 123456789 --message "hi"
```

## Probleemoplossing

**Bot reageert niet:**

- Controleer het token: `openclaw channels status --probe`
- Controleer of de afzender is goedgekeurd (koppeling of `allowFrom`)
- Controleer de Gateway-logboeken: `openclaw logs --follow`

**Webhook ontvangt geen gebeurtenissen:**

- Controleer of de webhook-URL HTTPS gebruikt
- Controleer of het geheim 8-256 tekens lang is
- Controleer of het HTTP-eindpunt van de Gateway bereikbaar is via het geconfigureerde pad
- Controleer of getUpdates-polling niet ook actief is (ze sluiten elkaar wederzijds uit)
- Een piek in het aantal aanvragen kan HTTP 429 opleveren (120 aanvragen / 60s per pad+IP); wacht langer tussen pogingen en probeer het opnieuw

## Configuratiereferentie

Volledige configuratie: [Configuratie](/nl/gateway/configuration)

| Instelling                                    | Beschrijving                                      | Standaard              |
| -------------------------------------------- | ------------------------------------------------- | --------------------- |
| `channels.zalo.enabled`                      | Opstarten van kanaal in-/uitschakelen             | `true`                |
| `channels.zalo.accounts.<id>.botToken`       | Bottoken van Zalo Bot Platform                    | -                     |
| `channels.zalo.accounts.<id>.tokenFile`      | Token uit een bestand lezen (symbolische koppelingen geweigerd) | -                     |
| `channels.zalo.accounts.<id>.name`           | Weergavenaam                                      | -                     |
| `channels.zalo.accounts.<id>.enabled`        | Dit account in-/uitschakelen                      | `true`                |
| `channels.zalo.accounts.<id>.dmPolicy`       | Beleid voor directe berichten per account         | `pairing`             |
| `channels.zalo.accounts.<id>.allowFrom`      | Toegestane gebruikers voor directe berichten (gebruikers-ID's) | -                     |
| `channels.zalo.accounts.<id>.groupPolicy`    | Groepsbeleid per account                          | zie [Groepen](#groups) |
| `channels.zalo.accounts.<id>.groupAllowFrom` | Toegestane groepsafzenders; valt terug op `allowFrom` | -                     |
| `channels.zalo.accounts.<id>.mediaMaxMb`     | Limiet voor inkomende/uitgaande media (MB)         | `5`                   |
| `channels.zalo.accounts.<id>.webhookUrl`     | Webhookmodus inschakelen (HTTPS vereist)           | -                     |
| `channels.zalo.accounts.<id>.webhookSecret`  | Webhookgeheim (8-256 tekens)                      | -                     |
| `channels.zalo.accounts.<id>.webhookPath`    | Webhookpad op de HTTP-server van de Gateway        | pad van webhook-URL   |
| `channels.zalo.accounts.<id>.proxy`          | Proxy-URL voor API-aanvragen                      | -                     |
| `channels.zalo.accounts.<id>.responsePrefix` | Overschrijving van voorvoegsel voor uitgaande antwoorden | -                     |
| `channels.zalo.defaultAccount`               | Standaardaccount wanneer meerdere zijn geconfigureerd | `default`             |

`channels.zalo.botToken`, `channels.zalo.dmPolicy` en andere vlakke sleutels op het hoogste niveau vormen de verouderde verkorte notatie voor één account voor de bovenstaande velden; beide vormen worden ondersteund.

Omgevingsoptie: `ZALO_BOT_TOKEN=...` levert alleen het token van het standaardaccount op.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) - alle ondersteunde kanalen
- [Koppeling](/nl/channels/pairing) - authenticatie voor directe berichten en koppelingsflow
- [Groepen](/nl/channels/groups) - gedrag van groepschats en vereiste vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) - sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) - toegangsmodel en beveiliging aanscherpen
