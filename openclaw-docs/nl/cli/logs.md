---
read_when:
    - Je moet Gateway-logboeken op afstand volgen (zonder SSH)
    - Je wilt JSON-logregels voor tooling
summary: CLI-referentie voor `openclaw logs` (Gateway-logboeken volgen via RPC)
title: Logboeken
x-i18n:
    generated_at: "2026-07-27T05:00:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

Volg Gateway-bestandslogboeken via RPC. Werkt in externe modus.

## Opties

- `--limit <n>`: maximaal aantal te retourneren logregels (standaard `200`)
- `--max-bytes <n>`: maximaal aantal bytes om uit het logbestand te lezen (standaard `250000`)
- `--follow`: volg de logstroom
- `--interval <ms>`: pollinginterval tijdens het volgen (standaard `1000`)
- `--json`: genereer door regels gescheiden JSON-gebeurtenissen
- `--plain`: uitvoer als platte tekst zonder opgemaakte vormgeving
- `--no-color`: schakel ANSI-kleuren uit
- `--local-time`: geef tijdstempels weer in je lokale tijdzone (standaard)
- `--utc`: geef tijdstempels weer in UTC

## Gedeelde RPC-opties voor de Gateway

- `--url <url>`: WebSocket-URL van de Gateway
- `--token <token>`: Gateway-token
- `--timeout <ms>`: time-out in ms (standaard `30000`)
- `--expect-final`: wacht op een definitief antwoord wanneer de Gateway-aanroep door een agent wordt afgehandeld

Door `--url` door te geven, worden automatisch toegepaste configuratiegegevens voor authenticatie overgeslagen; neem `--token` expliciet op als de doel-Gateway authenticatie vereist.

## Voorbeelden

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

Het geselecteerde hoofdprofiel komt overeen met het roterende bestand van de Gateway: het standaardprofiel gebruikt `openclaw-YYYY-MM-DD.log`, terwijl benoemde profielen
`openclaw-<profile>-YYYY-MM-DD.log` gebruiken (bijvoorbeeld
`openclaw-dev-YYYY-MM-DD.log`).

## Gedrag bij terugval en herstel

- Als de impliciete lokale loopback-Gateway om koppeling vraagt, tijdens het verbinden wordt gesloten of een time-out optreedt voordat `logs.tail` antwoordt, valt `openclaw logs` automatisch terug op het geconfigureerde Gateway-logbestand. Expliciete `--url`-doelen gebruiken deze terugval nooit.
- `--follow` valt na een RPC-fout van een impliciete lokale Gateway niet terug op dat geconfigureerde bestand — een verouderd bestand ernaast kan een live gevolgde logstroom verkeerd weergeven. Op Linux wordt in plaats daarvan, indien beschikbaar, het actieve Gateway-journaal van systemd voor de gebruiker op basis van PID gebruikt (de geselecteerde bron wordt weergegeven); anders blijft de live Gateway opnieuw worden geprobeerd.
- Tijdens `--follow` leiden tijdelijke verbrekingen (sluiting van WebSocket, time-out, wegvallende verbinding) tot automatische herverbinding met exponentiële vertraging: maximaal 8 pogingen, met maximaal 30s tussen pogingen. Bij elke nieuwe poging wordt een waarschuwing naar stderr geschreven en zodra een poll slaagt, wordt eenmaal een `[logs] gateway reconnected`-melding weergegeven. In de modus `--json` worden beide als `{"type":"notice"}`-records naar stderr geschreven. Niet-herstelbare fouten (mislukte authenticatie, onjuiste configuratie) leiden nog steeds tot onmiddellijke afsluiting.
- In de modus `--follow --json` worden overgangen tussen logbronnen als `{"type":"meta"}`-records gegenereerd. Houd cursors bij per `sourceKind`: een stroom kan van uitvoer uit een Gateway-bestand (`sourceKind: "file"`) overgaan naar de terugval op het lokale journaal (`sourceKind: "journal"`, `localFallback: true`, met `service.pid`/`service.unit`) en na herstel teruggaan naar uitvoer uit een Gateway-bestand. Ga niet uit van één stabiele bron of cursor voor de hele sessie en sta overlappende regels toe wanneer bij herstel de cursor van het Gateway-bestand opnieuw wordt afgespeeld.

## Gerelateerd

- [Overzicht van logboekregistratie](/nl/logging)
- [Gateway-CLI](/nl/cli/gateway)
- [CLI-referentie](/nl/cli)
- [Gateway-logboekregistratie](/nl/gateway/logging)
