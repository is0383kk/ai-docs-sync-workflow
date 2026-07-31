---
read_when:
    - Je wijzigt hoe tijdstempels aan het model of gebruikers worden weergegeven
    - Je debugt de tijdnotatie in berichten of de uitvoer van de systeemprompt
summary: Datum- en tijdverwerking in enveloppen, prompts, tools en connectors
title: Datum en tijd
x-i18n:
    generated_at: "2026-07-27T06:13:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e6f923022c021c1cf18ba306cd7b9a4873f5df947bb9a8fae9c737a89f64cbf2
    source_path: date-time.md
    workflow: 16
---

OpenClaw gebruikt **hostlokale tijd voor transporttijdstempels** en neemt **alleen de tijdzone** op in de systeemprompt.
Tijdstempels van providers blijven behouden, zodat tools hun eigen semantiek behouden. Wanneer de agent de huidige
tijd nodig heeft, voert deze de tool `session_status` uit.

## Berichtomhulsels (standaard lokaal)

Inkomende berichten worden voorzien van een weekdag en een tijdstempel met precisie tot op de seconde:

```
[WhatsApp +1555 Mon 2026-01-05 16:26:34 PST] berichttekst
```

De tijdstempel van het omhulsel is **standaard hostlokaal**, ongeacht de tijdzone van de provider.
Overschrijf dit onder `agents.defaults`:

```json5
{
  agents: {
    defaults: {
      envelopeTimezone: "local", // "utc" | "local" | "user" | IANA-tijdzone
      envelopeTimestamp: "on", // "on" | "off"
      envelopeElapsed: "on", // "on" | "off"
    },
  },
}
```

| Sleutel             | Waarden                                              | Gedrag                                                                                                                                                                          |
| ------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `envelopeTimezone`  | `local` (standaard), `utc`, `user`, expliciete IANA-naam | `user` gebruikt `agents.defaults.userTimezone` (hosttijdzone wanneer niet ingesteld). Een expliciete IANA-naam (bijv. `"America/Chicago"`) legt een vaste zone vast; niet-herkende namen vallen terug op UTC. |
| `envelopeTimestamp` | `on` (standaard), `off`                                | `off` verwijdert absolute tijdstempels uit omhulselkoppen, directe voorvoegsels van agentprompts en ingebedde voorvoegsels van modelinvoer.                                                       |
| `envelopeElapsed`   | `on` (standaard), `off`                                | `off` verwijdert het achtervoegsel met verstreken tijd (in de stijl `+30s` / `+2m`) dat de tijd sinds het vorige bericht in de sessie toont.                                                               |

### Voorbeelden

**Lokaal (standaard):**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 PST] hallo
```

**Tijdzone van de gebruiker:**

```
[WhatsApp +1555 Sun 2026-01-18 00:19:42 CST] hallo
```

**Verstreken tijd met `envelopeTimezone: "utc"`:**

```
[WhatsApp +1555 +30s Sun 2026-01-18T05:19:00Z] vervolgbericht
```

## Systeemprompt: huidige datum en tijd

De systeemprompt bevat een sectie **Huidige datum en tijd** met **alleen de tijdzone**
(geen kloktijd of tijdnotatie), zodat promptcaching stabiel blijft:

```
Tijdzone: America/Chicago
```

De zone is `agents.defaults.userTimezone` wanneer deze is geconfigureerd, anders de hosttijdzone.
De prompt instrueert de agent ook om de tool `session_status` uit te voeren wanneer deze de
huidige datum, tijd of dag van de week nodig heeft.

## Systeemgebeurtenisregels (standaard lokaal)

Systeemgebeurtenissen in de wachtrij die in de agentcontext worden ingevoegd, krijgen een tijdstempel als voorvoegsel met
dezelfde `envelopeTimezone`-selectie als berichtomhulsels (standaard: hostlokaal).

```
Systeem: [2026-01-12 12:19:17 PST] Model gewijzigd.
```

### Tijdzone en notatie van de gebruiker configureren

```json5
{
  agents: {
    defaults: {
      userTimezone: "America/Chicago",
      timeFormat: "auto", // auto | 12 | 24
    },
  },
}
```

- `userTimezone` stelt de **gebruikerslokale tijdzone** in voor de promptcontext (en voor `envelopeTimezone: "user"`).
- `timeFormat` bepaalt de **12-/24-uursweergave** voor tijden in prompts. `auto` volgt de voorkeuren van het besturingssysteem.

## Detectie van tijdnotatie (automatisch)

Wanneer `timeFormat: "auto"`, controleert OpenClaw de voorkeur van het besturingssysteem (macOS en Windows)
en valt het terug op de notatie van de landinstelling. De gedetecteerde waarde wordt **per proces gecachet**
om herhaalde systeemaanroepen te voorkomen.

## Toolpayloads en connectors (onbewerkte providertijd en genormaliseerde velden)

Kanaaltools retourneren **tijdstempels in de eigen notatie van de provider** en voegen voor consistentie genormaliseerde velden toe:

- `timestampMs`: epoch-milliseconden (UTC)
- `timestampUtc`: ISO 8601 UTC-tekenreeks

Onbewerkte providervelden blijven behouden, zodat niets verloren gaat.

- Discord: UTC-tijdstempels in ISO-notatie
- Slack: epoch-achtige tekenreeksen van de API
- Telegram/WhatsApp: providerspecifieke numerieke/ISO-tijdstempels

Als je lokale tijd nodig hebt, converteer je deze verderop met behulp van de bekende tijdzone.

## Gerelateerde documentatie

- [Systeemprompt](/nl/concepts/system-prompt)
- [Tijdzones](/nl/concepts/timezone)
- [Berichten](/nl/concepts/messages)
