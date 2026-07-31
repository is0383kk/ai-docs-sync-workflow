---
read_when:
    - Werken aan functies voor het Nextcloud Talk-kanaal
summary: Ondersteuningsstatus, mogelijkheden en configuratie van Nextcloud Talk
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-07-27T04:47:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59f4fe51555bcb13d630140866307b1a49ba077059818ec116ee50ef0c877b2b
    source_path: channels/nextcloud-talk.md
    workflow: 16
---

Nextcloud Talk is een downloadbare kanaalplugin (`@openclaw/nextcloud-talk`) die OpenClaw via een Talk-webhookbot verbindt met een zelfgehoste Nextcloud-instantie. Directe berichten, ruimtes, reacties en markdown-berichten worden ondersteund; media worden als URL's verzonden.

## Installeren

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

Gebruik de kale pakketspecificatie om de huidige officiële releasetag te volgen. Zet alleen een exacte versie vast wanneer je een reproduceerbare installatie nodig hebt.

Vanuit een lokale checkout (ontwikkelworkflows):

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

Start de Gateway na de installatie opnieuw. Details: [Plugins](/nl/tools/plugin)

## Snelle configuratie (beginner)

1. Installeer de plugin (hierboven).
2. Maak op je Nextcloud-server een bot:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature webhook --feature response --feature reaction
   ```

   Behoud `--feature response`: zonder dit mislukken uitgaande antwoorden met 401. Herstel een bestaande bot met `./occ talk:bot:state --feature webhook --feature response --feature reaction <botId> 1`.

3. Schakel de bot in via de instellingen van de doelruimte.
4. Configureer OpenClaw:
   - Configuratie: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - Of via de omgeving: `NEXTCLOUD_TALK_BOT_SECRET` (alleen standaardaccount)

   CLI-configuratie (`--url`/`--token` zijn aliassen voor de expliciete velden; `nc-talk` en `nc` werken als kanaalaliassen):

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --url https://cloud.example.com \
     --token "<shared-secret>"
   ```

   Gelijkwaardige expliciete velden:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret "<shared-secret>"
   ```

   Bestandsgebaseerd geheim:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret-file /path/to/nextcloud-talk-secret
   ```

5. Start de Gateway opnieuw (of voltooi de configuratie).

Minimale configuratie:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## Opmerkingen

- Bots kunnen geen DM's initiëren. De gebruiker moet eerst een bericht naar de bot sturen.
- De webhook-URL moet vanaf de Nextcloud-server bereikbaar zijn; stel `webhookPublicUrl` in wanneer de Gateway zich achter een proxy bevindt. Webhookverzoeken worden met het botgeheim via HMAC-SHA256 ondertekend; ongeldige handtekeningen worden geweigerd en beperkt in aanvraagsnelheid.
- Media-uploads worden niet ondersteund door de bot-API; uitgaande media worden toegevoegd als een regel met `Attachment: <url>`.
- De webhookpayload maakt geen onderscheid tussen DM's en ruimtes; stel `apiUser` + `apiPassword` in om opzoekacties voor het ruimtetype in te schakelen (ongeveer 5 minuten gecachet). Zonder deze instellingen wordt elk gesprek als een ruimte behandeld.
- Uitgaande verzoeken lopen via de SSRF-beveiliging. Meld je met `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork: true` aan voor een Nextcloud-host op een vertrouwd privé/intern netwerk.
- Als `apiUser`/`apiPassword` en `webhookPublicUrl` zijn ingesteld, controleert `openclaw channels status` de bot en waarschuwt het wanneer de functie `response` ontbreekt.

## Toegangsbeheer (DM's)

- Standaard: `channels.nextcloud-talk.dmPolicy = "pairing"`. Onbekende afzenders krijgen een koppelingscode.
- Goedkeuren via:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- Openbare DM's: `channels.nextcloud-talk.dmPolicy="open"` plus `channels.nextcloud-talk.allowFrom=["*"]`.
- `allowFrom` komt alleen overeen met Nextcloud-gebruikers-ID's (in kleine letters); weergavenamen worden genegeerd.

## Ruimtes (groepen)

- Standaard: `channels.nextcloud-talk.groupPolicy = "allowlist"` (vermelding vereist).
- Sta ruimtes toe met `channels.nextcloud-talk.rooms`, geïndexeerd op ruimtetoken; `"*"` stelt een standaard met jokerteken in:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- Sleutels per ruimte: `requireMention` (standaard true), `enabled` (false schakelt de ruimte uit), `allowFrom` (lijst met toegestane afzenders per ruimte), `tools` (overschrijvingen om tools toe te staan/te weigeren), `skills` (beperk geladen Skills), `systemPrompt`.
- Houd de lijst met toegestane ruimtes leeg of stel `channels.nextcloud-talk.groupPolicy="disabled"` in om geen ruimtes toe te staan.

## Mogelijkheden

| Functie            | Status               |
| ------------------ | -------------------- |
| Directe berichten  | Ondersteund          |
| Ruimtes            | Ondersteund          |
| Threads            | Niet ondersteund     |
| Media              | Alleen URL's         |
| Reacties           | Ondersteund          |
| Systeemeigen opdrachten | Niet ondersteund |

## Configuratiereferentie (Nextcloud Talk)

Volledige configuratie: [Configuratie](/nl/gateway/configuration)

Provideropties:

- `channels.nextcloud-talk.enabled`: opstarten van het kanaal in-/uitschakelen.
- `channels.nextcloud-talk.baseUrl`: URL van de Nextcloud-instantie.
- `channels.nextcloud-talk.botSecret`: gedeeld botgeheim (tekenreeks of geheime verwijzing).
- `channels.nextcloud-talk.botSecretFile`: pad naar een geheim in een regulier bestand. Symbolische koppelingen worden geweigerd.
- `channels.nextcloud-talk.apiUser`: API-gebruiker voor het opzoeken van ruimtes (DM-detectie) en de statuscontrole.
- `channels.nextcloud-talk.apiPassword`: API-/app-wachtwoord voor het opzoeken van ruimtes.
- `channels.nextcloud-talk.apiPasswordFile`: bestandspad van het API-wachtwoord.
- `channels.nextcloud-talk.webhookPort`: poort van de webhooklistener (standaard: 8788).
- `channels.nextcloud-talk.webhookHost`: webhookhost (standaard: 0.0.0.0).
- `channels.nextcloud-talk.webhookPath`: webhookpad (standaard: /nextcloud-talk-webhook).
- `channels.nextcloud-talk.webhookPublicUrl`: extern bereikbare webhook-URL.
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled` (standaard: koppelen). `open` vereist `allowFrom=["*"]`.
- `channels.nextcloud-talk.allowFrom`: lijst met toegestane DM-gebruikers (gebruikers-ID's).
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled` (standaard: lijst met toegestane items).
- `channels.nextcloud-talk.groupAllowFrom`: lijst met toegestane afzenders in ruimtes (gebruikers-ID's); valt terug op `allowFrom` wanneer niet ingesteld.
- `channels.nextcloud-talk.rooms`: instellingen per ruimte en lijst met toegestane ruimtes (zie hierboven).
- Naar statische toegangsgroepen voor afzenders kan vanuit `allowFrom` en `groupAllowFrom` worden verwezen met `accessGroup:<name>`.
- `channels.nextcloud-talk.historyLimit`: limiet voor groepsgeschiedenis (0 schakelt dit uit).
- `channels.nextcloud-talk.dmHistoryLimit`: limiet voor DM-geschiedenis (0 schakelt dit uit).
- `channels.nextcloud-talk.dms`: overschrijvingen per DM, geïndexeerd op gebruikers-ID (`historyLimit`).
- `channels.nextcloud-talk.textChunkLimit`: segmentgrootte van uitgaande tekst in tekens (standaard: 4000).
- `channels.nextcloud-talk.streaming.chunkMode`: `length` (standaard) of `newline` om vóór segmentering op lengte te splitsen op lege regels (alineagrenzen).
- `channels.nextcloud-talk.streaming.block.enabled`: blokstreaming voor dit kanaal in- of uitschakelen.
- `channels.nextcloud-talk.streaming.block.coalesce`: afstemming voor het samenvoegen van blokstreaming.
- `channels.nextcloud-talk.responsePrefix`: voorvoegsel voor uitgaande antwoorden.
- `channels.nextcloud-talk.markdown.tables`: weergavemodus voor markdown-tabellen (`off | bullets | code | block`).
- `channels.nextcloud-talk.mediaMaxMb`: limiet voor inkomende media (MB).
- `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork`: privé/interne Nextcloud-hosts door de SSRF-beveiliging toelaten.
- `channels.nextcloud-talk.accounts.<id>`: overschrijvingen per account (dezelfde sleutels); `defaultAccount` kiest de standaard. Omgevingsvariabelen `NEXTCLOUD_TALK_BOT_SECRET` / `NEXTCLOUD_TALK_API_PASSWORD` zijn alleen van toepassing op het standaardaccount.

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsproces
- [Groepen](/nl/channels/groups) — gedrag van groepschats en vereiste vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en versterking
