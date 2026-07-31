---
read_when:
    - Sie möchten Agentenläufe über Skripte oder die Befehlszeile auslösen
    - Sie müssen Agentenantworten programmgesteuert an einen Chatkanal senden.
summary: Agent-Durchläufe über die CLI ausführen und Antworten optional an Kanäle übermitteln
title: Agentenversand
x-i18n:
    generated_at: "2026-07-26T18:05:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ad3da0feea102725ebb5555e0dd375ed6f3a0396d8ffd0ab916ced303201eabc
    source_path: tools/agent-send.md
    workflow: 16
---

`openclaw agent` führt einen einzelnen Agentendurchlauf über die Befehlszeile ohne eine
eingehende Chatnachricht aus. Verwenden Sie dies für skriptgesteuerte Workflows, Tests und
programmatische Zustellung. Vollständige Referenz zu Flags und Verhalten:
[Agent-CLI-Referenz](/de/cli/agent).

## Schnellstart

<Steps>
  <Step title="Einen einfachen Agentendurchlauf ausführen">
    ```bash
    openclaw agent --agent main --message "Wie ist das Wetter heute?"
    ```

    Sendet die Nachricht über das Gateway und gibt die Antwort aus.

  </Step>

  <Step title="Einen mehrzeiligen Prompt aus einer Datei senden">
    ```bash
    openclaw agent --agent ops --message-file ./task.md
    ```

    Liest eine gültige UTF-8-Datei als Nachrichtentext für den Agenten.

  </Step>

  <Step title="Einen bestimmten Agenten oder eine bestimmte Sitzung ansprechen">
    ```bash
    # Einen bestimmten Agenten ansprechen
    openclaw agent --agent ops --message "Protokolle zusammenfassen"

    # Eine Telefonnummer ansprechen (leitet den Sitzungsschlüssel ab)
    openclaw agent --to +15555550123 --message "Statusaktualisierung"

    # Eine vorhandene Sitzung wiederverwenden
    openclaw agent --session-id abc123 --message "Aufgabe fortsetzen"

    # Einen exakten Sitzungsschlüssel verwenden
    openclaw agent --session-key agent:ops:incident-42 --message "Status zusammenfassen"
    ```

  </Step>

  <Step title="Die Antwort an einen Kanal zustellen">
    ```bash
    # An WhatsApp zustellen (Standardkanal)
    openclaw agent --to +15555550123 --message "Bericht ist fertig" --deliver

    # An Slack zustellen
    openclaw agent --agent ops --message "Bericht erstellen" \
      --deliver --reply-channel slack --reply-to "#reports"
    ```

  </Step>
</Steps>

## Flags

| Flag                        | Beschreibung                                                          |
| --------------------------- | -------------------------------------------------------------------- |
| `--message <text>`          | Zu sendende Inline-Nachricht                                               |
| `--message-file <path>`     | Nachricht aus einer gültigen UTF-8-Datei lesen (max. 4 MiB)                 |
| `--to <dest>`               | Sitzungsschlüssel aus einem Ziel ableiten (Telefonnummer, Chat-ID)                    |
| `--session-key <key>`       | Einen expliziten Sitzungsschlüssel verwenden                                          |
| `--agent <id>`              | Einen konfigurierten Agenten ansprechen (verwendet dessen `main`-Sitzung)                  |
| `--session-id <id>`         | Eine vorhandene Sitzung anhand ihrer ID wiederverwenden                                      |
| `--model <id>`              | Modellüberschreibung für diesen Durchlauf (`provider/model` oder Modell-ID)           |
| `--local`                   | Lokale eingebettete Laufzeit erzwingen (Gateway überspringen)                          |
| `--deliver`                 | Antwort an einen Chatkanal senden                                     |
| `--channel <name>`          | Zustellungskanal; gilt mit `--agent` + `--to` auch für den DM-Geltungsbereich     |
| `--reply-to <target>`       | Zustellungsziel überschreiben                                             |
| `--reply-channel <name>`    | Zustellungskanal überschreiben                                            |
| `--reply-account <id>`      | Zustellungskonto-ID überschreiben                                         |
| `--thinking <level>`        | Denkstufe für das ausgewählte Modellprofil festlegen                    |
| `--verbose <on\|full\|off>` | Ausführlichkeitsstufe für die Sitzung speichern (`full` protokolliert auch die Werkzeugausgabe) |
| `--timeout <seconds>`       | Zeitüberschreitung des Agenten überschreiben (Standard: 600 oder Konfigurationswert)                |
| `--json`                    | Strukturiertes JSON ausgeben                                               |

## Verhalten

- Standardmäßig läuft die CLI **über das Gateway**. Fügen Sie `--local` hinzu, um die
  eingebettete Laufzeit auf dem aktuellen Rechner zu erzwingen.
- Übergeben Sie genau eines von `--message` oder `--message-file`. Dateinachrichten behalten
  mehrzeilige Inhalte bei, nachdem eine optionale UTF-8-BOM entfernt wurde. Dateien mit mehr als
  4 MiB werden vor der Weiterleitung abgelehnt.
- Nach vorübergehenden Wiederholungsversuchen beim Handshake führt eine Gateway-Zeitüberschreitung oder eine geschlossene Verbindung
  zum Fehlschlagen des Befehls mit einem Hinweis auf stderr; die CLI führt den Durchlauf niemals stillschweigend
  erneut eingebettet aus. Das Gateway kann einen angenommenen Durchlauf dennoch abschließen. Prüfen Sie daher den Gateway-
  und Sitzungsstatus, bevor Sie den Vorgang wiederholen oder mit `--local` erneut ausführen.
- Sitzungsauswahl: `--to` leitet den Sitzungsschlüssel ab (Gruppen-/Kanalziele
  behalten die Isolation bei; Direktchats werden zu `main` zusammengeführt). Wenn `--agent`,
  `--channel` und `--to` gemeinsam verwendet werden, folgt das Routing dem kanonischen
  Empfänger des Kanals und `session.dmScope`. Stabile Identitäten, die ausschließlich für ausgehende Nachrichten verwendet werden, nutzen eine
  vom Provider verwaltete Sitzung, die von der Hauptsitzung des Agenten isoliert ist.
- `--session-key` wählt einen expliziten Schlüssel aus. Mit einem Agentenpräfix versehene Schlüssel müssen
  `agent:<agent-id>:<session-key>` verwenden, und `--agent` muss mit dieser Agenten-ID übereinstimmen, wenn
  beide angegeben werden. Unqualifizierte Schlüssel, die keine Sentinel-Schlüssel sind, werden auf `--agent` beschränkt, wenn
  dies angegeben ist; beispielsweise wird `--agent ops --session-key incident-42` an
  `agent:ops:incident-42` weitergeleitet. Ohne `--agent` werden unqualifizierte Schlüssel, die keine Sentinel-Schlüssel sind, auf
  den konfigurierten Standardagenten beschränkt. Die literalen Werte `global` und `unknown` bleiben
  nur dann unbeschränkt, wenn kein `--agent` angegeben ist.
- `--reply-channel` und `--reply-account` wirken sich nur auf die Zustellung aus.
- Denk- und Ausführlichkeits-Flags werden im Sitzungsspeicher gespeichert.
- Ausgabe: standardmäßig Klartext oder `--json` für strukturierte Nutzdaten und Metadaten.
- Mit `--json --deliver` enthält das JSON den Zustellungsstatus für gesendete,
  unterdrückte, teilweise und fehlgeschlagene Sendungen. Siehe
  [JSON-Zustellungsstatus](/de/cli/agent#json-delivery-status).

## Beispiele

```bash
# Einfacher Durchlauf mit JSON-Ausgabe
openclaw agent --to +15555550123 --message "Protokolle nachverfolgen" --verbose on --json

# Durchlauf mit Modellüberschreibung
openclaw agent --agent ops --model openai/gpt-5.4 --message "Protokolle zusammenfassen"

# Durchlauf mit Denkstufe
openclaw agent --session-id 1234 --message "Posteingang zusammenfassen" --thinking medium

# Mehrzeiliger Prompt aus einer Datei
openclaw agent --agent ops --message-file ./task.md

# Exakter Sitzungsschlüssel
openclaw agent --session-key agent:ops:incident-42 --message "Status zusammenfassen"

# Auf einen Agenten beschränkter Legacy-Schlüssel
openclaw agent --agent ops --session-key incident-42 --message "Status zusammenfassen"

# An einen anderen Kanal als den der Sitzung zustellen
openclaw agent --agent ops --message "Warnung" --deliver --reply-channel telegram --reply-to "@admin"
```

## Verwandte Themen

<CardGroup cols={2}>
  <Card title="Agent-CLI-Referenz" href="/de/cli/agent" icon="terminal">
    Vollständige Referenz zu Flags und Optionen von `openclaw agent`.
  </Card>
  <Card title="Unteragenten" href="/de/tools/subagents" icon="users">
    Starten von Unteragenten im Hintergrund.
  </Card>
  <Card title="Sitzungen" href="/de/concepts/session" icon="comments">
    Funktionsweise von Sitzungsschlüsseln und wie `--to`, `--agent` und `--session-id` sie auflösen.
  </Card>
  <Card title="Slash-Befehle" href="/de/tools/slash-commands" icon="slash">
    Nativer Befehlskatalog zur Verwendung innerhalb von Agentensitzungen.
  </Card>
</CardGroup>
