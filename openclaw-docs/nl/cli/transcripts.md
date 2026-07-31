---
read_when:
    - Je wilt opgeslagen transcriptsamenvattingen vanuit de terminal lezen
    - Je hebt het pad naar een Markdown-samenvatting van transcripten nodig
    - Je debugt de opslagindeling van de kerntranscripten
summary: CLI-referentie voor `openclaw transcripts` (opgeslagen transcripten weergeven, bekijken en exporteren)
title: Transcript-CLI
x-i18n:
    generated_at: "2026-07-27T04:55:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

Inspectie- en exportopdracht voor duurzame vergadertranscripten. Browserdeelnemers aan Google Meet,
Microsoft Teams en Zoom leggen automatisch notities vast;
de agenttool `transcripts` ondersteunt ook vastlegging via providers en handmatige import.

De canonieke transcriptstatus bevindt zich in de gedeelde SQLite-database op
`$OPENCLAW_STATE_DIR/state/openclaw.sqlite`. `show` en `path` materialiseren expliciet
gebruikersgerichte artefacten onder de statusmap:

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

Deze bestanden zijn exports, geen tweede runtime-opslag. OpenClaw leest ze niet
terug tijdens het vastleggen, samenvatten of weergeven van lijsten. De standaardstatusmap is
`~/.openclaw`; overschrijf deze met `OPENCLAW_STATE_DIR`. De datummap wordt afgeleid
van de starttijd van de sessie; de sessiemap is een bestandssysteemveilige slug
die van de sessie-id is afgeleid.

## Opdrachten

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| Opdracht                      | Beschrijving                                           |
| ----------------------------- | ------------------------------------------------------ |
| `list`                        | Opgeslagen sessies weergeven.                          |
| `show <session>`              | `summary.md` afdrukken en materialiseren.              |
| `path <session>`              | Het pad `summary.md` materialiseren en afdrukken.      |
| `path <session> --dir`        | Alle artefacten materialiseren en hun map afdrukken.   |
| `path <session> --metadata`   | `metadata.json` materialiseren en afdrukken.           |
| `path <session> --transcript` | `transcript.jsonl` materialiseren en afdrukken.        |
| `--json`                      | Machineleesbare uitvoer afdrukken (elke subopdracht).  |

`<session>` accepteert een losse sessie-id of een datumgekwalificeerde selector
(`YYYY-MM-DD/<session>`). Gebruik de gekwalificeerde vorm wanneer dezelfde sessie-id
op meer dan één dag voorkomt, bijvoorbeeld `openclaw transcripts show
2026-05-22/standup`. Standaardsessie-id's bevatten een tijdstempel en een willekeurig
achtervoegsel; geef een sessie alleen een vaste id wanneer die id binnen de dag uniek is.

## Uitvoer

`list` drukt per sessie één door tabs gescheiden regel af: selector, starttijd, titel,
pad naar samenvatting.

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  Wekelijkse stand-up  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

De selector is de veiligste waarde om terug te geven aan `show` of `path`.

`list --json` retourneert objecten met `sessionId`, `selector`, `date`, `title`,
`startedAt`, `stoppedAt`, `source`, `path`, `summaryPath`, `hasSummary`.
Opgeslagen bron-URL's van vergaderingen bevatten alleen de oorsprong en het pad; querytekenreeksen,
fragmenten en ingesloten aanmeldgegevens worden vóór opslag verwijderd.

`show --json` retourneert de opgeslagen sessiemetadata, selector, sessiemap,
het pad naar de samenvatting en de Markdown-tekst van de samenvatting.

`path --json` retourneert het geselecteerde pad en of dat artefact kon worden
gematerialiseerd. Metadata- en transcriptexports bestaan altijd voor een opgeslagen
sessie; een samenvattingspad rapporteert `exists: false` totdat de sessie een samenvatting heeft.

## Meerdere sessies per dag

Sessies worden eerst op datum en vervolgens op sessie-id gegroepeerd. Tien vergaderingen op één dag worden
tien mappen op hetzelfde niveau:

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

Gebruik standaard gegenereerde id's voor automatisering. Gebruik alleen een vaste id zoals `standup`
wanneer deze niet op dezelfde datum wordt herhaald.

## Ontbrekende samenvattingen

Live sessies slaan `summary.md` op en materialiseren dit wanneer de sessie stopt;
geïmporteerde transcripten doen dit onmiddellijk na de import. Een sessie kan zonder
samenvatting in `list` verschijnen terwijl de vastlegging nog actief is, als een provider
tijdens het stoppen is mislukt of als metadata werden opgeslagen voordat er uitingen binnenkwamen.

Gebruik `path <session> --transcript` om het onbewerkte alleen-toevoegen-transcript te inspecteren,
of voer de actie `summarize` van de tool `transcripts` uit om de Markdown-
samenvatting opnieuw te genereren.

## De verouderde bestandsopslag upgraden

OpenClaw-releases van vóór de SQLite-opslag schreven de canonieke runtimestatus
rechtstreeks onder `$OPENCLAW_STATE_DIR/transcripts/`. Voer uit:

```bash
openclaw doctor --fix
```

Doctor importeert de volledige verouderde boomstructuur in SQLite, verifieert aantallen rijen en
volgorde, registreert migratiebewijzen en verplaatst de geverifieerde bronstructuur naar een
van een tijdstempel voorzien `transcripts.migrated-*`-archief. Runtimeopdrachten vallen niet terug
op de verouderde bestanden. Bewaar het archief totdat je de geïmporteerde
sessies en alle exports waarvan je afhankelijk bent, hebt geverifieerd.

## Configuratie

Het vastleggen van vergadertranscripten is standaard ingeschakeld. Om dit globaal uit te schakelen:

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled` (standaard `true`): automatische vergadernotities, de transcripttool
  en geconfigureerde bronnen voor automatisch starten inschakelen. Stel dit in op `false` wanneer vergadernotities
  niet op de host mogen worden opgeslagen. Een expliciet aangevraagde vergadermodus
  `transcribe` behoudt zijn bestaande begrensde live-ondertiteling, maar schrijft geen
  duurzame rijen zolang deze instelling onwaar is.
  Configureer bronnen voor automatisch starten met `transcripts.autoStart`. Elke vermelding is
  ingeschakeld doordat deze aanwezig is; laat een vermelding weg om die bron uit te schakelen. `discord-voice`
  is de gebundelde bron die automatisch starten ondersteunt en vereist `guildId` en
  `channelId`:

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

De id's van de vergaderproviders zijn `google-meet`, `teams` en `zoom`. Hun aliassen
zijn respectievelijk `googlemeet`/`meet`, `teams-meetings`/`microsoft-teams`/`msteams` en
`zoom-meetings`. Vergaderproviders koppelen zich aan een reeds actieve
vergaderbotsessie; normale deelnames aan vergaderingen hebben geen `autoStart`-vermelding nodig.
