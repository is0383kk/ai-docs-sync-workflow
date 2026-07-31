---
read_when:
    - Je wilt een systeemgebeurtenis in de wachtrij plaatsen zonder een cron-taak te maken
    - Je moet Heartbeats in- of uitschakelen
    - Je wilt de aanwezigheidsvermeldingen van het systeem inspecteren
summary: CLI-referentie voor `openclaw system` (systeemgebeurtenissen, Heartbeat, aanwezigheid)
title: Systeem
x-i18n:
    generated_at: "2026-07-27T05:47:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aaca206d8b463fd33f9e3cb21382bbf36469e9daa2706d8a9e2c7fab14b76e7a
    source_path: cli/system.md
    workflow: 16
---

# `openclaw system`

Helpers op systeemniveau voor de Gateway: systeemgebeurtenissen in de wachtrij plaatsen, heartbeats beheren en aanwezigheid bekijken.

Alle `system`-subcommando's gebruiken Gateway-RPC en accepteren de gedeelde clientvlaggen:

| Vlag              | Standaardwaarde                      | Beschrijving                                                                                                                                                                                            |
| ----------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `--url <url>`     | `gateway.remote.url` indien geconfigureerd | Gateway-WebSocket-URL.                                                                                                                                                                                 |
| `--token <token>` | geen                                 | Gateway-token (indien vereist).                                                                                                                                                                           |
| `--timeout <ms>`  | `30000`                              | RPC-time-out in milliseconden.                                                                                                                                                                           |
| `--expect-final`  | uit                                  | Wachten op definitief antwoord (agent).                                                                                                                                                                       |
| `--json`          | uit                                  | JSON uitvoeren. `heartbeat last/enable/disable` en `system presence` drukken altijd de onbewerkte JSON-payload van RPC af, ongeacht deze vlag; `system event` gebruikt deze om te schakelen tussen JSON en een gewone `ok`-regel. |

## Algemene commando's

```bash
openclaw system event --text "Controleren op dringende vervolgacties" --mode now
openclaw system event --text "Controleren op dringende vervolgacties" --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
openclaw system heartbeat enable
openclaw system heartbeat last
openclaw system presence
```

## `system event`

Plaats standaard een systeemgebeurtenis in de wachtrij van de **hoofd**sessie. De volgende heartbeat voegt deze als een `System:`-regel in de prompt in. Gebruik `--mode now` om de heartbeat onmiddellijk te activeren; `next-heartbeat` (standaard) wacht op de volgende geplande cyclus.

Geef `--session-key` door om een specifieke sessie als doel in te stellen, bijvoorbeeld om de voltooiing van een asynchrone taak terug te sturen naar het kanaal dat deze heeft gestart.

<Note>
**Uitzondering voor timing met `--session-key`:** wanneer `--session-key` wordt opgegeven, wordt `--mode next-heartbeat` teruggebracht tot een onmiddellijke, gerichte wekactie in plaats van te wachten op de volgende geplande cyclus. Gerichte wekacties gebruiken de heartbeat-intentie `immediate`, zodat ze de niet-aan-de-beurt-poort van de runner omzeilen, die anders een wekactie met de intentie `event` zou uitstellen (en feitelijk laten vervallen). Als je uitgestelde bezorging wilt, laat je `--session-key` weg, zodat de gebeurtenis in de hoofdsessie terechtkomt en met de volgende reguliere heartbeat wordt meegevoerd.
</Note>

Vlaggen:

- `--text <text>`: vereiste tekst van de systeemgebeurtenis.
- `--mode <mode>`: `now` of `next-heartbeat` (standaard).
- `--session-key <sessionKey>`: optioneel; richt je op een specifieke agentsessie in plaats van op de hoofdsessie van de agent. Sleutels die niet bij de gevonden agent horen, vallen terug op de hoofdsessie van de agent.

## `system heartbeat last|enable|disable`

- `last`: toon de laatste heartbeat-gebeurtenis.
- `enable`: schakel heartbeats weer in (gebruik dit als ze waren uitgeschakeld).
- `disable`: pauzeer heartbeats.

## `system presence`

Geef de huidige systeemaanwezigheidsitems weer die bij de Gateway bekend zijn (nodes, instanties en vergelijkbare statusregels).

## Opmerkingen

- Vereist een actieve Gateway die bereikbaar is via je huidige configuratie (lokaal of extern).
- Systeemgebeurtenissen zijn tijdelijk en worden niet bewaard bij herstarts.

## Gerelateerd

- [CLI-referentie](/nl/cli)
