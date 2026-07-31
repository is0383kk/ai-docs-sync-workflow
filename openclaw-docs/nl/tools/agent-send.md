---
read_when:
    - Je wilt agentruns activeren vanuit scripts of via de opdrachtregel
    - Je moet antwoorden van de agent programmatisch naar een chatkanaal sturen
summary: Voer agentbeurten uit via de CLI en stuur antwoorden optioneel naar kanalen
title: Agent verzenden
x-i18n:
    generated_at: "2026-07-27T05:23:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3da0feea102725ebb5555e0dd375ed6f3a0396d8ffd0ab916ced303201eabc
    source_path: tools/agent-send.md
    workflow: 16
---

`openclaw agent` voert één agentbeurt uit vanaf de opdrachtregel zonder een
inkomend chatbericht. Gebruik dit voor gescripte workflows, tests en
programmatische levering. Volledige naslag voor vlaggen en gedrag:
[Agent-CLI-naslag](/nl/cli/agent).

## Snel aan de slag

<Steps>
  <Step title="Een eenvoudige agentbeurt uitvoeren">
    ```bash
    openclaw agent --agent main --message "Wat voor weer is het vandaag?"
    ```

    Verstuurt het bericht via de Gateway en drukt het antwoord af.

  </Step>

  <Step title="Een prompt met meerdere regels vanuit een bestand versturen">
    ```bash
    openclaw agent --agent ops --message-file ./task.md
    ```

    Leest een geldig UTF-8-bestand als de berichttekst voor de agent.

  </Step>

  <Step title="Een specifieke agent of sessie kiezen">
    ```bash
    # Een specifieke agent kiezen
    openclaw agent --agent ops --message "Logboeken samenvatten"

    # Een telefoonnummer kiezen (leidt de sessiesleutel af)
    openclaw agent --to +15555550123 --message "Statusupdate"

    # Een bestaande sessie hergebruiken
    openclaw agent --session-id abc123 --message "Doorgaan met de taak"

    # Een exacte sessiesleutel kiezen
    openclaw agent --session-key agent:ops:incident-42 --message "Status samenvatten"
    ```

  </Step>

  <Step title="Het antwoord aan een kanaal leveren">
    ```bash
    # Leveren aan WhatsApp (standaardkanaal)
    openclaw agent --to +15555550123 --message "Rapport gereed" --deliver

    # Leveren aan Slack
    openclaw agent --agent ops --message "Rapport genereren" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## Vlaggen

| Vlag                        | Beschrijving                                                          |
| --------------------------- | -------------------------------------------------------------------- |
| `--message <text>`          | Inlinebericht om te versturen                                               |
| `--message-file <path>`     | Het bericht uit een geldig UTF-8-bestand lezen (max. 4 MiB)                 |
| `--to <dest>`               | Sessiesleutel afleiden van een doel (telefoon, chat-id)                    |
| `--session-key <key>`       | Een expliciete sessiesleutel gebruiken                                          |
| `--agent <id>`              | Een geconfigureerde agent kiezen (gebruikt de `main`-sessie ervan)                  |
| `--session-id <id>`         | Een bestaande sessie hergebruiken op basis van id                                      |
| `--model <id>`              | Modeloverschrijving voor deze uitvoering (`provider/model` of model-id)           |
| `--local`                   | Lokale ingebedde runtime afdwingen (Gateway overslaan)                          |
| `--deliver`                 | Het antwoord naar een chatkanaal sturen                                     |
| `--channel <name>`          | Leveringskanaal; met `--agent` + `--to` ook van toepassing op het DM-bereik     |
| `--reply-to <target>`       | Overschrijving van leveringsdoel                                             |
| `--reply-channel <name>`    | Overschrijving van leveringskanaal                                            |
| `--reply-account <id>`      | Overschrijving van account-id voor levering                                         |
| `--thinking <level>`        | Denkniveau instellen voor het geselecteerde modelprofiel                    |
| `--verbose <on\|full\|off>` | Uitgebreidheidsniveau voor de sessie opslaan (`full` registreert ook tooluitvoer) |
| `--timeout <seconds>`       | Time-out van de agent overschrijven (standaard 600, of configuratiewaarde)                |
| `--json`                    | Gestructureerde JSON uitvoeren                                               |

## Gedrag

- Standaard gaat de CLI **via de Gateway**. Voeg `--local` toe om de
  ingebedde runtime op de huidige machine af te dwingen.
- Geef precies één van `--message` of `--message-file` door. Bestandsberichten behouden
  inhoud met meerdere regels nadat een optionele UTF-8-BOM is verwijderd. Bestanden groter dan
  4 MiB worden vóór verzending geweigerd.
- Na tijdelijke nieuwe verbindingspogingen tijdens de handshake zorgt een Gateway-time-out of gesloten verbinding
  ervoor dat de opdracht mislukt met een aanwijzing op stderr; de CLI voert de beurt nooit stilzwijgend opnieuw
  ingebed uit. De Gateway kan een geaccepteerde beurt nog steeds voltooien, dus controleer de status van de Gateway
  en sessie voordat je het opnieuw probeert of opnieuw uitvoert met `--local`.
- Sessieselectie: `--to` leidt de sessiesleutel af (doelen voor groepen/kanalen
  behouden isolatie; directe chats worden samengevoegd tot `main`). Met `--agent`,
  `--channel` en `--to` samen volgt de routering de canonieke
  ontvanger en `session.dmScope` van het kanaal. Stabiele identiteiten die alleen voor uitgaand verkeer worden gebruikt, gebruiken een
  sessie die eigendom is van de provider en is geïsoleerd van de hoofdsessie van de agent.
- `--session-key` selecteert een expliciete sleutel. Sleutels met een agentvoorvoegsel moeten
  `agent:<agent-id>:<session-key>` gebruiken, en `--agent` moet overeenkomen met die agent-id wanneer
  beide worden opgegeven. Kale sleutels die geen sentinel zijn, worden beperkt tot `--agent` wanneer
  die wordt opgegeven; `--agent ops --session-key incident-42` wordt bijvoorbeeld gerouteerd naar
  `agent:ops:incident-42`. Zonder `--agent` worden kale sleutels die geen sentinel zijn, beperkt
  tot de geconfigureerde standaardagent. Letterlijke `global` en `unknown` blijven
  alleen onbeperkt wanneer geen `--agent` wordt opgegeven.
- `--reply-channel` en `--reply-account` zijn alleen van invloed op de levering.
- Vlaggen voor denken en uitgebreidheid worden in de sessieopslag bewaard.
- Uitvoer: standaard platte tekst, of `--json` voor een gestructureerde payload + metagegevens.
- Met `--json --deliver` bevat de JSON de leveringsstatus voor verzonden,
  onderdrukte, gedeeltelijke en mislukte verzendingen. Zie
  [JSON-leveringsstatus](/nl/cli/agent#json-delivery-status).

## Voorbeelden

```bash
# Eenvoudige beurt met JSON-uitvoer
openclaw agent --to +15555550123 --message "Logboeken traceren" --verbose on --json

# Beurt met een modeloverschrijving
openclaw agent --agent ops --model openai/gpt-5.4 --message "Logboeken samenvatten"

# Beurt met denkniveau
openclaw agent --session-id 1234 --message "Postvak IN samenvatten" --thinking medium

# Prompt met meerdere regels vanuit een bestand
openclaw agent --agent ops --message-file ./task.md

# Exacte sessiesleutel
openclaw agent --session-key agent:ops:incident-42 --message "Status samenvatten"

# Verouderde sleutel beperkt tot een agent
openclaw agent --agent ops --session-key incident-42 --message "Status samenvatten"

# Leveren aan een ander kanaal dan de sessie
openclaw agent --agent ops --message "Waarschuwing" --deliver --reply-channel telegram --reply-to "@admin"
```

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Agent-CLI-naslag" href="/nl/cli/agent" icon="terminal">
    Volledige naslag voor vlaggen en opties van `openclaw agent`.
  </Card>
  <Card title="Subagents" href="/nl/tools/subagents" icon="users">
    Subagents op de achtergrond starten.
  </Card>
  <Card title="Sessies" href="/nl/concepts/session" icon="comments">
    Hoe sessiesleutels werken en hoe `--to`, `--agent` en `--session-id` ze omzetten.
  </Card>
  <Card title="Slash-opdrachten" href="/nl/tools/slash-commands" icon="slash">
    Systeemeigen opdrachtencatalogus die binnen agentsessies wordt gebruikt.
  </Card>
</CardGroup>
