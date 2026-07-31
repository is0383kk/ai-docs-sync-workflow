---
read_when:
    - Je wilt snel de status van de actieve Gateway controleren
summary: CLI-referentie voor `openclaw health` (momentopname van Gateway-status via RPC)
title: Status
x-i18n:
    generated_at: "2026-07-27T05:40:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51cc0e3dd61af3e6fa460dd646bfa1c3e5bd1a52da860eac26c12101151d081d
    source_path: cli/health.md
    workflow: 16
---

# `openclaw health`

Haal via WebSocket-RPC een statusmomentopname op van de actieve Gateway (geen directe kanaalsockets vanuit de CLI).

## Opties

| Vlag             | Standaard | Beschrijving                                                                       |
| ---------------- | ------- | --------------------------------------------------------------------------------- |
| `--json`         | `false` | Druk machineleesbare JSON af in plaats van tekst.                                      |
| `--timeout <ms>` | `10000` | Verbindingstime-out in milliseconden.                                               |
| `--verbose`      | `false` | Dwingt een livecontrole af en breidt de uitvoer uit met alle geconfigureerde accounts en agents. |
| `--debug`        | `false` | Alias voor `--verbose`.                                                            |

Voorbeelden:

```bash
openclaw health
openclaw health --json
openclaw health --timeout 2500
openclaw health --verbose
openclaw health --debug
```

## Gedrag

- Zonder `--verbose` kan de Gateway een gecachte momentopname retourneren (maximaal 60 seconden actueel en ongewijzigd ten opzichte van de live runtimestatus van kanalen) en deze op de achtergrond vernieuwen voor de volgende aanroeper.
- `--verbose` dwingt een livecontrole af (accountcontroles per kanaal), toont verbindingsgegevens van de Gateway en breidt de voor mensen leesbare uitvoer uit met alle geconfigureerde accounts en agents in plaats van alleen de standaardagent.
- `--json` retourneert altijd de volledige momentopname: kanalen, controles per account, laadstatus van plugins, quarantainestatus van de contextengine, cachestatus van modelprijzen, status van de eventloop, dead letters uit de bezorgingswachtrij en sessieopslag per agent.
- Wanneer uitgaande bezorgingen of inkomende kanaalgebeurtenissen als dead letter worden opgeslagen, vermeldt de tekstuitvoer hun aantallen en de ouderdom van de oudste fout. Inkomende aantallen worden gegroepeerd per kanaalaccount; inspecteer of herstel afzonderlijke gebeurtenissen met [`openclaw channels dead-letters`](/nl/cli/channels#inbound-dead-letters).

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [`openclaw status`](/nl/cli/status) — lokale diagnose en kanaalcontroles zonder een volledige statusmomentopname
- [Gateway-status](/nl/gateway/health)
