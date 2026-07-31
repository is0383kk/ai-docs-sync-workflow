---
read_when:
    - Je wilt één agentbeurt vanuit scripts uitvoeren (en eventueel het antwoord afleveren)
summary: CLI-referentie voor `openclaw agent` (verstuur één agentbeurt via de Gateway)
title: Agent
x-i18n:
    generated_at: "2026-07-27T06:07:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1a4c139a3b235d6a56ba63063737b80f93448c2dbb7a92c6d0756fb19a9f95e4
    source_path: cli/agent.md
    workflow: 16
---

# `openclaw agent`

Voer één agentbeurt uit via de Gateway. De expliciete vlag `--local` is het enige ingebedde uitvoeringspad.

Geef ten minste één sessieselector door: `--to`, `--session-key`, `--session-id` of `--agent`.

Gerelateerd: [Tool voor verzenden door agent](/nl/tools/agent-send)

## Opties

- `-m, --message <text>`: berichttekst
- `--message-file <path>`: lees de berichttekst uit een UTF-8-bestand
- `-t, --to <dest>`: ontvanger die wordt gebruikt om de sessiesleutel af te leiden
- `--session-key <key>`: expliciete sessiesleutel voor routering
- `--session-id <id>`: expliciete sessie-id
- `--agent <id>`: agent-id; overschrijft routeringskoppelingen
- `--model <id>`: modeloverschrijving voor deze uitvoering (`provider/model` of model-id)
- `--thinking <level>`: denkniveau van de agent (`off`, `minimal`, `low`, `medium`, `high`, plus door de provider ondersteunde aangepaste niveaus zoals `xhigh`, `adaptive` of `max`)
- `--verbose <on|off>`: sla het uitvoerigheidsniveau op voor de sessie
- `--channel <channel>`: afleveringskanaal; laat weg om het hoofdkanaal van de sessie te gebruiken
- `--reply-to <target>`: overschrijving van het afleveringsdoel
- `--reply-channel <channel>`: overschrijving van het afleveringskanaal
- `--reply-account <id>`: overschrijving van het afleveringsaccount
- `--local`: voer de ingebedde agent rechtstreeks uit (nadat het Plugin-register vooraf is geladen)
- `--deliver`: stuur het antwoord terug naar het geselecteerde kanaal/doel
- `--timeout <seconds>`: overschrijf de deadline voor de agentbeurt van deze opdracht (standaard 600, of `agents.defaults.timeoutSeconds`); `0` schakelt de algehele deadline uit. De terugvalwaarde van 600 seconden hoort bij deze CLI-opdracht, niet bij gewone Gateway-beurten, waarvan de standaardwaarde 48 uur is.
- `--json`: voer JSON uit

## Voorbeelden

```bash
openclaw agent --to +15555550123 --message "status update" --deliver
openclaw agent --agent ops --message "Summarize logs"
openclaw agent --agent ops --message-file ./task.md
openclaw agent --agent ops --model openai/gpt-5.4 --message "Summarize logs"
openclaw agent --session-key agent:ops:incident-42 --message "Summarize status"
openclaw agent --agent ops --session-key incident-42 --message "Summarize status"
openclaw agent --session-id 1234 --message "Summarize inbox" --thinking medium
openclaw agent --to +15555550123 --message "Trace logs" --verbose on --json
openclaw agent --agent ops --message "Generate report" --deliver --reply-channel slack --reply-to "#reports"
openclaw agent --agent ops --message "Run locally" --local
```

## Opmerkingen

- Geef precies één van `--message` of `--message-file` door. `--message-file` verwijdert een voorafgaande UTF-8-BOM en behoudt inhoud met meerdere regels; bestanden die geen geldige UTF-8 bevatten, worden geweigerd. Bestanden groter dan 4 MiB worden vóór verzending geweigerd.
- Slash-opdrachten (bijvoorbeeld `/compact`) kunnen niet via `--message` worden uitgevoerd. De CLI weigert ze en verwijst je in plaats daarvan naar de volwaardige opdracht (`openclaw sessions compact <key>` voor Compaction).
- Uitvoeringen met `--local` zijn eenmalig: gebundelde MCP-loopbackresources en warme Claude-stdio-sessies die voor de uitvoering zijn geopend, worden na het antwoord beëindigd, zodat gescripte aanroepen geen lokale onderliggende processen actief laten. Uitvoeringen via de Gateway behouden daarentegen MCP-loopbackresources die eigendom zijn van de Gateway onder het actieve Gateway-proces.
- Zelfstandige ingebedde uitvoering met `--local` weigert een bestaande hoofdsessie opnieuw te gebruiken zolang herstel na een herstart in behandeling is. Voer de beurt via een gezonde Gateway uit of stel deze daar opnieuw in met `/new` of `/reset`; een onafhankelijk ingebed proces kan de eigenaar van dat herstel niet veilig coördineren met de Gateway-scanner.
- Wanneer `--agent`, `--channel` en `--to` samen worden gebruikt, volgt de sessieroutering de canonieke ontvanger van het kanaal en `session.dmScope`. Kanalen met een stabiele, uitsluitend uitgaande ontvangersidentiteit gebruiken een sessie van de provider die is geïsoleerd van de hoofdsessie van de agent. `--reply-channel` en `--reply-account` beïnvloeden alleen de aflevering.
- `--session-key` selecteert een expliciete sessiesleutel. Sleutels met een agentvoorvoegsel moeten `agent:<agent-id>:<session-key>` gebruiken en `--agent` moet overeenkomen met de agent-id van de sleutel wanneer beide zijn opgegeven. Kale sleutels die geen sentinel zijn, worden beperkt tot `--agent` wanneer die is opgegeven, of anders tot de geconfigureerde standaardagent; `--agent ops --session-key incident-42` wordt bijvoorbeeld gerouteerd naar `agent:ops:incident-42`. De letterlijke sleutels `global` en `unknown` blijven alleen onbeperkt wanneer geen `--agent` is opgegeven.
- `--json` reserveert stdout voor het JSON-antwoord; diagnostiek van de Gateway, Plugins en `--local` gaat naar stderr, zodat scripts stdout rechtstreeks kunnen parseren.
- Nadat tijdelijke nieuwe handshakepogingen zijn uitgeput, zorgt een Gateway-time-out of gesloten verbinding ervoor dat de opdracht mislukt; de CLI voert de beurt nooit stilzwijgend opnieuw ingebed uit. Verlies van transport is dubbelzinnig — de Gateway kan de beurt hebben geaccepteerd en nog steeds voltooien — daarom adviseert de stderr-melding om `openclaw gateway status` en het sessietranscript te controleren voordat je het opnieuw probeert of opnieuw uitvoert met `--local`, om te voorkomen dat de beurt tweemaal wordt uitgevoerd.
- `SIGTERM`/`SIGINT` onderbreken een wachtend verzoek via de Gateway; als de Gateway de uitvoering al heeft geaccepteerd, stuurt de CLI vóór afsluiting ook `chat.abort` voor die uitvoerings-id. Uitvoeringen met `--local` ontvangen hetzelfde signaal, maar sturen geen `chat.abort`. Een onderliggend proces van het startprogramma dat door de eerste doorgestuurde `SIGINT` of `SIGTERM` wordt beëindigd, sluit af met respectievelijk status 130 of 143. Als de interne sleutel voor het dedupliceren van uitvoeringen al een actieve uitvoering voor deze sessie heeft, meldt het antwoord `status: "in_flight"` en drukt de niet-JSON-CLI een diagnostische melding af naar stderr in plaats van een leeg antwoord. Behoud voor externe cron-/systemd-wrappers een noodstop met geforceerde beëindiging, zoals `timeout -k 60 600 openclaw agent ...`, zodat het toezichtsproces het proces kan opruimen als het afsluiten niet kan worden voltooid.
- Wanneer deze opdracht het opnieuw genereren van `models.json` activeert, worden door SecretRef beheerde providerreferenties opgeslagen als niet-geheime markeringen (bijvoorbeeld namen van omgevingsvariabelen, `secretref-env:ENV_VAR_NAME` of `secretref-managed`), nooit als platte tekst met opgeloste geheimen. Markeringen worden geschreven vanuit de actieve momentopname van de bronconfiguratie, niet vanuit opgeloste geheime runtimewaarden.

## JSON-afleveringsstatus

Met `--json --deliver` bevat het JSON-antwoord van de CLI `deliveryStatus` op het hoogste niveau, zodat scripts onderscheid kunnen maken tussen verzonden, onderdrukte, gedeeltelijke en mislukte verzendingen:

```json
{
  "payloads": [{ "text": "Report ready", "mediaUrl": null }],
  "meta": { "durationMs": 1200 },
  "deliveryStatus": {
    "requested": true,
    "attempted": true,
    "status": "sent",
    "succeeded": true,
    "resultCount": 1
  }
}
```

CLI-antwoorden via de Gateway behouden ook de onbewerkte resultaatstructuur van de Gateway in `result.deliveryStatus`.

`deliveryStatus.status` is een van:

| Status           | Betekenis                                                                                                                                  |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `sent`           | Aflevering voltooid.                                                                                                                       |
| `suppressed`     | Aflevering is opzettelijk niet verzonden (bijvoorbeeld omdat een hook voor berichtverzending deze annuleerde of er geen zichtbaar resultaat was). Definitief, niet opnieuw proberen. |
| `partial_failed` | Ten minste één payload is verzonden voordat een latere payload mislukte.                                                                   |
| `failed`         | Geen duurzame verzending voltooid, of de voorafgaande afleveringscontrole is mislukt.                                                      |

Algemene velden:

- `requested`: altijd `true` wanneer het object aanwezig is.
- `attempted`: `true` zodra het duurzame verzendpad is uitgevoerd; `false` bij fouten tijdens de voorafgaande controle of wanneer er geen zichtbare payloads zijn.
- `succeeded`: `true`, `false` of `"partial"`; `"partial"` hoort bij `status: "partial_failed"`.
- `reason`: reden in kleine letters en snake_case uit duurzame aflevering of voorafgaande validatie. Bekende waarden zijn onder meer `cancelled_by_message_sending_hook`, `no_visible_payload`, `no_visible_result`, `channel_resolved_to_internal`, `unknown_channel`, `invalid_delivery_target` en `no_delivery_target`; mislukte duurzame verzendingen kunnen ook de mislukte fase melden. Behandel onbekende waarden als ondoorzichtig, omdat de verzameling kan worden uitgebreid.
- `resultCount`: aantal verzendresultaten van het kanaal, indien beschikbaar.
- `sentBeforeError`: `true` wanneer bij een gedeeltelijke mislukking ten minste één payload is verzonden voordat een fout optrad.
- `error`: `true` voor mislukte of gedeeltelijk mislukte verzendingen.
- `errorMessage`: alleen aanwezig wanneer een onderliggende foutmelding voor de aflevering is vastgelegd. Fouten tijdens de voorafgaande controle bevatten `error`/`reason`, maar geen `errorMessage`.
- `payloadOutcomes`: optionele resultaten per payload met `index`, `status`, `reason`, `resultCount`, `error`, `stage`, `sentBeforeError` of hookmetadata, indien beschikbaar.

## Gerelateerd

- [CLI-referentie](/nl/cli)
- [Agent-runtime](/nl/concepts/agent)
