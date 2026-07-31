---
read_when:
    - Een bugrapport of ondersteuningsverzoek voorbereiden
    - Gateway-crashes, herstarts, geheugendruk of te grote payloads debuggen
    - Controleren welke diagnostische gegevens worden vastgelegd of geredigeerd
summary: Maak deelbare diagnostiekbundels voor de Gateway voor bugrapporten
title: Diagnostische gegevens exporteren
x-i18n:
    generated_at: "2026-07-27T05:46:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 97a805fed8d51de2e63e5c6a12ce03e91701d69654882cca7795c9f3553b1c55
    source_path: gateway/diagnostics.md
    workflow: 16
---

OpenClaw kan een lokale diagnostische `.zip` voor bugrapporten maken: opgeschoonde Gateway-
status, statuscontroles, logboeken, configuratiestructuur en recente stabiliteitsgebeurtenissen zonder payload.

Behandel diagnostische bundels als geheimen totdat ze zijn gecontroleerd. Payloads en aanmeldgegevens
worden standaard geredigeerd, maar de bundel bevat nog steeds een samenvatting van lokale Gateway-logboeken en
runtimestatus op hostniveau.

## Snel aan de slag

```bash
openclaw gateway diagnostics export
```

Toont het pad van het geschreven zipbestand. Kies een uitvoerpad:

```bash
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
```

Voor automatisering:

```bash
openclaw gateway diagnostics export --json
```

## Chatopdracht

Eigenaren kunnen `/diagnostics [note]` in elk gesprek uitvoeren om een lokale
Gateway-export aan te vragen als één kopieerbaar ondersteuningsrapport:

1. Verstuur `/diagnostics`, eventueel met een korte opmerking (`/diagnostics bad tool choice`).
2. OpenClaw stuurt een inleiding en vraagt om één expliciete uitvoeringsgoedkeuring, waarmee
   `openclaw gateway diagnostics export --json` wordt uitgevoerd. Keur diagnostiek niet goed via
   een regel die alles toestaat.
3. Na goedkeuring antwoordt OpenClaw met het lokale bundelpad, een samenvatting van het manifest,
   privacyopmerkingen en relevante sessie-id's.

In groepschats kan een eigenaar nog steeds `/diagnostics` uitvoeren, maar OpenClaw stuurt het
exportresultaat, de goedkeuringsverzoeken en het overzicht van Codex-sessies en -threads
privé naar de eigenaar. De groep ziet alleen een korte melding dat de diagnostiek
privé is verzonden. Als er geen privéroute naar de eigenaar bestaat, wordt de opdracht veilig geweigerd en wordt
de eigenaar gevraagd deze vanuit een privébericht uit te voeren.

Wanneer de actieve sessie de native OpenAI Codex-harness gebruikt, geldt dezelfde uitvoeringsgoedkeuring
ook voor het uploaden van OpenAI-feedback over de Codex-threads die OpenClaw
kent. Die upload staat los van het lokale Gateway-zipbestand en vindt alleen
plaats voor Codex-harnesssessies. In het goedkeuringsverzoek staat dat goedkeuring
ook Codex-feedback verstuurt, zonder Codex-sessie- of thread-id's te vermelden. Na
goedkeuring vermeldt het antwoord kanalen, OpenClaw-sessie-id's, Codex-thread-id's en
lokale hervattingsopdrachten voor de threads die naar OpenAI zijn verzonden. Als de goedkeuring wordt
geweigerd of genegeerd, worden de export, de upload van Codex-feedback en de
lijst met Codex-id's overgeslagen.

Dat maakt de foutopsporingscyclus voor Codex kort: merk ongewenst gedrag in een kanaal op,
voer `/diagnostics` uit, keur één keer goed, deel het rapport en voer vervolgens lokaal de weergegeven
opdracht `codex resume <thread-id>` uit als je de thread
zelf wilt inspecteren. Zie [Codex-harness](/nl/plugins/codex-harness#inspect-codex-threads-locally).

## Wat de export bevat

- `summary.md`: voor mensen leesbaar overzicht voor ondersteuning.
- `diagnostics.json`: machineleesbare samenvatting van configuratie, logboeken, status, statuscontroles
  en stabiliteitsgegevens.
- `manifest.json`: exportmetadata en bestandenlijst.
- Opgeschoonde configuratiestructuur en niet-geheime configuratiegegevens.
- Opgeschoonde logboeksamenvattingen en recente geredigeerde logboekregels.
- Gateway-status- en statuscontrolemomentopnamen op basis van beschikbare gegevens.
- `stability/latest.json`: nieuwste opgeslagen stabiliteitsbundel, indien beschikbaar.

De export blijft nuttig wanneer de Gateway niet naar behoren werkt: als status- of statuscontroleverzoeken
mislukken, worden lokale logboeken, de configuratiestructuur en de nieuwste stabiliteitsbundel
nog steeds verzameld indien beschikbaar.

## Privacymodel

Behouden: namen van subsystemen, plugin-id's, provider-id's, kanaal-id's, geconfigureerde
modi, statuscodes, tijdsduren, aantallen bytes, wachtrijstatus, geheugengegevens,
opgeschoonde logboekmetadata, geredigeerde operationele berichten, configuratiestructuur en
niet-geheime functie-instellingen.

Weggelaten of geredigeerd: chattekst, prompts, instructies, webhook-bodies, tooluitvoer,
aanmeldgegevens, API-sleutels, tokens, cookies, geheime waarden, onbewerkte
request-/response-bodies, account-id's, bericht-id's, onbewerkte sessie-id's,
hostnamen en lokale gebruikersnamen.

Wanneer een logboekbericht lijkt op tekst uit een gebruikers-, chat-, prompt- of toolpayload,
vermeldt de export alleen dat een bericht is weggelaten, plus het aantal bytes.

## Stabiliteitsrecorder

De Gateway registreert standaard een begrensde stabiliteitsstroom zonder payload wanneer
diagnostiek is ingeschakeld. Deze legt operationele feiten vast, geen inhoud.

Dezelfde Heartbeat bemonstert ook de activiteit wanneer de eventloop of CPU
verzadigd lijkt, en genereert `diagnostic.liveness.warning`-gebeurtenissen met eventloopvertraging,
eventloopgebruik, CPU-kernverhouding, aantallen actieve/wachtende/in de wachtrij geplaatste sessies,
de huidige opstart-/runtimefase (indien bekend), recente fasespannen en
begrensde werklabels. Deze worden alleen Gateway-logboekregels op niveau `warn` wanneer
werk wacht of in de wachtrij staat, of wanneer actief werk samenvalt met aanhoudende eventloopvertraging;
anders worden ze vastgelegd op `debug`. Activiteitsmetingen bij inactiviteit worden nog steeds geregistreerd
als diagnostische gebeurtenissen, maar leiden op zichzelf nooit tot een waarschuwing.

Opstartfasen genereren `diagnostic.phase.completed`-gebeurtenissen met kloktijd- en
CPU-timing. Diagnostiek voor vastgelopen ingebedde uitvoeringen markeert `terminalProgressStale=true`
wanneer de laatste bridge-voortgang terminaal leek (bijvoorbeeld een onbewerkt response-item
of een gebeurtenis voor voltooiing van een response), maar de Gateway de
ingebedde uitvoering nog steeds als actief beschouwt.

Inspecteer de live recorder:

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --json
```

Inspecteer de nieuwste opgeslagen bundel na een fatale afsluiting, time-out bij afsluiten of
een mislukte opstart na herstart:

```bash
openclaw gateway stability --bundle latest
```

Maak een diagnostisch zipbestand van de nieuwste opgeslagen bundel:

```bash
openclaw gateway stability --bundle latest --export
```

Opgeslagen bundels bevinden zich onder `~/.openclaw/logs/stability/` wanneer er gebeurtenissen bestaan.

## Nuttige opties

```bash
openclaw gateway diagnostics export \
  --output openclaw-diagnostics.zip \
  --log-lines 5000 \
  --log-bytes 1000000
```

| Vlag                    | Standaard                                                                       | Beschrijving                                        |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------------------------------------- |
| `--output <path>`       | `$OPENCLAW_STATE_DIR/logs/support/openclaw-diagnostics-<timestamp>-<pid>.zip` | Schrijf naar een specifiek zipbestandspad (of een specifieke map).       |
| `--log-lines <count>`   | `5000`                                                                        | Maximaal aantal op te nemen opgeschoonde logboekregels.            |
| `--log-bytes <bytes>`   | `1000000`                                                                     | Maximaal aantal te inspecteren logboekbytes.                      |
| `--url <url>`           | -                                                                             | Gateway-WebSocket-URL voor status- en statuscontrolemomentopnamen. |
| `--token <token>`       | -                                                                             | Gateway-token voor status- en statuscontrolemomentopnamen.         |
| `--password <password>` | -                                                                             | Gateway-wachtwoord voor status- en statuscontrolemomentopnamen.      |
| `--timeout <ms>`        | `3000`                                                                        | Time-out voor status- en statuscontrolemomentopnamen.                    |
| `--no-stability-bundle` | uit                                                                           | Sla het zoeken naar een opgeslagen stabiliteitsbundel over.            |
| `--json`                | uit                                                                           | Toon machineleesbare exportmetadata.            |

## Diagnostiek uitschakelen

Diagnostiek is standaard ingeschakeld. De stabiliteitsrecorder en
verzameling van diagnostische gebeurtenissen uitschakelen:

```json5
{
  diagnostics: {
    enabled: false,
  },
}
```

Het uitschakelen van diagnostiek vermindert de details in bugrapporten; dit heeft geen invloed op de normale
Gateway-logboekregistratie.

Gebeurtenissen voor geheugendruk registreren RSS-, heap-, drempel- en groeigegevens
(`rss_threshold`, `heap_threshold`, `rss_growth`) zonder een
bestandssysteemscan uit te voeren of een momentopname van vóór een OOM te schrijven.

## Gerelateerd

- [Statuscontroles](/nl/gateway/health)
- [Gateway-CLI](/nl/cli/gateway#gateway-diagnostics-export)
- [Gateway-protocol](/nl/gateway/protocol#rpc-method-families)
- [Logboekregistratie](/nl/logging)
- [OpenTelemetry-export](/nl/gateway/opentelemetry) - afzonderlijke flow voor het streamen van diagnostiek naar een collector
