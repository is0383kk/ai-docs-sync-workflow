---
read_when:
    - Je wilt dat OpenClaw privéberichten ontvangt via Nostr
    - Je stelt gedecentraliseerde berichtenuitwisseling in
summary: Nostr-DM-kanaal via NIP-04-versleutelde berichten
title: Nostr
x-i18n:
    generated_at: "2026-07-27T04:56:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 31fa283f706036a37795ddad71602058ba94388a9cb01044927c4bb2d83ba4a8
    source_path: channels/nostr.md
    workflow: 16
---

Nostr is een downloadbare kanaalplugin (`@openclaw/nostr`) waarmee OpenClaw versleutelde directe NIP-04-berichten via Nostr-relays kan ontvangen en beantwoorden. Eén account per Gateway; alleen DM's.

## Installeren

```bash
openclaw plugins install @openclaw/nostr
```

Gebruik de kale pakketspecificatie om de huidige officiële releasetag te volgen. Zet alleen een exacte versie vast wanneer je een reproduceerbare installatie nodig hebt.

Vanuit een lokale checkout (ontwikkelworkflows):

```bash
openclaw plugins install --link <path-to-local-nostr-plugin>
```

Start de Gateway opnieuw nadat je plugins hebt geïnstalleerd of ingeschakeld. Onboarding (`openclaw onboard`) en `openclaw channels add` tonen Nostr vanuit de gedeelde kanaalcatalogus zodra de plugin is geïnstalleerd.

### Niet-interactieve configuratie

```bash
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY"
openclaw channels add --channel nostr --private-key "$NOSTR_PRIVATE_KEY" --relay-urls "wss://relay.damus.io,wss://relay.primal.net"
```

Gebruik `--use-env` om `NOSTR_PRIVATE_KEY` in de omgeving te bewaren in plaats van de sleutel in de configuratie op te slaan (alleen voor het standaardaccount).

## Snelle configuratie

1. Genereer een Nostr-sleutelpaar (indien nodig):

```bash
# Met nak
nak key generate
```

2. Voeg dit toe aan de configuratie:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
    },
  },
}
```

3. Exporteer de sleutel:

```bash
export NOSTR_PRIVATE_KEY="nsec1..."
```

4. Start de Gateway opnieuw.

## Configuratiereferentie

| Sleutel          | Type     | Standaardwaarde                             | Beschrijving                                             |
| ------------ | -------- | ------------------------------------------- | -------------------------------------------------------- |
| `privateKey` | string   | vereist                                     | Privésleutel in `nsec`- of hex-indeling; geheime verwijzingen toegestaan |
| `relays`     | string[] | `['wss://relay.damus.io', 'wss://nos.lol']` | Relay-URL's (WebSocket)                                  |
| `dmPolicy`   | string   | `pairing`                                   | Toegangsbeleid voor DM's                                 |
| `allowFrom`  | string[] | `[]`                                        | Toegestane publieke sleutels van afzenders               |
| `enabled`    | boolean  | `true`                                      | Kanaal in-/uitschakelen                                  |
| `name`       | string   | -                                           | Weergavenaam                                             |
| `profile`    | object   | -                                           | NIP-01-profielmetadata                                   |

## Profielmetadata

Profielgegevens worden gepubliceerd als een NIP-01-`kind:0`-event. Je kunt ze beheren via de Control UI (Channels -> Nostr -> Profile) of rechtstreeks instellen in de configuratie.

Voorbeeld:

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      profile: {
        name: "openclaw",
        displayName: "OpenClaw",
        about: "Persoonlijke assistent voor DM's",
        picture: "https://example.com/avatar.png",
        banner: "https://example.com/banner.png",
        website: "https://example.com",
        nip05: "openclaw@example.com",
        lud16: "openclaw@example.com",
      },
    },
  },
}
```

Opmerkingen:

- Profiel-URL's moeten `https://` gebruiken.
- Bij importeren vanuit relays worden velden samengevoegd en lokale overschrijvingen behouden.

## Toegangsbeheer

### DM-beleid

- **pairing** (standaard): onbekende afzenders krijgen een koppelingscode.
- **allowlist**: alleen publieke sleutels in `allowFrom` kunnen DM's sturen.
- **open**: openbaar inkomende DM's (vereist `allowFrom: ["*"]`).
- **disabled**: inkomende DM's negeren.

Opmerkingen over handhaving:

- Handtekeningen van inkomende events worden vóór het afzenderbeleid en de NIP-04-ontsleuteling geverifieerd, zodat vervalste events vroegtijdig worden geweigerd.
- Koppelingsantwoorden worden verzonden zonder de inhoud van de oorspronkelijke DM te ontsleutelen of te verwerken.
- Voor inkomende DM's gelden frequentielimieten (globaal en per afzender) en te grote payloads worden vóór ontsleuteling verwijderd.

### Voorbeeld van een toelatingslijst

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      dmPolicy: "allowlist",
      allowFrom: ["npub1abc...", "npub1xyz..."],
    },
  },
}
```

## Sleutelindelingen

Geaccepteerde indelingen:

- **Privésleutel:** `nsec...` of hex van 64 tekens
- **Publieke sleutels (`allowFrom`):** `npub...` of hex

## Relays

Standaardwaarden: `relay.damus.io` en `nos.lol`.

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["wss://relay.damus.io", "wss://relay.primal.net", "wss://nostr.wine"],
    },
  },
}
```

Tips:

- Gebruik 2-3 relays voor redundantie.
- Vermijd te veel relays (latentie, duplicatie).
- Betaalde relays kunnen de betrouwbaarheid verbeteren.
- Lokale relays zijn geschikt voor tests (`ws://localhost:7777`).

## Protocolondersteuning

| NIP    | Status       | Beschrijving                              |
| ------ | ------------ | ----------------------------------------- |
| NIP-01 | Ondersteund  | Basisindeling voor events + profielmetadata |
| NIP-04 | Ondersteund  | Versleutelde DM's (`kind:4`)    |
| NIP-17 | Gepland      | In cadeauverpakking verpakte DM's         |
| NIP-44 | Gepland      | Versleuteling met versiebeheer            |

## Testen

### Lokale relay

```bash
# Start strfry
docker run -p 7777:7777 ghcr.io/hoytech/strfry
```

```json5
{
  channels: {
    nostr: {
      privateKey: "${NOSTR_PRIVATE_KEY}",
      relays: ["ws://localhost:7777"],
    },
  },
}
```

### Handmatige test

1. Noteer de publieke sleutel van de bot uit de Gateway-logboeken of `openclaw channels status` (hex; converteer indien nodig naar npub in je client).
2. Open een Nostr-client (Amethyst, Damus enzovoort).
3. Stuur een DM naar de publieke sleutel van de bot.
4. Controleer het antwoord.

## Probleemoplossing

### Geen berichten ontvangen

- Controleer of de privésleutel geldig is.
- Zorg dat relay-URL's bereikbaar zijn en `wss://` gebruiken (of `ws://` voor lokaal gebruik).
- Controleer of `enabled` niet `false` is.
- Controleer de Gateway-logboeken op verbindingsfouten met relays.

### Geen antwoorden verzenden

- Controleer of de relay schrijfbewerkingen accepteert.
- Controleer de uitgaande connectiviteit.
- Let op frequentielimieten van relays.

### Dubbele antwoorden

- Dit is te verwachten bij gebruik van meerdere relays.
- Berichten worden op event-ID gededupliceerd; alleen de eerste aflevering activeert een antwoord.

## Beveiliging

- Commit privésleutels nooit.
- Gebruik omgevingsvariabelen voor sleutels.
- Overweeg `allowlist` voor productiebots.
- Handtekeningen worden vóór het afzenderbeleid geverifieerd en het afzenderbeleid wordt vóór ontsleuteling gehandhaafd, zodat vervalste events vroegtijdig worden geweigerd en onbekende afzenders geen volledige cryptografische verwerking kunnen afdwingen.

## Beperkingen (MVP)

- Alleen directe berichten (geen groepschats).
- Geen mediabijlagen.
- Alleen NIP-04 (NIP-17-cadeauverpakking gepland).

## Gerelateerd

- [Overzicht van kanalen](/nl/channels) — alle ondersteunde kanalen
- [Koppelen](/nl/channels/pairing) — DM-authenticatie en koppelingsflow
- [Groepen](/nl/channels/groups) — gedrag van groepschats en beperking op basis van vermeldingen
- [Kanaalroutering](/nl/channels/channel-routing) — sessieroutering voor berichten
- [Beveiliging](/nl/gateway/security) — toegangsmodel en beveiliging aanscherpen
