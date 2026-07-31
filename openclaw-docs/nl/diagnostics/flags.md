---
read_when:
    - Je hebt gerichte debuglogboeken nodig zonder de algemene logniveaus te verhogen
    - Je moet subsysteemspecifieke logboeken vastleggen voor ondersteuning
summary: Diagnostische vlaggen voor gerichte debuglogboeken
title: Diagnostische vlaggen
x-i18n:
    generated_at: "2026-07-27T05:03:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3bdab6ba1fd98ba58c99c93f9a12d31f57e2655cb0c1eb2de09e34b970f56c
    source_path: diagnostics/flags.md
    workflow: 16
---

Diagnostische vlaggen schakelen extra logboekregistratie voor één subsysteem in zonder
`logging.level` globaal te verhogen. Een vlag heeft geen effect tenzij een subsysteem deze controleert.

## Hoe het werkt

- Vlaggen zijn hoofdletterongevoelige tekenreeksen, afgeleid uit `diagnostics.flags` in
  de configuratie plus de omgevingsoverschrijving `OPENCLAW_DIAGNOSTICS`, ontdubbeld en omgezet naar kleine letters.
- `name.*` komt overeen met `name` zelf en alles onder `name.` (bijvoorbeeld
  `telegram.*` komt overeen met `telegram.http`).
- `*` of `all` schakelt elke vlag in.
- Start de Gateway opnieuw nadat je `diagnostics.flags` in de configuratie hebt gewijzigd; deze wordt niet
  dynamisch opnieuw geladen.

## Bekende vlaggen

| Vlag                  | Schakelt in                                               |
| --------------------- | --------------------------------------------------------- |
| `telegram.http`       | Logboekregistratie van HTTP-fouten van de Telegram Bot API |
| `brave.http`          | Logboekregistratie van Brave Search-verzoeken, -antwoorden en -cache |
| `profiler`            | Profiler voor de antwoordfase en profiler voor de Codex-appserver (beide) |
| `reply.profiler`      | Alleen de profiler voor de antwoordfase                   |
| `codex.profiler`      | Alleen de profiler voor de Codex-appserver                |
| `health`              | Foutopsporingsdetails voor statuscontrole, account en binding van de Gateway |
| `ingress.timing`      | Timing van sessieladen, modelselectie en modelcatalogus    |
| `plugin.load-profile` | Timing van het synchroon laden van Plugin-modules         |
| `timeline`            | Gestructureerd JSONL-tijdlijnartefact (zie hieronder)     |

## Inschakelen via configuratie

```json
{
  "diagnostics": {
    "flags": ["telegram.http"]
  }
}
```

Meerdere vlaggen:

```json
{
  "diagnostics": {
    "flags": ["telegram.http", "brave.http", "gateway.*"]
  }
}
```

## Omgevingsoverschrijving (eenmalig)

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,brave.http
```

Waarden worden gesplitst op komma's of witruimte. Speciale waarden:

| Waarde                      | Effect                                   |
| --------------------------- | ---------------------------------------- |
| `0`, `false`, `off`, `none` | Schakelt alle vlaggen uit en overschrijft ook de configuratie |
| `1`, `true`, `all`, `*`     | Schakelt elke vlag in                    |

`OPENCLAW_DIAGNOSTICS=0` schakelt vlaggen uit zowel de omgeving als de configuratie uit voor dat
proces. Dit is handig om tijdelijk een profilervlag te onderdrukken die in de configuratie is blijven staan,
zonder het bestand te bewerken.

## Profilervlaggen

Profilervlaggen beheren lichtgewicht tijdmetingen; wanneer ze uitstaan, veroorzaken ze geen overhead.

Schakel voor één uitvoering van de Gateway alle door de profiler beheerde tijdmetingen in:

```bash
OPENCLAW_DIAGNOSTICS=profiler openclaw gateway run
```

Schakel alleen de profilermetingen voor antwoorddispatch in:

```bash
OPENCLAW_DIAGNOSTICS=reply.profiler openclaw gateway run
```

Schakel alleen de profilermetingen voor opstarten, tools en threads van de Codex-appserver in:

```bash
OPENCLAW_DIAGNOSTICS=codex.profiler openclaw gateway run
```

`profiler` schakelt zowel de antwoordprofiler als de Codex-profiler in; gebruik de
specifieke vlagnamen om er slechts één in te schakelen.

Of stel dit in de configuratie in:

```json
{
  "diagnostics": {
    "flags": ["reply.profiler", "codex.profiler"]
  }
}
```

Start de Gateway opnieuw nadat je configuratievlaggen hebt gewijzigd. Om een profilervlag uit te schakelen,
verwijder je deze uit `diagnostics.flags` en start je opnieuw, of start je het proces met
`OPENCLAW_DIAGNOSTICS=0` om elke diagnostische vlag voor die uitvoering te overschrijven.

## Tijdlijnartefacten

De vlag `timeline` (alias: `diagnostics.timeline`) schrijft gestructureerde timinggebeurtenissen voor opstarten
en uitvoering als JSONL, voor externe QA-harnassen:

```bash
OPENCLAW_DIAGNOSTICS=timeline \
OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=/tmp/openclaw-timeline.jsonl \
openclaw gateway run
```

Of schakel deze in de configuratie in:

```json
{
  "diagnostics": {
    "flags": ["timeline"]
  }
}
```

Het uitvoerpad komt altijd uit `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH`, zelfs
wanneer de vlag zelf in de configuratie is ingesteld; er is geen configuratiesleutel voor het pad.
Wanneer `timeline` alleen via de configuratie is ingeschakeld, ontbreken de vroegste tijdmetingen voor het laden van de configuratie,
omdat OpenClaw de configuratie dan nog niet heeft gelezen; latere opstartmetingen
worden normaal vastgelegd.

`OPENCLAW_DIAGNOSTICS=1`, `=all` en `=*` schakelen de tijdlijn ook in, omdat ze
elke vlag inschakelen. Geef de voorkeur aan de specifieke vlag `timeline` wanneer je alleen het
JSONL-artefact wilt en niet elke andere diagnostische vlag.

Voor metingen van vertraging in de gebeurtenislus in de tijdlijn is naast
`timeline` nog een extra expliciete inschakeling nodig: stel `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` (of `on`/`true`/`yes`) in
naast het inschakelen van de tijdlijn.

Tijdlijnrecords gebruiken de envelop `openclaw.diagnostics.v1` en kunnen
proces-id's, fasenamen, namen van tijdmetingen, duurwaarden, Plugin-id's, aantallen afhankelijkheden,
metingen van vertraging in de gebeurtenislus, namen van providerbewerkingen, de afsluitstatus van onderliggende processen
en namen/berichten van opstartfouten bevatten. Behandel tijdlijnbestanden als lokale
diagnostische artefacten; controleer ze voordat je ze buiten je computer deelt.

## Waar logboeken terechtkomen

Vlaggen schrijven logboekgegevens naar het standaardbestand voor diagnostische logboeken. Standaard:

```
/tmp/openclaw/openclaw-YYYY-MM-DD.log
```

Benoemde profielen gebruiken `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log`; 
`--dev` gebruikt bijvoorbeeld `openclaw-dev-YYYY-MM-DD.log`.

Als je `logging.file` instelt, gebruik dan in plaats daarvan dat pad. Logboeken zijn JSONL (één JSON-
object per regel). Redactie blijft van toepassing op basis van `logging.redactSensitive`.
Zie [Logboekregistratie](/nl/logging) voor het volledige model voor padbepaling, rotatie en
redactie van logboeken.

## Logboeken extraheren

Lees het nieuwste logboekbestand van het actieve profiel:

```bash
openclaw logs --plain
# Voorbeeld met benoemd profiel:
openclaw --profile work logs --plain
```

Filter op HTTP-diagnostiek van Telegram:

```bash
openclaw logs --plain --limit 5000 | rg "telegram http error"
```

Filter op HTTP-diagnostiek van Brave Search:

```bash
openclaw logs --plain --limit 5000 | rg "brave http"
```

Of volg het logboek tijdens het reproduceren:

```bash
openclaw logs --follow --plain | rg "telegram http error"
```

Gebruik voor externe Gateways in plaats daarvan `openclaw logs --follow` (zie
[/cli/logs](/nl/cli/logs)).

## Opmerkingen

- Als `logging.level` hoger is ingesteld dan `warn`, kunnen door vlaggen beheerde logboekberichten worden
  onderdrukt. De standaardwaarde `info` is geschikt.
- `brave.http` registreert verzoek-URL's/queryparameters van Brave Search, antwoordstatus/
  timing en gebeurtenissen voor cachetreffers, cachemissers en cacheschrijfacties. De API-sleutel
  (verzonden als verzoekheader) of antwoordinhoud wordt niet geregistreerd, maar zoekopdrachten kunnen
  gevoelig zijn.
- Vlaggen kunnen veilig ingeschakeld blijven; ze beïnvloeden alleen het logboekvolume voor het
  specifieke subsysteem.
- Gebruik [/logging](/nl/logging) om logboekbestemmingen, niveaus en redactie te wijzigen.

## Gerelateerd

- [Gateway-diagnostiek](/nl/gateway/diagnostics)
- [Problemen met de Gateway oplossen](/nl/gateway/troubleshooting)
